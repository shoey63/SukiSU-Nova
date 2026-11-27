# Self-hosted runner CI

These workflows run on a Forgejo Actions runner inside WSL on a Windows host, with
no access to `github.com`. Every `uses:` that pointed at a marketplace action has
been replaced with a plain `run:` step.

## Where the CI files live

Everything under `.github/` is executed by the CI itself. The two exceptions are
`image/` and the host scripts, whose headers say where to run them.

```
.github/
├── workflows/       one stage per file, each dispatchable on its own
├── image/           the CI image definitions (docker build on the runner host)
├── scripts/ci/      shell libraries sourced inside a job
│                    common.sh, cnb-raw-git.sh, parallel.sh
│                    plus scripts a single step calls:
│                    ci-checkout.sh, telegram-upload.sh, forgejo-release.sh,
│                    cnb-raw-publish.sh, cnb-raw-manifest.py
├── scripts/         upload-telegram.sh (called by telegram-upload.sh on the
│                    runner), setup-rust-build.sh
├── riscv64-cargo.toml
├── CI.md            this file
└── CI-TEMPLATE.md   template for writing a new workflow (Chinese)
```

Tools that are never run by CI live outside, in `scripts/`: `scripts/dev/` for the
developer machine, `scripts/host/` for the runner host. Two of them relate to CI
and both run on the host, not in a job: `seed-ci-tools.sh` (installs
`/ci/tools/bin/ci-checkout`) and `seed-cargo-cache.sh` (prewarms the cargo
registry, see "Things this network cannot reach").

### Checkout lives on the host, not in the repository

Every job starts with `run: /ci/tools/bin/ci-checkout`, and that file is not in
this repository: a job has to clone before it can read anything the repository
ships, so a checkout helper cannot be sourced from the tree it is fetching.

The source of truth is `.github/scripts/ci/ci-checkout.sh`; the copy on the host
is installed from it:

```bash
# Docker runner with a named volume (`forgejo-ci:/ci`, the layout this repo uses)
CI_TOOLS_VOLUME=forgejo-ci bash scripts/host/seed-ci-tools.sh

# Runner jobs that bind-mount a host directory instead
bash scripts/host/seed-ci-tools.sh
```

`/ci/tools/bin` already holds `forgejo-runner` for the same reason, so a host is
expected to be seeded. A host without this file fails at checkout with
`/ci/tools/bin/ci-checkout: No such file or directory` (exit 127) — every job,
in two seconds, before anything is built.

## Workflow layout

Each stage is its own workflow, so any of them can be dispatched alone:

| File | Builds | Notes |
| --- | --- | --- |
| `build-lkm.yml` | `*_kernelsu.ko` for 8 KMIs | matrix, calls `ddk-lkm.yml`, uses the `ddk-min` images |
| `ksud.yml` | `ksud` | all four targets by default, subset via `targets=` |
| `ksuinit.yml` | `ksuinit` | |
| `manager.yml` | the manager APK | build + repack/sign (no spoofed variant) |
| `publish.yml` | — | rolling `manager-ci` (main) / `manager-dev` (dev) build and Telegram upload, one job |
| `release.yml` | — | orchestrates the stages, then assembles `/ci/runs/<id>/release` |
| `shellcheck.yml`, `clippy.yml`, `clang-format.yml`, `rustfmt.yml` | — | lint jobs |
| `validate.yml` | — | `forgejo-runner validate` on the workflow files, a check that every helper the workflows source still exists, and one that every script they call by path is still on disk |

### Showing the job graph

