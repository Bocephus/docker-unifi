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
  `/volume1/docker/unifi/db-data`, `/volume1/docker/unifi/db-init`,
  `/volume1/docker/unifi/network-application-config`.
- `MONGO_INITDB_ROOT_PASSWORD` and `MONGO_PASS` in `stack.env` are
  intentionally blank — real values go directly into Portainer's stack
  "Environment variables" field at deploy time, not into this repo.
- These Mongo credentials only take effect on a genuinely empty `/data/db`.
  If you're migrating existing Mongo data onto the new NFS path rather than
  starting fresh, the existing data's credentials apply instead, regardless
  of what's set here.

## Deploying in Portainer

1. **Stacks → Add stack → Repository**, pointed at this repo, branch `main`,
   compose path `docker-compose.yml`.
2. Load `stack.env`'s contents into the stack's environment variables
   (paste, or upload if your Portainer version supports it), then add
   `MONGO_INITDB_ROOT_PASSWORD` and `MONGO_PASS` manually with their real
   values — these two are deliberately not in the committed file.
3. Deploy. Since this is a Swarm stack, Portainer schedules the two services
   onto whichever swarm node has capacity; no node pinning is configured.
