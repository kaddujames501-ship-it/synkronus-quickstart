# Migrating Synkronus data to the new layout

Recent Synkronus releases store all mutable files under **one directory** next to the binary (`/app/synkronus` → **`/app/data`** in Docker). There is **no** `DATA_DIR` or `APP_BUNDLE_PATH` environment variable for the default layout.

## New directory layout (inside the volume mounted at `/app/data`)

| Path | Purpose |
|------|---------|
| `app-bundle/active/` | Extracted app bundle currently served by the API |
| `app-bundle/versions/` | Numbered versions (`0001/`, `0002/`, …), `CURRENT_VERSION`, per-version `bundle.zip` |
| `attachments/` | Attachment blobs for sync |

## What changed from older quickstarts

| Old (typical) | New |
|---------------|-----|
| `app-bundles/` (active files) | `app-bundle/active/` |
| `app-bundle-versions/` (sibling folder) | `app-bundle/versions/` |
| (often missing or wrong path) | `attachments/` next to the above |

If you only ever used defaults, your named volume may still contain **`app-bundles/`** and **`app-bundle-versions/`** at the top level of `/app/data`.

### Attachments may not have been on the same volume (important)

In **ode-synkronus-quickstart**, the compose file has long used a single named volume at **`appdata:/app/data`**. In practice, **app bundle** files under `/app/data/...` were what people cared about first, but **older Synkronus builds** could still store **attachment blobs outside that tree** (e.g. under the container’s writable filesystem, such as **`/app/attachments`**, depending on version and working directory). Those paths are **not** part of the `appdata` volume unless you added an extra mount.

So:

- **Rebuilding or replacing the container** (`compose up --force-recreate`, new image, new container name) can **wipe** attachment files that only lived in the container layer, even though **`appdata` is preserved**.
- Using **`docker run --rm`** for one-off commands only affects **that** throwaway container; it does not delete your **named volumes**. Confusion usually comes from what is on the volume vs what was only in the old container’s filesystem.

**Before** you migrate layout or recreate the Synkronus service, check whether attachments exist **only** in the old container:

```bash
# While the old synkronus container is still running (or stopped but not removed):
podman compose exec synkronus sh -c 'ls -la /app/data/attachments 2>/dev/null; ls -la /app/attachments 2>/dev/null; du -sh /app/data /app/attachments 2>/dev/null'
```

If you see files under **`/app/attachments`** (or similar) but **`/app/data/attachments`** is empty or missing, **copy them into the volume** before dropping the old container. The repo includes **`utilities/backup-attachments.sh`**, which copies both **`/app/data/attachments`** and **`/app/attachments`** (if non-empty) from a **running** container to a timestamped folder on the host:

```bash
chmod +x ./utilities/backup-attachments.sh
./utilities/backup-attachments.sh -c synkronus   # or: -c <id from `podman ps`
```

Then merge the salvaged files into your `appdata` volume under `attachments/` while the stack is stopped (see volume mountpoint from `podman volume inspect`). After upgrading to current Synkronus + this quickstart’s compose, **attachments live at `/app/data/attachments`** on **`appdata`**.

The **`utilities/migrate-synkronus-data.sh`** script only rearranges directories **inside the data volume** (legacy `app-bundles` / `app-bundle-versions`). It does **not** pull files from the container layer; use **`backup-attachments.sh`** first if attachments might only exist on the container filesystem.

## Before you start

1. **Stop the stack** so nothing writes during migration:
   ```bash
   podman compose down
   # or: docker compose down
   ```
2. **Back up the volume** (example for a named volume — adjust project/volume name from `podman volume ls` / `docker volume ls`):
   ```bash
   docker run --rm -v <your_appdata_volume>:/data -v "$(pwd)":/backup alpine \
     tar czf /backup/synkronus-appdata-backup.tgz -C /data .
   ```

## Option A — automated script (recommended)

From this repo, with the stack **stopped**:

```bash
chmod +x ./utilities/migrate-synkronus-data.sh
```

**Bind mount** (you know the host directory that backs `/app/data`):

```bash
sudo ./utilities/migrate-synkronus-data.sh /path/to/host/data
```

**Dry run** (prints actions only):

```bash
./utilities/migrate-synkronus-data.sh --dry-run /path/to/host/data
```

**Named volume** (one-shot container; replace `<volume_name>` from `podman volume ls` / `docker volume ls`, e.g. `ode-synkronus-quickstart_appdata`):

```bash
podman run --rm \
  -v <volume_name>:/data:Z \
  -v "$PWD/utilities/migrate-synkronus-data.sh:/migrate.sh:ro,Z" \
  docker.io/library/alpine:3.21 \
  sh /migrate.sh /data
```

Use `docker run` instead of `podman run` if you use Docker.

To inspect a volume’s mountpoint on the host (optional):

```bash
podman volume inspect <volume_name> --format '{{ .Mountpoint }}'
```

The script is **idempotent**: if `app-bundle/active` already looks populated, it skips overwriting.

## Option B — manual steps

Inside the data root (the volume root that becomes `/app/data` in the container):

```bash
mkdir -p app-bundle/active app-bundle/versions attachments

# Active bundle
if [ -d app-bundles ] && [ "$(ls -A app-bundles 2>/dev/null)" ]; then
  cp -a app-bundles/. app-bundle/active/
fi

# Version history
if [ -d app-bundle-versions ] && [ "$(ls -A app-bundle-versions 2>/dev/null)" ]; then
  cp -a app-bundle-versions/. app-bundle/versions/
fi
```

Then rename old folders only after you have verified the service starts:

```bash
mv app-bundles app-bundles.pre-migration-$(date +%Y%m%d) 2>/dev/null || true
mv app-bundle-versions app-bundle-versions.pre-migration-$(date +%Y%m%d) 2>/dev/null || true
```

## After migration

1. Update `docker-compose.yml` from this repo (current Synkronus does not read `DATA_DIR` or bundle path overrides; data must live under `/app/data` as documented).
2. Start the stack:
   ```bash
   podman compose up -d
   ```
3. Check health and logs:
   ```bash
   curl -sS http://localhost:8080/health
   podman compose logs -f synkronus
   ```

## Non-standard layouts

Current Synkronus always uses **`<exe-dir>/data/app-bundle/active`** and **`.../versions`**. If you used custom bind mounts or old env overrides, **move or copy** those files into that tree on the volume (or under your single `/app/data` mount) before starting the new image. There are no environment variables to point the server at alternate directories.

## Rollback

Restore from the tarball taken in “Back up the volume”, or rename `app-bundles.pre-migration-*` back to `app-bundles` and use an older `docker-compose.yml` that still pointed `APP_BUNDLE_PATH` at `/app/data/app-bundles` (not recommended long term).
