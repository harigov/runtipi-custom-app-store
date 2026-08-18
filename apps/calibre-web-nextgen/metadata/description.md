# Calibre-Web NextGen

A self-hosted ebook library: a modern web UI over a standard Calibre library, plus the automation that Calibre desktop normally makes you do by hand. Drop a file into a watch folder and it gets converted, tagged, cover-fetched, and filed into the library without you touching anything.

[Calibre-Web NextGen](https://github.com/new-usemame/Calibre-Web-NextGen) is the community continuation of [Calibre-Web-Automated](https://github.com/crocodilestick/Calibre-Web-Automated), which is itself a fork of the original Calibre-Web. Same data format, same UI, same features — NextGen picked up the maintenance after upstream CWA went quiet (its last release was `v4.0.6` on 4 February 2026), merging the stalled PR queue and shipping fixes every few days.

## Why NextGen rather than crocodilestick/calibre-web-automated

This store exists to track upstream, and upstream CWA has not shipped a release in six months. NextGen is a drop-in replacement — the on-disk format is identical, so an existing CWA library, `app.db`, user accounts, OAuth tokens, and KOReader sync state all carry over by pointing the directory fields below at the old paths. If upstream CWA resumes releasing, switching back is a one-line image change in `docker-compose.json`.

## Features

- Web UI for browsing, searching, and reading (EPUB, PDF, CBZ/CBR, TXT)
- **Auto-ingest** — anything dropped in the ingest folder is imported, converted to the target format, and deleted from the ingest folder
- **Format conversion** via a bundled Calibre binary, plus `kepubify` for proper Kobo EPUBs
- **Kobo sync** and **KOReader sync** (reading position and progress across devices)
- Send-to-Kindle / send-to-device by email
- Metadata enforcement and bulk editing, with optional [Hardcover](https://hardcover.app) and [ComicVine](https://comicvine.gamespot.com) lookups
- Multi-user accounts with per-user shelves and permissions

## Directory configuration

Three separate host directories, each configurable at install time. All three are optional — leave a field blank and it lands under this app's own data directory instead.

| Field | Container path | Holds |
|---|---|---|
| **Calibre library directory** | `/calibre-library` | `metadata.db` and the book files. This is the library. |
| **Settings directory** | `/config` | `app.db` and `cwa.db` — users, OAuth tokens, KOReader sync state, logs |
| **Ingest directory** | `/cwa-book-ingest` | Import staging only |

Defaults when left blank:

- `${APP_DATA_DIR}/data/calibre-library`
- `${APP_DATA_DIR}/data/config`
- `${APP_DATA_DIR}/data/ingest`

Three things to get right:

1. **Use absolute host paths that already exist.** Docker creates a missing bind-mount source as an empty root-owned directory rather than failing, which then shows up as an empty library and permission errors on ingest.
2. **Don't nest them.** The ingest folder in particular must not sit inside the library — files there are deleted after a successful import.
3. **Match the UID/GID.** The container writes as `PUID:PGID` (default `1000:1000`). Check the real owner with `stat -c '%u %g' /path/to/library` and set the two fields to match, or fix the existing tree with `docker exec calibre-web-nextgen chown -R abc:abc /calibre-library`.

If the library lives on an NFS or SMB/CIFS mount, turn on **Library is on a network share**. It disables SQLite WAL mode and inotify-based watching, both of which misbehave on network filesystems — WAL can corrupt the database and inotify silently misses new files. Leave it off for local disks; it polls instead, which is slower.

## First login

Set **Admin password** at install time and the app never sits on the well-known default. The username stays `admin`. Leave the field blank and the upstream default `admin123` applies instead — in that case change it immediately under Profile → Account.

The password must satisfy the app's own policy: at least 8 characters with an uppercase letter, a lowercase letter, a digit, and a special character. A weaker value is rejected by the app, and the container still starts normally and logs

```
[runtipi] WARNING: admin password not applied - it must be at least 8 characters ...
```

leaving the previous password in place. It never blocks startup.

The password is applied only when the field's value *changes*, tracked by a hash in `/config/.runtipi-admin-password.sig`. So a restart will not overwrite a password you later changed in the web UI, while editing the field in Runtipi does rotate it. To force a re-apply, delete that file.

To use the metadata-fetch and cover-download features, also enable "Allow Uploads" under Admin → Feature Configuration.

## Timezone

Leave the **Timezone** field blank and the container inherits the Runtipi server's timezone (Settings → General → Timezone), which Runtipi exposes to every app as `TZ`. Set it only if this app should run on a different clock than the server. Without it the image reads the clock as UTC — its `/etc/localtime` is symlinked to `Etc/UTC` — so "date added" timestamps and scheduled tasks drift by your UTC offset.

## Reverse proxy

**Trusted proxy count** defaults to `1`, which is correct for Runtipi: Traefik is the single proxy in front. Getting this wrong shows up as a login loop, where a successful login bounces straight back to the login page because the app sees the client IP change between requests. Raise it only if you front Runtipi with a second proxy such as Cloudflare.

## Adopting an existing library

1. Install the app **without** starting to add books.
2. Set **Calibre library directory** to the existing library root — the folder containing `metadata.db`.
3. If migrating from a CWA or Calibre-Web install, set **Settings directory** to that install's `/config` folder to keep users, shelves, and sync state.
4. Leave **Trusted proxy count** at `1` — it must be a plain integer, since the app calls `int()` on it with no error handling and fails to start on anything else.
5. Confirm ownership matches the UID/GID fields, then start the app.

Note that when **Library is on a network share** is enabled, the image deliberately skips its startup `chown` of `/config`, `/calibre-library`, and `/cwa-book-ingest` — on a network share that operation is slow and often fails outright. Ownership on those mounts is then entirely yours to get right.

## Upgrades

The image is pinned to an exact tag (`v4.1.38`) rather than `latest`, so upgrades happen when this store bumps the tag — not silently on the next container restart. NextGen releases frequently; check the [releases page](https://github.com/new-usemame/Calibre-Web-NextGen/releases) before bumping.
