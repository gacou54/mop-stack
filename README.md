# mop-stack

Docker Compose stack for a World of Warcraft **Mists of Pandaria 5.4.8**
(build 18414) test server: [SkyFire_548](https://github.com/gacou54/SkyFire_548)
with [mod-playerbots](https://github.com/gacou54/mod-playerbots), and MySQL.

> [!WARNING]
> **Highly experimental, for testing only.** This stack is a personal test
> setup for developing SkyFire and mod-playerbots. It is not a supported way to
> run a server, it may not build or start at any given commit, and it changes
> without notice. Defaults are insecure (for example `password` for the bot
> accounts). Do not expect it to work, and do not use it for a public server.

| Service       | Image        | Ports (host)                                       |
|---------------|--------------|----------------------------------------------------|
| `db`          | `mysql:8.4`  | `127.0.0.1:3306`                                   |
| `authserver`  | `mop-stack`  | `3724`                                             |
| `worldserver` | `mop-stack`  | `8085`, RA `127.0.0.1:3443`, SOAP `127.0.0.1:7878` |
| `extractor`   | `mop-stack`  | (profile `tools`, extracts client data)            |

The `mop-stack` image is built with SkyFire_548's `Dockerfile`, with
mod-playerbots compiled in as an out-of-tree module. This repository only pins
which versions of the two go together and how they are configured. Container
and config details are in SkyFire_548's
[doc/Docker.md](https://github.com/gacou54/SkyFire_548/blob/docker-compose/doc/Docker.md).

## Quick start

```bash
cp .env.example .env          # optional; every value has a default
docker compose build          # the first build takes a while

# Once: extract dbc/maps/vmaps from a 5.4.8 client into ./data
WOW_CLIENT_DIR="/path/to/World of Warcraft" docker compose run --rm extractor

docker compose up -d
docker compose logs -f worldserver
```

The first start creates the databases, downloads and imports the world
database (several minutes), and creates the bot accounts and characters.

GM console and an account for yourself:

```bash
docker attach mop-worldserver
SF> account create admin admin
SF> account set gmlevel admin 3 -1
# detach with Ctrl-p Ctrl-q (Ctrl-c stops the server)
```

Point the client's `realmlist.wtf` at `REALM_ADDRESS` (default `127.0.0.1`).

## Versions

`SKYFIRE_REF` and `PLAYERBOTS_REF` in `.env` select what gets built. They
accept a branch, a tag or a commit SHA. Use SHAs for a reproducible build,
and bump them together once a combination is tested.

Rebuild after changing a ref: `docker compose build && docker compose up -d`.

## Developing SkyFire or the module

To build from local checkouts next to this folder (`../SkyFire_548`,
`../mod-playerbots`), add the dev override:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml build
```

or put `COMPOSE_FILE=docker-compose.yml:docker-compose.dev.yml` in `.env` so
every `docker compose` command uses it. `SKYFIRE_SRC` and `PLAYERBOTS_SRC`
change the paths. Builds share a ccache, so a rebuild after an edit only
recompiles what changed.

## Playerbots settings

The `PLAYERBOTS_*` variables in `.env.example` cover the common settings:

| Variable                      | playerbots.conf key                       | Default    |
|-------------------------------|-------------------------------------------|------------|
| `PLAYERBOTS_ENABLE`           | `Playerbots.Enable`                       | `1`        |
| `PLAYERBOTS_RANDOM_BOTS`      | `Playerbots.RandomBots.Enable`            | `1`        |
| `PLAYERBOTS_MAX_BOTS`         | `Playerbots.RandomBots.MaxBots`           | `20`       |
| `PLAYERBOTS_AUTOCREATE`       | `Playerbots.AutoCreate.OnStartup`         | `1`        |
| `PLAYERBOTS_ACCOUNTS`         | `Playerbots.AutoCreate.AccountCount`      | `20`       |
| `PLAYERBOTS_ACCOUNT_PASSWORD` | `Playerbots.AutoCreate.AccountPassword`   | `password` |
| `PLAYERBOTS_MIN_LEVEL`        | `Playerbots.AutoCreate.MinLevel`          | `90`       |
| `PLAYERBOTS_MAX_LEVEL`        | `Playerbots.AutoCreate.MaxLevel`          | `90`       |

Any other key from
[playerbots.conf.dist](https://github.com/gacou54/mod-playerbots/blob/main/conf/playerbots.conf.dist)
or `worldserver.conf.dist` can be set in `.env` as `SFCONF_` plus the key,
with `.` written as `__`:

```bash
SFCONF_Playerbots__RandomBotJoinLfg=1
SFCONF_Rate__XP__Kill=5
```

To use a full config file instead, mount it in a `docker-compose.override.yml`:

```yaml
services:
  worldserver:
    volumes:
      - ./playerbots.conf:/opt/skyfire/etc/playerbots.conf:ro
```

The bot accounts can log in through the authserver like any account. If
`AUTH_PORT` is reachable from other machines, set
`PLAYERBOTS_ACCOUNT_PASSWORD` before the first start.

## Data and reset

- Client data lives in `./data` (`SKYFIRE_DATA_DIR`). `docker compose run --rm
  extractor mmaps` adds mmaps, which takes hours.
- The databases, the cached world dump and the logs are Docker volumes.
  `docker compose down -v` deletes them.
- `mysql/init/01-skyfire.sh` is a copy of SkyFire_548's
  `docker/mysql/init/01-skyfire.sh`. It only runs on an empty `db-data` volume.
