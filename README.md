# BookOrbit Setup & Migration Guide

A runbook for my self-hosted [BookOrbit](https://github.com/bobberb/bookorbit) library: how it is built on a Raspberry Pi 4, and how to move it to another device with the full database.

> **Before you publish this repo:** it contains no secrets, only placeholders. Never commit `.env`, `bookorbit-backup.sql` or anything under `data/`. Add them to `.gitignore`:
>
> ```
> .env
> *.sql
> data/
> ```

## Contents

1. [Overview](#overview)
2. [Back up everything on the old Pi](#back-up-everything-on-the-old-pi)
3. [Compose file and .env](#compose-file-and-env)
4. [Set up the new device](#set-up-the-new-device)
5. [Restore the database and start BookOrbit](#restore-the-database-and-start-bookorbit)
6. [Syncthing and Tailscale](#syncthing-and-tailscale)
7. [BookOrbit settings to check or recreate](#bookorbit-settings-to-check-or-recreate)
8. [Lessons learned and gotchas](#lessons-learned-and-gotchas)
9. [Verification checklist and rollback](#verification-checklist-and-rollback)

## Overview

BookOrbit runs on a Raspberry Pi 4 (4 GB) as two Docker containers, and every file that matters lives on an external SSD mounted at `/mnt/games`.

- **BookOrbit app** (`bookorbit-app`, port 3000): the web app, KOReader sync endpoint and metadata fetcher.
- **PostgreSQL with pgvector** (`bookorbit-db`): holds libraries, metadata, reading progress and provider settings.
- **Syncthing** (systemd service, not Docker): copies EPUBs from the phone into the Books library folder.
- **Tailscale**: remote access to the Pi away from home.
- **KOReader plugin** on the Kobo and the Android phone: syncs reading progress with BookOrbit.

Where things live on the old Pi:

```
~/bookorbit/
  docker-compose.yml        # the stack definition
  .env                      # secrets and paths (never share)
/mnt/games/bookorbit/
  books/                    # EPUB library (Syncthing writes here)
  manga/                    # CBZ library, one folder per series
  data/app/                 # covers, thumbnails, app state
  data/postgres/            # live database files (do not copy while running)
~/bookorbit-backup.sql      # database dump (pg_dump)
```

BookOrbit never edits the book files: "Write metadata to files" and "Rename files" are both off, so all metadata edits exist only in the database.

## Back up everything on the old Pi

Take these backups first, and leave the old Pi untouched until the new device passes the checklist at the end.

| Item | Location | Needed |
| --- | --- | --- |
| Database dump | `~/bookorbit-backup.sql` | Yes |
| `docker-compose.yml` and `.env` | `~/bookorbit/` | Yes, the secrets must match |
| EPUB library | `/mnt/games/bookorbit/books` | Yes |
| Manga library | `/mnt/games/bookorbit/manga` | Yes (about 97 GB) |
| Covers and app data | `/mnt/games/bookorbit/data/app` | Recommended, covers can be rebuilt with "Refresh covers" |
| Syncthing config | folder printed by `syncthing paths` | Recommended, keeps the same device ID |

Run on the old Pi:

```bash
# 1. Database dump (safe while BookOrbit is running)
docker exec bookorbit-db pg_dump -U bookorbit bookorbit > ~/bookorbit-backup.sql
ls -lh ~/bookorbit-backup.sql        # should be a few MB, not 0

# 2. Compose file and secrets
mkdir -p ~/bookorbit-config-backup
cp ~/bookorbit/docker-compose.yml ~/bookorbit/.env ~/bookorbit-config-backup/

# 3. Show where Syncthing keeps its config
syncthing paths
```

Do not copy `/mnt/games/bookorbit/data/postgres` while the database is running, because the files can be half-written. It also fails if the new device uses a different CPU type or PostgreSQL version. The dump works in both cases.

Keep `~/bookorbit-config-backup/.env` somewhere private, such as a password manager. It holds the database password and the app secrets, which you should reuse unchanged on the new device.

## Compose file and .env

The stack is the stock BookOrbit compose file with three edits: the app and database data folders point at the SSD, and a `/manga` volume was added. This copy was rebuilt from the stock file plus those edits, so on the old Pi run `cat ~/bookorbit/docker-compose.yml` and prefer that output if it differs.

```yaml
name: bookorbit

services:
  app:
    container_name: bookorbit-app
    image: ${APP_IMAGE?required}
    restart: unless-stopped
    init: true
    env_file:
      - .env
    ports:
      - "${APP_PORT:-3000}:${PORT:-3000}"
    environment:
      NODE_ENV: production
      PORT: ${PORT:-3000}
      DATABASE_URL: ${DATABASE_URL:-}
      POSTGRES_HOST: ${POSTGRES_HOST:-postgres}
      POSTGRES_PORT: ${POSTGRES_PORT:-5432}
      POSTGRES_USER: ${POSTGRES_USER?required}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD?required}
      POSTGRES_DB: ${POSTGRES_DB?required}
      JWT_SECRET: ${JWT_SECRET?required}
      PODCAST_ENCRYPTION_KEY: ${PODCAST_ENCRYPTION_KEY:-}
      SETUP_BOOTSTRAP_TOKEN: ${SETUP_BOOTSTRAP_TOKEN?required}
      APP_URL: ${APP_URL?required}
      CLIENT_URL: ${CLIENT_URL:-${APP_URL?required}}
      TZ: ${TZ:-UTC}
      PUID: ${PUID:-1000}
      PGID: ${PGID:-1000}
      NODE_MAX_OLD_SPACE_SIZE: ${NODE_MAX_OLD_SPACE_SIZE:-auto}
      BOOK_DOCK_PATH: ${BOOK_DOCK_PATH:-}
      LIBRARY_BROWSE_ROOT: ${LIBRARY_BROWSE_ROOT:-}
      DISABLE_LOCAL_AUTH: ${DISABLE_LOCAL_AUTH:-false}
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ${BOOKS_HOST_PATH:-./books}:/books
      - /mnt/games/bookorbit/manga:/manga
      # - /mnt/games/bookorbit/lightnovels:/lightnovels   # optional, planned
      - /mnt/games/bookorbit/data/app:/data
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - SETGID
      - SETUID
    security_opt:
      - no-new-privileges:true
    stop_grace_period: 30s

  kokoro:
    container_name: bookorbit-kokoro
    image: ${KOKORO_IMAGE:-ghcr.io/remsky/kokoro-fastapi-cpu:v0.8.0}
    profiles: ["tts"]
    restart: unless-stopped
    init: true
    expose:
      - "8880"
    security_opt:
      - no-new-privileges:true
    stop_grace_period: 30s
    healthcheck:
      test:
        [
          "CMD-SHELL",
          'python -c "import urllib.request;urllib.request.urlopen(''http://127.0.0.1:8880/v1/audio/voices'')"',
        ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  postgres:
    container_name: bookorbit-db
    image: pgvector/pgvector:pg18
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER?required}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD?required}
      POSTGRES_DB: ${POSTGRES_DB?required}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - /mnt/games/bookorbit/data/postgres:/var/lib/postgresql/data
    healthcheck:
      test:
        ["CMD-SHELL", 'pg_isready -U "$${POSTGRES_USER}" -d "$${POSTGRES_DB}"']
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 20s
```

The `kokoro` service is the optional text-to-speech container and only starts with `--profile tts`, so you can delete it.

The matching `.env` (save a copy of this as `.env.example`, and keep the real values out of the repo). Copy the four secrets and the database password unchanged from the old file:

```
APP_IMAGE=ghcr.io/bookorbit/bookorbit:latest
APP_PORT=3000
BOOKS_HOST_PATH=/mnt/games/bookorbit/books
PUID=1000
PGID=1000
POSTGRES_USER=bookorbit
POSTGRES_PASSWORD=<copy from old .env>
POSTGRES_DB=bookorbit
JWT_SECRET=<copy from old .env>
PODCAST_ENCRYPTION_KEY=<copy from old .env>
SETUP_BOOTSTRAP_TOKEN=<copy from old .env>
APP_URL=http://<new-device-address>:3000
NODE_MAX_OLD_SPACE_SIZE=1024
TZ=Asia/Kolkata
```

On a device with more RAM, `NODE_MAX_OLD_SPACE_SIZE` can go back to 2048. `PUID` and `PGID` must match the owner of the SSD folders: check with `id` on the new device.

## Set up the new device

Install Docker, mount the SSD at the same path as before (`/mnt/games`), and the compose file works without any path changes.

1. **Install a 64-bit Linux** (Raspberry Pi OS, Debian or Ubuntu). Create the user `gaurav` so its UID is 1000, matching `PUID` and `PGID` in `.env`. Check with `id`.
2. **Install Docker** and log out and in again:

    ```bash
    curl -fsSL https://get.docker.com | sh
    sudo usermod -aG docker $USER
    ```

3. **Bring the SSD over.** Plug it into the new device and find its UUID and filesystem type:

    ```bash
    lsblk -f
    sudo mkdir -p /mnt/games
    sudo nano /etc/fstab
    ```

    Add one line, then mount it and check the folders are there:

    ```
    UUID=<uuid-from-lsblk> /mnt/games <fstype> defaults,nofail 0 2
    ```

    ```bash
    sudo mount -a
    ls /mnt/games/bookorbit
    ```

4. **If the SSD stays behind**, copy the libraries over the network instead (about 97 GB for manga, so use `tmux` and make sure the new disk has the space):

    ```bash
    mkdir -p /mnt/games/bookorbit/{books,manga,data/app,data/postgres}
    rsync -a --info=progress2 gaurav@<old-ip>:/mnt/games/bookorbit/books/ /mnt/games/bookorbit/books/
    rsync -a --info=progress2 gaurav@<old-ip>:/mnt/games/bookorbit/manga/ /mnt/games/bookorbit/manga/
    rsync -a gaurav@<old-ip>:/mnt/games/bookorbit/data/app/ /mnt/games/bookorbit/data/app/
    ```

5. **Reset the database folder.** If the SSD came with its old `data/postgres`, move it aside so PostgreSQL starts empty and the dump can be loaded into it:

    ```bash
    mv /mnt/games/bookorbit/data/postgres /mnt/games/bookorbit/data/postgres-old
    mkdir -p /mnt/games/bookorbit/data/postgres
    ```

6. **Copy the stack files and the dump** from the old Pi, then set the new address in `.env` (`APP_URL`):

    ```bash
    mkdir -p ~/bookorbit
    scp gaurav@<old-ip>:~/bookorbit-config-backup/docker-compose.yml ~/bookorbit/
    scp gaurav@<old-ip>:~/bookorbit-config-backup/.env ~/bookorbit/
    scp gaurav@<old-ip>:~/bookorbit-backup.sql ~/
    nano ~/bookorbit/.env
    ```

## Restore the database and start BookOrbit

Start only PostgreSQL, load the dump into its empty database, then start the app. This is standard PostgreSQL usage but has not yet been run end to end on this setup, so keep the old Pi running until the checklist passes.

1. **Start the database alone** and wait for it to show `healthy`:

    ```bash
    cd ~/bookorbit
    docker compose up -d postgres
    docker compose ps
    ```

2. **Load the dump** into the empty `bookorbit` database:

    ```bash
    docker exec -i bookorbit-db psql -U bookorbit -d bookorbit < ~/bookorbit-backup.sql
    ```

    Errors saying objects "already exist" mean the database was not empty. Fix it with `docker compose down`, empty `/mnt/games/bookorbit/data/postgres`, and repeat steps 1 and 2.

3. **Start everything** and watch the app log for errors:

    ```bash
    docker compose up -d
    docker compose logs -f app
    ```

4. **Open** `http://<new-address>:3000` and sign in with your existing account. If you see the first-time setup screen instead, the restore did not take.
5. **Rebuild covers if they are missing**: Settings > Libraries, the `...` menu on a library, then **Refresh covers**.
6. **Scan** each library once (Settings > Libraries > **Scan**) so BookOrbit matches the files on the new disk.

## Syncthing and Tailscale

Copying the Syncthing config keeps the same device ID, so the phone reconnects without re-pairing. Install both services after BookOrbit works.

**Syncthing (systemd service)**

1. On the new device: `sudo apt install syncthing`, then `sudo systemctl enable --now syncthing@gaurav`, then stop it again with `sudo systemctl stop syncthing@gaurav`.
2. On the old Pi, stop Syncthing (`sudo systemctl stop syncthing@gaurav`) and note the folder printed by `syncthing paths`.
3. Copy that config folder to the same location on the new device, then start Syncthing there:

    ```bash
    rsync -a gaurav@<old-ip>:<config-folder>/ <config-folder>/
    sudo systemctl start syncthing@gaurav
    ```

4. In the Syncthing web UI (port 8384), check that the books folder points at `/mnt/games/bookorbit/books` and the phone shows as connected.

If you start fresh instead, add the phone as a remote device, then add the folder with the **same Folder ID** and the path above. A folder's path cannot be edited after it is created, so if it is wrong, remove the folder and add it again.

**Tailscale (remote access)**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

The new device gets a new Tailscale address. Update it everywhere BookOrbit's address is saved: `APP_URL` in `.env` (then `docker compose up -d`), the BookOrbit plugin on the Kobo and the phone, and any OPDS catalog entries in reader apps.

## BookOrbit settings to check or recreate

The database restore brings all of these back, so only verify them. If you start from an empty database instead, set them by hand as listed.

| Area | Where | Value |
| --- | --- | --- |
| Books library | Settings > Libraries | Folder `/books`, File as Book, watching every hour |
| Manga library | Settings > Libraries | Folder `/manga`, File as Book, manual scans only |
| File updates | Library edit > File updates | "Rename files" and "Write metadata to files" both off |
| Providers on | Settings > Metadata > Providers | Hardcover (API key), Goodreads, Open Library, iTunes, ComicVine (API key) |
| Providers off | Same page | Amazon (needs a session cookie), Google Books, Kobo, RanobeDB (until light novels), audiobook sources |
| Global field rules | Metadata > Field Rules > Global defaults | Hardcover first. Fill missing on Title, Subtitle, Authors, Cover, Series name, Series index. Description overwrites. Genres merge |
| Manga overrides | Field Rules > Manga tab | ComicVine first, then Hardcover. Fill missing on everything except Genres (merge) |
| Genre limits | Bottom of Field Rules | Combine genres from all providers on, maximum 5, store provider IDs on |
| Auto-fetch | Metadata > Books | Auto-fetch and Trigger on import on, fetch if never fetched, score under 60, or Description or Cover missing. Manga inherits from global |

API keys for Hardcover and ComicVine come from your accounts on those sites. If the Providers page shows them as not connected after the restore, paste them in again.

**KOReader on the Kobo and phone**

- The BookOrbit plugin comes from Settings > Integrations > KOReader > Download Plugin in BookOrbit. Copy it into KOReader's `plugins` folder and restart.
- After a migration, reconnect it to the new server address from Tools > BookOrbit Sync.
- The sync gesture lives at Settings (gear) > Taps and gestures > Gesture manager > Hold corner, in the action list as "BookOrbit: sync all books".
- On the Android phone, manga reads with the Panels+ plugin set to Manga mode, Loose crop and tap-to-navigate.

## Lessons learned and gotchas

Most problems so far came from metadata rules that overwrite good data, and from deleting a library without noting its settings.

- **State at the time of writing:** during the manga rebuild test, most series were parked in `/mnt/games/manga-hold`. Before migrating, run `ls /mnt/games/manga-hold`, move anything left back with `mv /mnt/games/manga-hold/* /mnt/games/bookorbit/manga/`, then scan the Manga library.
- **Deleting a library wipes its settings.** Per-library field rules and fetch overrides disappear with it, so redo the Manga tab in Field Rules after re-adding a library.
- **Deleting a library keeps the files.** It removes books, metadata, progress, bookmarks and annotations from the database only. Checked on a 20-volume test series: all 20 files were still on disk.
- **"Overwrite if provided" on series and covers damaged the manga library** (wrong series, wrong covers). Use Fill missing. "Refresh covers" on the library restored the covers, and series were fixed by hand.
- **Setting "Score below" to 100 makes every book eligible.** Use it only for a deliberate bulk fetch, then set it back to 60.
- **Wrong volume numbers break the order.** A volume with series index 1 sorts next to volume 1. Fix it in Edit Metadata and lock the field with the padlock.
- **Duplicate authors** (typos, translators) are merged from the Authors page: open the correct author and merge the wrong ones into it.
- **Syncthing folder paths cannot be edited.** Remove the folder and add it again with the same Folder ID.
- **Never copy PostgreSQL's data folder while it runs.** Use `pg_dump`, and keep a dated copy before any bulk change.
- **Leftovers on the old Pi** that are safe to delete once the new setup is verified: `/mnt/games/bookdl` (old EPUB backup), `/mnt/games/calibre-library`, `/mnt/games/manga-backup-test`, and `/mnt/games/bookorbit/data/postgres-backup`.

## Verification checklist and rollback

The migration is done when every box below is ticked on the new device, and not before.

- [ ] `docker compose ps` shows `bookorbit-app` and `bookorbit-db` running, and the database healthy
- [ ] You can sign in with your existing account, not the first-time setup screen
- [ ] Book counts in Settings > Libraries match the old Pi for Books and Manga
- [ ] Berserk, Attack on Titan and Chainsaw Man show all volumes in order, with the right covers
- [ ] Settings > Metadata > Providers shows Hardcover and ComicVine as connected
- [ ] A test EPUB sent from the phone through Syncthing appears in BookOrbit with a cover and description
- [ ] "BookOrbit: sync all books" works from KOReader on the Kobo
- [ ] BookOrbit opens through Tailscale from outside your home network
- [ ] A Scan of both libraries adds or removes nothing unexpected

**Rollback.** The old Pi is never modified by this process, so if a check fails, start the stack there again with `docker compose up -d` and point the Kobo, the phone and Tailscale back at it. Only one Syncthing instance with the same device ID should run at a time, so stop it on one device before starting it on the other. Keep the old Pi and the three backups (`bookorbit-backup.sql`, the `.env`, the compose file) for at least a week after the switch.