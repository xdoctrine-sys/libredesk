# Libredesk trial on Railway (tarshli.com project)

This folder documents the Railway trial deployment of Libredesk built **from this
fork**, not from the upstream Docker Hub image.

## Pinned source

| Item | Value |
|---|---|
| Fork (origin) | `https://github.com/xdoctrine-sys/libredesk` |
| Upstream | `https://github.com/abhinavxd/libredesk` |
| Trial branch | `trial/railway-tarshli-2026-11` |
| Pinned tag | `v2.9.2` |
| Pinned commit | `dcd62c9c3e649085cc5cd868d6d33cfc8e1063ee` |
| Licence | AGPL-3.0 (`LICENSE`, unchanged) |

`Dockerfile.railway` rebuilds the release pipeline from source (frontend build ->
Go binary -> `stuffbin` asset stuffing) because the repository's own `Dockerfile`
expects a pre-built binary produced by goreleaser.

## Railway resources

| Resource | Type | Purpose |
|---|---|---|
| `libredesk-trial-web` | Service, source = fork repo + branch | Application (single binary) |
| `libredesk-trial-postgres` | Postgres | Dedicated database (never shared with Chatwoot) |
| `libredesk-trial-redis` | Redis | Dedicated cache/queue |

A Railway volume is attached to `libredesk-trial-web` at `/libredesk/uploads`
for attachment storage (`upload.fs.upload_path`).

## Configuration

All configuration is injected through environment variables (the image keeps
`config.toml` from `config.sample.toml`, and environment variables override it).
See `env.example` for the variable **names** only - never commit real values.

Two settings are **not** environment driven:

* `app.root_url` - read from the database at startup, therefore set it in the
  application UI under *Settings -> General -> Root URL*.
* Anything else exposed in the settings screens.

## One-time database install

`--install` deletes existing data, so it is deliberately excluded from the
container command. The image command is:

```
sh -c "./libredesk --upgrade --yes && exec ./libredesk"
```

Install is performed **once**, against a verified empty database, by temporarily
overriding the service start command (`serviceInstanceUpdate` in the Railway
GraphQL API) with:

```
sh -c "./libredesk --install --yes"
```

Then read the deploy logs for `database schema installed successfully`, remove the
override and redeploy so the service returns to the normal command above.

## Redeploy / upgrade

1. Push the new commit to the trial branch.
2. Railway rebuilds and redeploys automatically.
3. The container command runs `--upgrade --yes` before serving, which applies
   pending migrations and is a no-op when the schema is current. It never
   re-installs and never wipes data.

## Stopping the trial later

Remove the three services (`libredesk-trial-web`, `libredesk-trial-postgres`,
`libredesk-trial-redis`), their volumes and the DNS record. No Chatwoot/Tarshli
resource has to change.