Forgejo lists a run's jobs but draws no dependencies between them
(forgejo#12717 is still open), so the shape is written where Forgejo does render
it: `determine-version` appends a Mermaid graph to its job summary, built from
the same `needs` the workflow uses. Forgejo 17 renders Mermaid in job summaries
(runner 13.2 or newer); GitHub renders it natively.

It is structure only. Durations, statuses and the click-through to a log stay in
the job list, which owns them — a static graph cannot show what a finished run
did anyway.

`manager.yml` and `publish.yml` take `artifacts_run=<id>`, which seeds this run's
artifact directory from an earlier run. That makes repackaging take minutes
instead of rebuilding the native chain.

## One image for everything

`.github/image/Dockerfile` builds `sukisu-ci:<tag>` from `catthehacker/ubuntu` and
bakes in JDK 21, the Android SDK (`platforms;android-37.0`,
`build-tools;37.0.0`, `cmake;3.22.1`, `cmake;3.31.6`), NDK `29.0.14206865`,
`libclang-21-dev`, Gradle 9.7.1, stable Rust with clippy/rustfmt plus the Android
targets, `nightly`/`nightly-2026-09-23` with `rust-src`, the KPM tools, and
git/curl/python3/zip/openssl/shellcheck/clang-format.

Installing that per job cost several gigabytes of downloads per run and left the
runner idle while a 1.5 GB NDK unpacked. A job now starts in seconds and its only
work is the build itself.

```bash
# on the runner host (WSL), from the repository root
bash .github/image/build-image.sh 6
```

Then point the runner at it:

```ini
# /etc/forgejo-runner/config.yml
runner:
  labels:
    - "ubuntu-latest:docker://sukisu-ci:6"
  capacity: 4
container:
  options: "--add-host=host.docker.internal:host-gateway --volume /srv/forgejo-ci:/ci"
  valid_volumes:
    - /srv/forgejo-ci
```

`capacity: 4` with `MAKEFLAGS=-j6` / `CARGO_BUILD_JOBS=6` per job keeps four
concurrent jobs inside a 32-core host. Build a new tag instead of overwriting
one, so a running job keeps the image it started with.

## What replaces what

| Was | Now |
| --- | --- |
| `actions/checkout` | `/ci/tools/bin/ci-checkout`, which clones from a shared bare mirror in `/ci/repo-cache` with `--shared` |
| `actions/setup-java`, `android-actions/setup-android`, `nttld/setup-ndk`, `Swatinem/rust-cache` | baked into the image; each job runs a `Check toolchain` step |
| `actions/upload-artifact`, `actions/download-artifact` | `ci_artifact_put` / `ci_download_artifact` from `.github/scripts/ci/common.sh`; CNB is the storage and the cross-runner transport, and summaries link to it |
| `softprops/action-gh-release` | files collected under `${CI_ARTIFACT_ROOT}/release` |
| `gh release` publishing | split APKs copied to `${CI_ARTIFACT_ROOT}/manager-<channel>/<sha>` and uploaded to the matching CNB release |

## Things this network cannot reach

`services.gradle.org` redirects to `github.com`. `github.com` itself is
reachable from a job container again — `builtzygisk.yml` clones BuiltZygisk and
its four loader submodules straight from upstream, exactly as their
`.gitmodules` name them — but the rest of the pipeline still avoids it on
purpose, and a step that needs a long fetch from it is asking for trouble:

- **Rust git dependencies** — eight crates come from git on `github.com`.
  `.github/image/mirror-git-deps.sh` mirrors them into `/srv/forgejo-ci/git-mirrors`
  with the revisions `Cargo.lock` pins tagged, the image bakes them in, and
  `git url.<mirror>.insteadOf` plus `cargo net.git-fetch-with-cli` redirect the
  fetches.
- **Gradle** — `gradle-wrapper.properties` would download its distribution from
  `github.com`. The image ships Gradle 9.7.1 and the workflows call `gradle`
  instead of `./gradlew`.
- **KPM tools** — `:app:downloadKpmTools` fetches `kptools-android` and `kpimg`
  from `raw.githubusercontent.com`. The workflow sets `KPM_TOOLS_BASE_URL` to the
  release assets of the mirrored KernelPatch repo, and the image also carries a
  copy at `/opt/kpm-tools` that the job seeds into `app/build/generated/kpmTools`
  after `gradle clean`.
- **crates.io** — `static.crates.io` accepts the TCP connection and then drops
  the TLS handshake (`unexpected eof while reading`), so a cargo download is a
  coin flip and one `hashbrown` download took a builtzygisk run down.
  `scripts/host/seed-cargo-cache.sh` fills the persistent `/ci/tools/cargo`
  registry cache from a container that has access (it clones BuiltZygisk's
  `zygiskd`, runs `cargo fetch` for the job's nightly, and the crates and the
  mirrored git dependencies land in the cache the job already mounts), and
  `builtzygisk.yml` then runs with `CARGO_NET_OFFLINE=1`. Re-run the script when
  `zygiskd` gains a dependency or floats to a version the cache does not have;
  nothing in the image needs rebuilding.

## Artifact transport

Jobs may land on different runners, and a re-run may need another run's output,
so artifacts move through these places, tried in this order by
`ci_download_artifact`:

1. **`/ci` volume** — `/ci/runs/<run_id>/<artifact>/`, written by
   `ci_artifact_put`. Free, but only exists on the host that produced the file.
2. **EasyTier peer** — each runner serves `/ci/runs` on port 18080
   (`scripts/host/artifact-server.sh`), so the other host can fetch the same tree.
   Flat files only: the manifest fetch rejects any entry containing `/`.
3. **CNB** — every artifact is also published as ordinary files in
   `CNB_ARTIFACT_REPO` (default `sukisu-nova/artifacts`): one branch per run
   (`run-<run_id>`), one directory per artifact, with the `manifest.json` that
   lists its files. This is the path that works across hosts *and* across runs,
   and it carries the directory layout the peer cannot. Downloads are anonymous
   raw URLs; no CNB release is created or read anywhere.

   Publishing does not download the stored files: the branch tree is fetched
   with `--filter=blob:none`, so a job pulls a few tree objects and uploads only
   its own files. `ci_artifact_cnb_prune` deletes the branches of runs older
   than the newest `CI_ARTIFACT_KEEP_RUNS` (default 5), once per run per runner,
   never touching the run that is pruning — deleting the branch is also what
   makes the files unreachable instead of accumulating in history. Older runs
   therefore cannot be fetched from CNB any more; they stay on `/ci` only where
   they were produced.
4. **Forgejo Artifacts API** — last resort (zip download); needs a token and is
   the slow endpoint.

CNB is where the artifacts live on purpose: the tunnel to the Forgejo server is
the bottleneck, so no job copies them back into a Forgejo artifact or release.
Job summaries and the release bodies link straight at the CNB download URLs; the
only thing Forgejo still stores is release metadata (title and body) the manager
parses.

Summary links percent-encode one character of the CNB account name
(`cnb.cool/sukisu-nova/artifacts/...` is written `cnb.cool/%73ukisu-nova/...`).
The runner masks every occurrence of a secret value in job output, and on this
instance one of those values is a CNB account name, so a plain link would arrive
as `cnb.cool/***/...`. CNB decodes the escape, so the link still works, and the
masker no longer matches.

```
/ci/repo-cache/                bare mirror per repository, shared by all jobs
/ci/runs/<run_id>/<artifact>/  per-run artifacts (+ manifest.json)
/ci/runs/<run_id>/release/     packaged release files
/ci/runs/<run_id>/manager-ci/   rolling main APKs
/ci/runs/<run_id>/manager-dev/  rolling dev APKs
/ci/tools/cargo/               cargo registry + git cache
/ci/tools/gradle/              Gradle user home
```

`/ci` is a Docker named volume (`forgejo-ci`), not a host directory: 
`container.options` mounts it as `--volume forgejo-ci:/ci`, and `valid_volumes`
has to allow the same name. There is no `/srv/forgejo-ci` on the host, so read
and write through a container:

```bash
docker run --rm -v forgejo-ci:/ci alpine ls /ci/tools/bin
```

`valid_volumes` matters: act silently drops a `--volume` it has not been told to
allow, and the job then writes to a throwaway `/ci` inside the container.

Without `CNB_TOKEN` the puts stay on `/ci` (and warn), which only works when
every job of the run landed on the same host; nothing fails, so a PR build still
runs.

## Secrets and variables

Signing uses repository **variables** (`KEYSTORE` as base64, `KEY_ALIAS`,
`KEYSTORE_PASSWORD`, `KEY_PASSWORD`) with the same names accepted as secrets.
Telegram upload reads `BOT_TOKEN` and `CHAT_ID`, plus `MESSAGE_THREAD_ID` for
normal and dev builds (`MESSAGE_DEV_THREAD_ID` overrides it on dev) and
`MESSAGE_RELEASES_THREAD_ID` for releases; it is skipped when `BOT_TOKEN` or
`CHAT_ID` is empty. Only the single Manager variant is published.

`TG_API_BASE` has to be set too. `api.telegram.org` is not reachable from every
host, so the delivery runs through a Cloudflare Worker proxy: the variable is the
Bot API base **including the worker's secret path**, e.g.
`https://tg.oddnexus.top/<secret>`, and `upload-telegram.sh` appends
`/bot<token>/<method>` to it. The worker only accepts `/<secret>/bot...` and
answers 404 for everything else, so the secret is what keeps the proxy from being
used by anyone else — keep it in the repository variable, never in this file.

Rolling channels: main publishes `manager-ci` and dev publishes `manager-dev`.
Each channel is a branch in the raw repository (`CNB_RAW_REPO`, default
`sukisu-nova/Release`), not a CNB release. The branch is a complete snapshot:
`manifest.json` plus `apks/<files>`. `cnb-raw-publish.sh` force-pushes the
snapshot, so a dev push never replaces the build main was tested with and the
repository does not grow by one binary blob per build. The manager reads
`/-/git/raw/<channel>/manifest.json`; every APK URL in it is a raw link.

Tagged releases use `release-<tag>` branches, and `stable` contains only the
manifest for the newest tag release (its asset URLs point at that immutable
branch). A dispatch from any other ref signs nothing and publishes nothing.

`CNB_RAW_TOKEN` (falling back to `CNB_TOKEN`) needs write access to
`CNB_RAW_REPO`; keeping it separate lets the publication account differ from
the one used for the internal artifact transport. `CNB_ARTIFACT_REPO` (default
`sukisu-nova/artifacts`) is separate and holds the build-to-build transport as
`run-<run_id>` branches of ordinary files; those are not published assets.
`CNB_ARTIFACT_TOKEN` overrides the token for that repository and falls back to
`CNB_RAW_TOKEN`, then `CNB_TOKEN`. Publishing back to Forgejo uses
`RELEASE_API_TOKEN` when set, and the built-in `GITHUB_TOKEN` otherwise.

Telegram delivery is a plain workflow step, not a CNB pipeline: the APKs are
already in the workspace when the build is published, so `telegram-upload.sh`
sends them from the runner (`apks/split` for a rolling build, `release/` for a
tag). A marker under `/ci/state` keeps a re-run of the same build from posting
twice, and the step is `continue-on-error` so a delivery failure only costs the
notification. CNB is used for artifact storage only — no build is triggered
there, which is what got the old repository removed.

## Disabled workflows

Renamed to `*.yml.disabled`; Forgejo and GitHub both ignore those names.

- `npm.yml` publishes to the npm registry with provenance — needs an OIDC token
  from GitHub that does not exist here.
- `ksud-extra.yml` builds with `cross`, installed from a pinned `github.com`
  commit, and needs a Docker socket inside the job container. Nothing calls it.

`builtzygisk.yml` is *not* disabled: `ci.yml` / `release.yml` both call it, and
it clones `external/BuiltZygisk` plus its four loader submodules from their
upstream repositories on github.com.

## Commit signing

Commits are signed with GPG (`commit.gpgsign=true`, key
`1C7DD4CE810618C3`). Forgejo only shows the *Verified* badge when the public key
is registered on the account, which also requires proving ownership: the key's
email address is not on the account, so the key was added with a signature over
the challenge token that the API returns.

```bash
# register (or replace) the key on the account
gpg --armor --export 1C7DD4CE810618C3 > pubkey.asc
curl -X POST -H "Authorization: token $FORGEJO_TOKEN"   -H 'Content-Type: application/json'   -d "$(jq -Rs '{armored_public_key: .}' pubkey.asc)"   https://git.zako.shirkneko.top:20004/api/v1/user/gpg_keys
# if it answers "None of the emails ... could be found", sign the token it prints
printf '%s' '<token>' | gpg --armor --detach-sign > token.asc
# and add "armored_signature": "<token.asc contents>" to the JSON body
```
