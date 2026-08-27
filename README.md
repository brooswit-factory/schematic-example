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

| | |
|---|---|
| Minecraft | 1.20.1 |
| Loader | Forge 47.4.10 |
| Mods | Create (`mc1.20.1-6.0.8`) |

## Updating from the template

### Pulling template changes

The template keeps evolving — Makefile fixes, `server/` tooling, README
clarifications, improvements to the reusable workflows. This repo pulls those in
with:

```sh
git fetch template
git merge template/main                                  # expect conflicts
git checkout ORIG_HEAD -- pack.toml index.toml mods      # own pack content wins, conflicted or not
git checkout --theirs -- <other conflicted files not customised here>
make refresh
git add -A
git commit
```

`ORIG_HEAD` is the branch tip as it stood immediately before the merge — checking out
`pack.toml`, `index.toml`, and `mods/` from it restores **this pack's own content**
there no matter what happened during the merge.

That last point matters: git only reports a conflict on a file **both sides changed**.
If the template deletes or changes a file this repo never touched — most importantly a
mod file in `mods/`, or `index.toml` — there's no conflict, and the merge silently
applies the template's version, which for a deleted mod means it's just gone with
nothing to resolve. That's the intended behaviour for tooling files (`Makefile`,
`server/`, workflow stubs) that should track the template automatically, but it's
exactly why the `git checkout ORIG_HEAD -- pack.toml index.toml mods` step above is
unconditional rather than only for conflicted paths — it protects the pack content
whether or not git flagged a conflict on it. A file the template *adds* under
`mods/` (its own placeholder for an empty pack, for instance) survives this checkout
untouched too — the checkout only overlays paths present in `ORIG_HEAD`, it doesn't
delete extras — and can safely be left in place.

For the `--theirs` line, "other conflicted files not customised here" means
`Makefile`, `server/README.md`, and `server/start.sh` — take the template's version
of these unless there are local edits worth preserving. `README.md` is the one file
almost certainly customised here: it's reconciled by hand instead of `--theirs`
wholesale — the pack-specific text is kept and template improvements are folded in
where they still apply, as was done for this very section. If
`.github/workflows/tag-v1.yml` was previously deleted here and the template has since
changed it, you'll see it as a delete/modify conflict instead of a clean merge; per
[Template-only files](#template-only-files) below, deleting it isn't necessary in the
first place, so the simplest fix is to keep the template's version
(`git checkout --theirs -- .github/workflows/tag-v1.yml`) rather than re-deleting it.

### The pinned workflow stubs

`ci.yml`, `release.yml`, and `server-update.yml` are thin stubs: each keeps only its
trigger (`on:`) and any `permissions:` it needs, and calls the template's actual logic
via `uses: brooswit-factory/schematic/.github/workflows/reusable-<name>.yml@v1`. `v1`
is a moving tag kept pointed at the template's `main`, so pinning these stubs to `@v1`
picks up fixes to the reusable workflows automatically, with zero merge effort. The
reusable workflows live here too, as template-only copies under
`.github/workflows/reusable-*.yml` — see [Template-only files](#template-only-files)
below — but the stubs call them at their upstream location, pinned `@v1`, so these
local copies never run.

`v1` promises backwards-compatible inputs, secrets, and variable names; anything that
would break a consumer's workflow ships as a `v2` instead. This repo tracks `@v1`;
pinning a stub's `uses:` line to a specific tag or commit SHA instead would opt out of
moving updates.

## Template-only files

Four files under `.github/workflows` are template-only, and safe to leave in place:

`tag-v1.yml` keeps the `v1` tag on **brooswit-factory/schematic** pointed at its own
`main`. It is guarded by a `github.repository` check, so it is inert here — the job is
skipped entirely, and it creates no tag in this repo.

`reusable-ci.yml`, `reusable-release.yml`, and `reusable-server-update.yml` are
`workflow_call`-only definitions — nothing invokes them by local path. This repo's
stubs (`ci.yml`, `release.yml`, `server-update.yml`) call them **upstream**, at
`brooswit-factory/schematic/.github/workflows/reusable-<name>.yml@v1`, so these local
copies never run.

There's no need to delete any of the four — doing so gains nothing, since they don't
run locally either way, and deleting one only creates work later: a
`git merge template/main` does not restore a file you deleted (the deletion simply
persists, merge or no merge) until the template itself changes that file, at which
point the merge stops with a delete/modify conflict you have to resolve by hand.
Leaving the four alone avoids that conflict entirely, on every future merge.

`ci.yml`, `release.yml`, and `server-update.yml` are the three stubs a consumer of
this template needs to care about — the four template-only files above need no
attention at all.

## Working on the pack

```sh
packwiz modrinth add <slug>     # e.g. packwiz modrinth add jei
packwiz remove <name>           # e.g. packwiz remove jei
```

Both commands update `index.toml` for you. Commit the resulting `mods/<name>.pw.toml`
along with the changed `index.toml` and `pack.toml` (`packwiz refresh` writes the new
index hash into `pack.toml`'s `[index]` block) — **CI fails if the index does not
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
pack.toml                           pack metadata — name, version, Minecraft and Forge versions
index.toml                          generated file list with hashes; do not edit by hand
mods/*.pw.toml                      one file per mod, pinning a version and its hash
.packwizignore                      repo files (docs, CI, Makefile) kept out of the pack
.github/workflows/ci.yml            validates the index and builds the .mrpack on every push
.github/workflows/release.yml       cuts a release (see Releasing below)
.github/workflows/server-update.yml keeps a game server in sync (see server/README.md)
.github/workflows/tag-v1.yml        template-only (see Template-only files above)
.github/workflows/reusable-*.yml    template-only (see Template-only files above)
Makefile                            the build entry point, shared by humans and CI
server/                             sample scripts for running/updating a Forge server for this pack
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
   - publishes the same file to Modrinth, if Modrinth is configured (see
     [Secrets & variables](#secrets--variables) below).

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
when their secrets aren't configured, so this works out of the box on a fresh clone —
you opt in by adding the secrets/variables below whenever you're ready.

## Secrets & variables

Everything below is optional. With none of them set, `ci.yml` still builds and uploads
the `.mrpack` as a workflow artifact, `release.yml` still builds and attaches it to the
GitHub Release, and any workflow step that needs a secret **skips cleanly** (the run
still finishes green) when it isn't configured.

| Name | Kind | Used by | Purpose |
|---|---|---|---|
| `MODRINTH_TOKEN` | secret | `release.yml` | Auth token for publishing to Modrinth (needs the **write-version** scope) |
| `MODRINTH_PROJECT_ID` | variable | `release.yml` | Identifies which Modrinth project to publish to |
| `FTP_HOST` | secret | `server-update.yml` | FTP server hostname for the pack upload |
| `FTP_USER` | secret | `server-update.yml` | FTP username |
| `FTP_PASSWORD` | secret | `server-update.yml` | FTP password |
| `FTP_REMOTE_DIR` | variable | `server-update.yml` | Remote directory to upload the pack to (default `/`) |
| `RCON_HOST` | secret | `server-update.yml` | RCON server hostname for the announce/restart |
| `RCON_PASSWORD` | secret | `server-update.yml` | RCON password |
| `RCON_PORT` | variable | `server-update.yml` | RCON port (default `25575`) |
| `RCON_RESTART_COMMAND` | variable | `server-update.yml` | Command sent after the announce (default `stop`; assumes your host auto-restarts) |
