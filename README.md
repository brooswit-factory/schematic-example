# schematic-example

The living example consumer of [brooswit-factory/schematic](https://github.com/brooswit-factory/schematic),
the packwiz-based Minecraft **1.20.1 / Forge** modpack template. It holds a real,
buildable pack — not a fork of the template's docs — so it doubles as the reference
for what a consumer actually has to change.

This repo was created with the **two-remotes recipe**, not the "Use this template"
button, precisely so it shares git history with the template:

```sh
gh repo create brooswit-factory/schematic-example --public \
  --description "Example consumer of the schematic modpack template"
git clone https://github.com/brooswit-factory/schematic.git schematic-example
cd schematic-example
git remote rename origin template
git remote add origin https://github.com/brooswit-factory/schematic-example.git
git push -u origin main
```

`template` is the remote pointing at `brooswit-factory/schematic`. A fresh clone of
*this* repo won't have it — add it back first:

```sh
git remote add template https://github.com/brooswit-factory/schematic.git
```

To pull in template updates:

```sh
git fetch template && git merge template/main
```

| | |
|---|---|
| Minecraft | 1.20.1 |
| Loader | Forge 47.4.10 |
| Mods | Create (`mc1.20.1-6.0.8`) |

## Working on the pack

```sh
packwiz modrinth add <slug>     # e.g. packwiz modrinth add jei
packwiz remove <name>           # e.g. packwiz remove jei
```

Both commands update `index.toml` for you. Commit the resulting `mods/<name>.pw.toml`
along with the changed `index.toml` and `pack.toml` — **CI fails if the index does not
match what is on disk.**

packwiz also has `packwiz curseforge add` and `packwiz url add` if a mod is not on
Modrinth. Note that `packwiz modrinth export` restricts downloads to domains Modrinth
allows, so URL-sourced mods may not be exportable.

```sh
make check     # fails if the committed index.toml is stale
make refresh   # rewrite index.toml after changing files by hand
make build     # -> build/schematic-example-<version>.mrpack
make clean     # remove build/ and bin/
```

### Prerequisites

- [packwiz](https://packwiz.infra.link) — the pack manager. It publishes no tagged
  releases, so this repo pins a commit SHA (`PACKWIZ_REF` in the `Makefile`).
- [Go](https://go.dev) 1.23+, only if you want `make tools` to install that pinned
  packwiz for you. If you already have a packwiz on your `PATH`, it is used instead.

```sh
make tools     # installs the pinned packwiz into ./bin (needs Go)
```

### What is in the repo

```
pack.toml               pack metadata — name, version, Minecraft and Forge versions
index.toml              generated file list with hashes; do not edit by hand
mods/*.pw.toml          one file per mod, pinning a version and its hash
.packwizignore          repo files (docs, CI, Makefile) kept out of the pack
.github/workflows/ci.yml  validates the index and builds the .mrpack on every push
Makefile                the build entry point, shared by humans and CI
```

No jars are committed — `.pw.toml` files reference downloads by URL and hash, and
packwiz fetches them at export time.

### CI

`.github/workflows/ci.yml` runs on every push to `main` and on every pull request. It
installs the pinned packwiz, fails if `packwiz refresh` produces a diff, builds the
pack, and uploads the resulting `.mrpack` as a workflow artifact.

## Releasing

`.github/workflows/release.yml` cuts a release. To publish a new version:

1. Create a GitHub Release with a tag of the form `vX.Y.Z` (e.g. `v0.2.0`).
2. On publish, the workflow:
   - sets the pack version from the tag (in-workflow only — nothing is committed back),
   - builds `build/schematic-example-<version>.mrpack` with the same `make` targets used locally
     and in CI,
   - attaches the `.mrpack` to the GitHub Release as a download,
   - publishes the same file to Modrinth, if Modrinth is configured (see below).

You can also dry-run the whole build-and-package path without creating a Release, via
`workflow_dispatch`:

```sh
gh workflow run release.yml --ref <branch> -f version=0.0.1-test
```

This builds and uploads the `.mrpack` as a workflow artifact but skips the
Release-asset step (there is no Release object to attach to) and the Modrinth publish
step, the same as any run without Modrinth configured.

## Updating a server

`.github/workflows/server-update.yml` runs on a published GitHub release, or on demand
via `workflow_dispatch` (with an optional `version` input; it otherwise falls back to
the `version` field in `pack.toml`). It uploads the packwiz pack to your game server
over FTP, then announces the update and restarts the server over RCON — see
[`server/README.md`](server/README.md) for how to set up the server side of this.

Both halves are independent and **skip cleanly** (the workflow still finishes green)
when their secrets aren't configured, so this works out of the box on a fresh fork —
you opt in by adding the secrets/variables below whenever you're ready.

## Reusable workflows

`ci.yml`, `release.yml`, and `server-update.yml` are thin stubs: each keeps only
its trigger (`on:`) and any `permissions:` it needs, and calls the template's
actual logic via `uses: brooswit-factory/schematic/.github/workflows/reusable-<name>.yml@v1`.
`v1` is a moving tag kept pointed at the template's `main`, so pinning a consumer's
stub to `@v1` picks up fixes to the reusable workflows automatically. The reusable
workflows themselves live only in [brooswit-factory/schematic](https://github.com/brooswit-factory/schematic),
not in this repo.

## Secrets & variables

Everything below is optional. With none of them set, `ci.yml` and `release.yml` still
build and attach the `.mrpack`, and every workflow stays green.

| Name | Kind | Required for | Purpose |
|---|---|---|---|
| `MODRINTH_TOKEN` | secret | `release.yml` | Auth token for publishing to Modrinth (needs the **write-version** scope) |
| `MODRINTH_PROJECT_ID` | variable | `release.yml` | Identifies which Modrinth project to publish to |
| `FTP_HOST` | secret | `server-update.yml` | FTP server hostname for the pack upload |
| `FTP_USER` | secret | `server-update.yml` | FTP username |
| `FTP_PASSWORD` | secret | `server-update.yml` | FTP password |
| `FTP_REMOTE_DIR` | variable | `server-update.yml` | Remote directory to upload the pack to (default `/`) |
| `RCON_HOST` | secret | `server-update.yml` | RCON server hostname for the announce/restart |
| `RCON_PORT` | variable | `server-update.yml` | RCON port (default `25575`) |
| `RCON_PASSWORD` | secret | `server-update.yml` | RCON password |
| `RCON_RESTART_COMMAND` | variable | `server-update.yml` | Command sent after the announce (default `stop`; assumes your host auto-restarts) |
