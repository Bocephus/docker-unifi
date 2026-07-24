# docker-unifi

UniFi Network Application + its MongoDB backend, deployed as a Docker Swarm
stack via Portainer. Persistent storage is NFS-backed (NAS-hosted, same
pattern as `docker-nginx-proxy-manager`), and services float across the
swarm rather than being pinned to a specific node.

## Files

- `docker-compose.yml` — service definitions, parameterized via environment
  variables; no per-environment edits needed.
- `stack.env` — settings for this stack. Mongo's root/app passwords are left
  blank here — see below.

## Before deploying

- The NFS export paths referenced in `docker-compose.yml` must already exist
  on the NAS (`nas01.lab.sal9000.tech`):
  `/volume1/docker/unifi/db-data`, `/volume1/docker/unifi/network-application-config`.
- `MONGO_INITDB_ROOT_PASSWORD` and `MONGO_PASS` in `stack.env` are
  intentionally blank — real values go directly into Portainer's stack
  "Environment variables" field at deploy time, not into this repo.
- These Mongo credentials only take effect on a genuinely empty `/data/db`.
  If you're migrating existing Mongo data onto the new NFS path rather than
  starting fresh, the existing data's credentials apply instead, regardless
  of what's set here — and the `unifi` app user won't get (re-)created
  automatically either, since Mongo only runs its init scripts on a
  first-ever empty data directory (see below).

### How the `unifi` Mongo user gets created

Docker Swarm's `docker stack deploy` doesn't support the extended
`depends_on: {condition: ...}` syntax (only a plain list, with no
start-order or completion guarantees) — so this can't rely on a separate
one-shot service to prep `/docker-entrypoint-initdb.d` before Mongo starts,
the way a standalone `docker compose` stack could. Instead, `unifi-db`
overrides its own entrypoint to write the `init-mongo.sh` script (the same
one from
[linuxserver/docker-unifi-network-application](https://github.com/linuxserver/docker-unifi-network-application)'s
own recipe) directly into the container at startup, then `exec`s into
Mongo's real `docker-entrypoint.sh mongod`. This keeps everything
sequential within a single container, with no cross-service race — but it
still only runs on a genuinely fresh `/data/db`; it won't retroactively fix
already-initialized data missing the `unifi` user (create it manually via
`mongosh` in that case).

## Deploying in Portainer

1. **Stacks → Add stack → Repository**, pointed at this repo, branch `main`,
   compose path `docker-compose.yml`.
2. Load `stack.env`'s contents into the stack's environment variables
   (paste, or upload if your Portainer version supports it), then add
   `MONGO_INITDB_ROOT_PASSWORD` and `MONGO_PASS` manually with their real
   values — these two are deliberately not in the committed file.
3. Deploy. Since this is a Swarm stack, Portainer schedules the two services
   onto whichever swarm node has capacity; no node pinning is configured.
