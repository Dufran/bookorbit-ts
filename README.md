# BookOrbit on Tailscale

A private Docker Compose stack for [BookOrbit](https://bookorbit.app) with PostgreSQL and a Tailscale sidecar.

No service publishes a port on the Docker host. BookOrbit shares the Tailscale container's network namespace, and Tailscale Serve terminates HTTPS and proxies tailnet traffic to BookOrbit on loopback. Tailscale Funnel is explicitly disabled.

```text
Tailnet client
    |
    +-- HTTPS :443 --> Tailscale sidecar --> BookOrbit :3000
                                            |
                                            +--> PostgreSQL + pgvector
```

## Requirements

- Docker Engine with Docker Compose v2
- [Task](https://taskfile.dev/) (recommended; direct Compose commands also work)
- a Tailscale tailnet with MagicDNS and HTTPS certificates enabled
- a reusable, pre-authorized Tailscale auth key, preferably tagged and ACL-restricted
- a host directory containing the book library

## Setup

1. Create the local configuration:

   ```bash
   task init
   # or: cp .env.example .env
   ```

2. Generate secrets and replace every `CHANGE_ME` in `.env`:

   ```bash
   openssl rand -hex 24  # POSTGRES_PASSWORD
   openssl rand -hex 32  # JWT_SECRET
   openssl rand -hex 16  # SETUP_BOOTSTRAP_TOKEN
   ```

3. Configure Tailscale and the public application URL:

   ```dotenv
   TS_HOSTNAME=bookorbit
   TS_AUTHKEY=tskey-auth-...
   APP_URL=https://bookorbit.example-tailnet.ts.net
   ```

   Replace `example-tailnet.ts.net` with the DNS name for your tailnet. `APP_URL` must exactly match the URL clients use.

4. Keep the default `PUID` and `PGID` unless the deployment requires a different container identity. Books are stored in the Docker-managed `books_data` volume.

5. Validate and start:

   ```bash
   task validate
   task pull
   task up
   task logs SERVICE=app
   ```

6. Open `APP_URL` from a device connected to the tailnet and complete setup using `SETUP_BOOTSTRAP_TOKEN`.

## Operations

```bash
task ps
task logs
task logs SERVICE=tailscale
task restart
task pull
task up
task down       # preserves all named volumes
```

Direct Compose commands work as well:

```bash
docker compose up -d
docker compose logs -f app
docker compose down
```

Do not run `docker compose down --volumes` unless you intend to delete the book library, BookOrbit application data, PostgreSQL data, and Tailscale state.

## Storage

| Data | Location |
| --- | --- |
| Book library | `books_data` named volume mounted at `/books` |
| BookOrbit application data | `app_data` named volume |
| PostgreSQL database | `postgres_data` named volume |
| Tailscale identity/state | `tailscale_state` named volume |

Named volumes and `.env` contain sensitive data and are not encrypted by this stack. Back them up separately. Database backups should use PostgreSQL-native tools rather than copying a live volume.

## Security notes

- The Compose file has no `ports` entries; neither BookOrbit nor PostgreSQL is bound to a host port.
- Tailscale Serve accepts tailnet HTTPS traffic only, and Funnel is disabled.
- Access should also be restricted with Tailscale ACLs/grants, especially when using a tagged auth key.
- The BookOrbit container keeps the upstream read-only filesystem and capability hardening.
- An app `post_start` hook sets the `books_data` volume root to `PUID:PGID`; no extra helper container is required.
- Pin image versions or digests before using this as a production deployment.

## Troubleshooting

### Tailscale node does not appear

```bash
docker compose logs tailscale
docker compose exec tailscale tailscale status
```

Confirm the auth key is valid, pre-authorized, and allowed to use the requested tag. Existing `tailscale_state` retains registration across restarts.

### BookOrbit is unavailable

```bash
docker compose ps
docker compose logs tailscale app postgres
```

Confirm `APP_URL` uses the MagicDNS name created for `TS_HOSTNAME`, and verify HTTPS is enabled in the Tailscale admin console.

### Library permission errors

Recreate the app so its post-start hook reapplies ownership to the named volume, then verify it is writable:

```bash
docker compose up -d --force-recreate app
docker compose exec app sh -lc 'touch /books/.write-test && rm /books/.write-test'
```

## References

- [BookOrbit installation](https://bookorbit.app/installation)
- [BookOrbit repository](https://github.com/bookorbit/bookorbit)
- [Tailscale Serve configuration](https://tailscale.com/kb/1242/tailscale-serve)
