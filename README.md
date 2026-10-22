# SMAgent — self-hosted deployment

SMAgent is a social-media agent: a Spring Boot backend with an H2 database and
a React SPA, packaged as a single container image with everything behind one
port and one base path (e.g. `https://example.com/myapp`).

## Requirements

- Docker with the Compose plugin (any recent version)
- A reverse proxy with TLS if you want `https://` (recommended; examples below)

## Quick start

```sh
# get these files (clone this repo, or download compose.yaml alone)
docker compose up -d
docker compose logs smagent | grep -B1 -A4 'password'
```

On first start the container generates and logs the credentials it created:
the initial owner password for the web UI, and the H2 database password
(kept in the data volume, you rarely need it). Open the URL you configured
and log in.

## Configuration

Everything is configured via environment variables in `compose.yaml`, or
optionally via mounted YAML files. `APP_BASE_URL` is the only required
variable.

| Variable | Required | Default | Meaning |
|---|---|---|---|
| `APP_BASE_URL` | **yes** | — | External URL incl. path, e.g. `https://example.com/myapp`. Drives OAuth callbacks, CORS, and (unless `BASE_PATH` is set) where the app is served. |
| `BASE_PATH` | no | path of `APP_BASE_URL` | Path the app is served under, leading + trailing slash (`/myapp/`, or `/` for a dedicated subdomain). |
| `SMAGENT_ADMIN_USERNAME` | no | `admin` | Initial owner username. |
| `SMAGENT_ADMIN_PASSWORD` | no | generated | Initial owner password (printed once to the logs, stored in the data volume). |
| `SMAGENT_ENCRYPTION_KEY` | no | generated key file | Base64 of exactly 32 bytes, encrypts stored platform credentials. If you set it, back it up — losing it loses all stored platform logins. |
| `SPRING_DATASOURCE_USERNAME` / `_PASSWORD` | no | generated | H2 database credentials. Only relevant when re-using a database created outside this container. |
| `LINKEDIN_CLIENT_ID` / `_SECRET`, `X_CLIENT_ID` / `_SECRET` | **yes**\* | — | OAuth app credentials for the LinkedIn/X integrations. \*Current release refuses to start without them (even unused); placeholders are fine until that's fixed. |
| `JAVA_OPTS` | no | JVM defaults | e.g. `-Xmx512m`. |

### File-based configuration

The `config/` directory ships with two fully commented-out templates:
`deployment.yaml` (misc settings) and `secrets.yaml` (credentials). The
compose file mounts it into the container read-only; the backend imports both
at startup. Uncomment only what you need — environment variables from the
table above always win over these files, and the files are read once at
startup.

```yaml
# config/deployment.yaml — example
smagent:
  admin:
    username: boss
```

> **Do not set the base URL in `deployment.yaml`** (`smagent.base-url`).
> The app's URL belongs in the `APP_BASE_URL` environment variable in
> `compose.yaml` only — the container derives the SPA routing, CORS origin
> and cookie settings from that variable, and setting the URL in the file
> instead desynchronizes them from the backend's OAuth redirects.

### What lives where

Everything persistent is in the `smagent-data` volume: H2 database files,
the encryption key file (if generated), and `.generated-secrets` (the
credentials printed at first start, mode 600). Back that volume up and you
can recreate the container freely.

## Reverse proxy (TLS)

**nginx** — see `nginx-proxy-example.conf`. The key rule: forward the path
**without stripping it**:

```nginx
location /myapp/ {
    proxy_pass http://127.0.0.1:8080;           # NO URI part after the port
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

**Caddy** — equivalent:

```
example.com {
    reverse_proxy /myapp/* 127.0.0.1:8080
}
```

The app detects `https` from `X-Forwarded-Proto` and marks session cookies
`Secure` accordingly. If you skip the proxy for a local test, plain
`http://localhost:8080/myapp/` works as-is (redirects from `/` into the base
path automatically).

## Upgrading

```sh
# edit the image tag in compose.yaml, then:
docker compose pull
docker compose up -d
```

The database and secrets live in the volume — they survive image upgrades.
In-flight sessions are lost on restart (users simply log in again).
Pin an exact version tag in production; `latest` is convenient but you
upgrade whenever you re-pull.

## Backup / restore

```sh
docker compose stop
docker run --rm -v smagent_smagent-data:/data -v "$PWD":/backup \
  alpine tar czf /backup/smagent-data.tgz -C /data .
# restore: same command with xzf into an empty volume, before first start
docker compose start
```

(`smagent_smagent-data` is the volume name for this compose project; check
with `docker volume ls` if unsure. The encryption key must be in the backup —
without it stored platform credentials are unrecoverable.)

## Troubleshooting

- **Container exits immediately, logs mention `LINKEDIN_CLIENT_ID`** —
  current release requires all four OAuth variables (see table above).
- **Forgot the generated owner password** — it's in the volume:
  `docker compose exec smagent cat /opt/smagent/data/.generated-secrets`.
  (It only seeds the account at first start; to change it later, use the
  app's own UI or set a new database-side value.)
- **OAuth buttons redirect to the wrong URL** — check that `APP_BASE_URL`
  matches the address in your browser exactly (scheme, host, path).
- **Warning: "APP_BASE_URL path ... does not match BASE_PATH"** — align them;
  the SPA calls the API under `BASE_PATH` while redirects use `APP_BASE_URL`.
- **Database inspection** — the H2 web console is intentionally not exposed.
  For emergencies:

  ```sh
  docker compose exec smagent bash -c '
    cd /tmp && java -Djarmode=tools -jar /opt/smagent/app.jar extract --destination /tmp/h2tools >/dev/null &&
    java -cp "/tmp/h2tools/BOOT-INF/lib/h2-*.jar" org.h2.tools.Shell \
      -url "jdbc:h2:file:/opt/smagent/data/smagent;AUTO_SERVER=TRUE" \
      -user "$(grep ^db_username /opt/smagent/data/.generated-secrets | cut -d= -f2)" \
      -password "$(grep ^db_password /opt/smagent/data/.generated-secrets | cut -d= -f2)"'
  ```

- **Health** — `docker ps` reports the container's health from
  `<base>/actuator/health`.
