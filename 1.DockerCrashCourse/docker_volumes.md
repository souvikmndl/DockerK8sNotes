# Docker Volumes & Storage — Detailed Notes

## 1. Why storage is a problem in Docker

A container's filesystem is built from **read-only image layers** stacked under a single **thin read-write layer** (the "container layer"), unified by a storage driver (usually `overlay2`) via a union filesystem.

Key consequence: **the read-write layer lives and dies with the container.**

- `docker stop` / `docker start` → data survives (same container, same RW layer).
- `docker rm` → the RW layer is deleted → **all data written there is gone.**
- The RW layer uses **copy-on-write (CoW)**: modifying a file from a lower image layer first copies it up into the RW layer. This is fine for small changes but slow and space-hungry for databases doing constant writes.

So for anything you want to *persist* (databases, uploads, logs) or *share* (config, source code), you must write it **outside** the container's writable layer. That is what mounts are for.

There are three mount types: **volumes**, **bind mounts**, and **tmpfs mounts**.

---

## 2. The three mount types at a glance

| Aspect | Volume | Bind mount | tmpfs |
|---|---|---|---|
| Managed by Docker | Yes | No | Yes (in memory) |
| Source location | `/var/lib/docker/volumes/<name>/_data` (Linux) | Any absolute host path you choose | Host RAM only |
| Survives container removal | Yes | Yes (it's just a host dir) | No |
| Portable across hosts/OS | Yes | No (host paths differ) | N/A |
| Good for | Production data, databases | Dev source code, config injection | Secrets, scratch data |
| Bypasses storage driver (fast writes) | Yes | Yes | Yes |

**Rule of thumb:** volumes for data you keep, bind mounts for files you edit on the host, tmpfs for sensitive/ephemeral data that must never touch disk.

An important shared property: writes to any of these **bypass the union/CoW filesystem** and go straight to the target (the volume dir, the host path, or RAM). That is why database performance on a volume is much better than on the container layer.

---

## 3. Volumes (Docker-managed)

A volume is a directory that Docker creates and manages under its own data root. On Linux the physical location is:

```
/var/lib/docker/volumes/<volume-name>/_data
```

The `_data` subdirectory is the actual mount point contents; the parent holds Docker's metadata.

> **Docker Desktop caveat (Mac/Windows):** there is no native Linux host. Docker runs inside a lightweight Linux VM, so `/var/lib/docker/volumes/...` lives *inside that VM*, not on your Mac/Windows filesystem. You can't just `cd` to it from Finder/Explorer. On native Linux you genuinely can `sudo ls` that path.

### Named vs anonymous volumes

- **Named volume** — you give it a name; easy to find, reuse, back up.
  ```bash
  docker volume create pgdata
  docker run -v pgdata:/var/lib/postgresql/data postgres
  ```
- **Anonymous volume** — no name given; Docker assigns a long random hash. Created when you mount a target with no source name, or when an image declares `VOLUME` (see the Postgres example). Persists after the container is removed but becomes **dangling** and hard to identify.
  ```bash
  docker run -v /var/lib/postgresql/data postgres   # anonymous
  ```

### Volume commands

```bash
docker volume create <name>       # create
docker volume ls                  # list
docker volume inspect <name>      # see Mountpoint, driver, labels
docker volume rm <name>           # remove one
docker volume prune               # remove all unused (dangling) volumes
```

`docker volume inspect <name>` is the quickest way to see exactly where the data sits:

```json
[
  {
    "Name": "pgdata",
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/pgdata/_data",
    "Scope": "local"
  }
]
```

### Volume drivers

The default driver is `local`. Plugin drivers let a volume live on network/cloud storage (NFS, cloud block storage, etc.), so the same volume can follow a container across hosts — useful in clustered/Swarm setups. You can also pass driver options (e.g. mount an NFS export as a "local" volume):

```bash
docker volume create --driver local \
  --opt type=nfs --opt o=addr=10.0.0.5,rw \
  --opt device=:/exported/path nfsdata
```

---

## 4. Bind mounts (host path → container path)

A bind mount maps a **specific path on the host** into the container. There is no Docker management or metadata — it's a direct pass-through to the host directory/file.

```bash
docker run -v /home/souvik/app:/usr/src/app node:20
```

- Changes on the host are instantly visible inside the container and vice versa. This is why bind mounts are the standard tool for **local development** (edit code on the host, run it in the container with hot reload).
- Also common for injecting a **single config file**:
  ```bash
  docker run -v /etc/myapp/config.yaml:/app/config.yaml:ro myapp
  ```
  The `:ro` makes it read-only inside the container.

### Caveats

- **Not portable** — the host path must exist and be identical wherever you run it; breaks CI/other machines.
- **Overshadowing** — if you bind-mount a host directory onto a container path that already had image content, the host directory *hides* whatever the image put there. If the host dir is empty, the container path appears empty. (Named volumes behave differently — see the next note.)
- **Permissions/SELinux** — UID/GID mismatches between host and container user cause "permission denied". On SELinux systems you often need the `:z` / `:Z` suffix to relabel.

### The one useful difference: empty volume pre-population

When you mount an **empty named volume** onto a **non-empty** directory from the image, Docker **copies the image's existing contents into the volume** on first use. A **bind mount does not do this** — it just overshadows. This behavior matters for images that ship seed data at the mount point.

---

## 5. tmpfs mounts (memory-backed)

A tmpfs mount stores data in the host's **RAM only** — nothing is written to disk, and everything vanishes when the container stops. Linux-only, and only for `docker run` (not for building images).

```bash
docker run --tmpfs /app/cache:size=64m,mode=1777 myapp
# or with --mount:
docker run --mount type=tmpfs,destination=/app/cache,tmpfs-size=64m myapp
```

Use for: secrets you don't want persisted, scratch/temp files, or reducing disk writes for throwaway data.

---

## 6. Syntax: `-v` / `--volume` vs `--mount`

Both attach storage; `--mount` is more explicit and is the recommended modern form (and the only way to use some options like Swarm/cluster settings).

**`-v` (short, positional, colon-separated):** `SOURCE:TARGET:OPTIONS`

```bash
docker run -v pgdata:/var/lib/postgresql/data:ro postgres
```
- If `SOURCE` looks like a name → named volume. If it's an absolute path → bind mount. If omitted → anonymous volume.
- Quirk: with `-v`, if a bind-mount host path **doesn't exist, Docker creates it as a directory**. This silently causes bugs.

**`--mount` (explicit `key=value` pairs):**

```bash
docker run \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  postgres

docker run \
  --mount type=bind,source=/home/souvik/app,target=/usr/src/app,readonly \
  node:20
```
- `type=` must be `volume`, `bind`, or `tmpfs`.
- With `type=bind`, if the host path **doesn't exist, it errors out** instead of silently creating it — safer.

Prefer `--mount` in scripts and Compose for clarity; `-v` is fine for quick ad-hoc runs.

### In `docker-compose.yml`

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data          # named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # bind mount

volumes:
  pgdata:        # declares the named volume
```

---

## 7. Detailed example: where PostgreSQL data actually lives

This is the part worth internalizing, because Postgres behaves subtly even if you *don't* ask for a volume.

### 7.1 The key fact: Postgres writes to `PGDATA`

Inside the official `postgres` image, the database cluster (all your tables, indexes, WAL, config) is stored at:

```
/var/lib/postgresql/data      # this is $PGDATA
```

Everything the DB persists goes under this single directory *inside the container*.

### 7.2 The image already declares a VOLUME

The official Postgres `Dockerfile` contains a line equivalent to:

```dockerfile
VOLUME /var/lib/postgresql/data
```

This means: **even if you never pass `-v`, Docker automatically creates an anonymous volume** for that path when the container starts. The image authors did this deliberately so a database write never lands in the slow, disposable container layer.

So there are three scenarios:

---

**Scenario A — you specify nothing (`docker run postgres`)**

```
Flow of a write:
Postgres process
  → write() to /var/lib/postgresql/data/base/...   (path inside container)
  → that path is a mount point, NOT the container RW layer
  → Docker routes it to an ANONYMOUS volume
  → physically lands at:
     /var/lib/docker/volumes/<random-64-hex>/_data/base/...   (on the Linux host)
```

- Data **survives `docker stop`/`start`** and even **`docker rm`** (the anonymous volume outlives the container).
- BUT the volume has a random hash name, so it's effectively orphaned/dangling — hard to find, easy to `docker volume prune` by accident. **Don't rely on this.**

---

**Scenario B — you use a named volume (recommended)**

```bash
docker run -d --name mypg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
```

```
Full data flow:

  ┌─────────────────────────────────────────────┐
  │  Container: mypg                             │
  │                                             │
  │  postgres backend process                   │
  │        │ write() row / index / WAL          │
  │        ▼                                     │
  │  /var/lib/postgresql/data   ◄── mount point │
  └────────────────────┬────────────────────────┘
                       │  (bypasses overlay2 / CoW layer entirely)
                       ▼
     Named volume "pgdata"  (driver: local)
                       │
                       ▼
     HOST filesystem (Linux):
     /var/lib/docker/volumes/pgdata/_data/
        ├── base/            ← actual table/index data files
        ├── global/
        ├── pg_wal/          ← write-ahead log
        ├── pg_xact/
        ├── postgresql.conf
        └── ...
```

Verify it yourself:

```bash
docker volume inspect pgdata
# -> "Mountpoint": "/var/lib/docker/volumes/pgdata/_data"

# On native Linux:
sudo ls /var/lib/docker/volumes/pgdata/_data
# base  global  pg_wal  pg_xact  postgresql.conf  ...
```

Now the lifecycle is clean:

- `docker rm -f mypg` → container gone, **`pgdata` volume untouched**.
- `docker run ... -v pgdata:/var/lib/postgresql/data postgres:16` again → new container **re-attaches the same data**, DB comes back exactly as it was.
- This is how you upgrade the Postgres image version without losing data (subject to major-version on-disk format compatibility).

**First-boot detail:** on a *fresh empty* `pgdata`, the Postgres entrypoint runs `initdb`, creates the cluster, applies `POSTGRES_PASSWORD`, and runs any scripts in `/docker-entrypoint-initdb.d/`. On subsequent starts it sees an existing cluster in the volume and **skips init** — so those init scripts and the password env only take effect on the very first initialization of that volume.

---

**Scenario C — you use a bind mount to a host directory**

```bash
docker run -d --name mypg \
  -e POSTGRES_PASSWORD=secret \
  -v /data/postgres:/var/lib/postgresql/data \
  postgres:16
```

- Data lands directly in `/data/postgres` on the host — easy to see and back up with normal host tools.
- Trade-offs: you own the permissions/ownership problems (the container's `postgres` user UID must be able to write there), and it's non-portable. Named volumes avoid most of this, which is why they're the default recommendation for databases.

### 7.3 What happens WITHOUT any of this (the anti-pattern)

If Postgres *didn't* declare a `VOLUME` and you ran it with no mount, every write would go to the container's CoW layer. Then:

- `docker rm` → **entire database lost.**
- Constant CoW copy-ups → slower writes and bloated container layer.

Postgres avoids this for you via the built-in `VOLUME`, but your *own* app images usually won't — so for any stateful service **you** must add the volume explicitly.

---

## 8. Practical checklist / gotchas

- **Always use a named volume (or deliberate bind mount) for databases.** Never trust the container layer or an anonymous volume for data you care about.
- **Back up volumes** by running a throwaway container that tars the volume out:
  ```bash
  docker run --rm -v pgdata:/data -v $(pwd):/backup alpine \
    tar czf /backup/pgdata.tar.gz -C /data .
  ```
  (For Postgres specifically, prefer a logical dump: `pg_dump` / `pg_dumpall`, which is version-safe.)
- **`docker volume prune` deletes dangling volumes** — an anonymous DB volume can vanish this way. Name your volumes.
- **`-v` auto-creates missing bind paths as dirs; `--mount` errors instead.** Use `--mount` to catch typos.
- **Mounting an empty named volume onto image content copies the content in; a bind mount does not.**
- **Docker Desktop stores volumes inside a VM**, so the `Mountpoint` path isn't directly browsable from macOS/Windows.
- **`ro`/`:ro` / `readonly`** makes a mount read-only inside the container — good for injected config.
- **Permissions:** container process UID must match the file ownership on bind mounts; mismatches are the #1 "it works with a volume but not a bind mount" cause.