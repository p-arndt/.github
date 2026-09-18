# p-arndt/.github

Reusable GitHub Actions workflows (`on: workflow_call`), so each repository only
needs a ~10-line caller workflow. Every third-party action is pinned to a full
commit SHA with a `# vX.Y.Z` comment; every job declares minimal `permissions:`.

## Workflows

| Workflow | Purpose | Caller must grant |
| --- | --- | --- |
| [`go-ci.yml`](.github/workflows/go-ci.yml) | vet, test (-race), build on an OS matrix; gofmt + golangci-lint | `contents: read` |
| [`go-release.yml`](.github/workflows/go-release.yml) | tag push -> stamp verify, test, static cross-compile, sha256 checksums, optional signing, GitHub Release, optional major-tag move | `contents: write` |
| [`rust-ci.yml`](.github/workflows/rust-ci.yml) | Cargo.lock check, rustfmt, clippy -D warnings, test on an OS matrix | `contents: read` |
| [`rust-release.yml`](.github/workflows/rust-release.yml) | tag push -> stamp verify, per-target archives (musl/win/mac), checksums, optional GHCR images, GitHub Release | `contents: write` (+ `packages: write` with images) |
| [`container-release.yml`](.github/workflows/container-release.yml) | tag push -> stamp verify, buildx + push to GHCR (semver + latest), GitHub Release | `contents: write`, `packages: write` |
| [`sveltekit-ci.yml`](.github/workflows/sveltekit-ci.yml) | pnpm (version from `packageManager`), frozen install, check, lint, test, build | `contents: read` |

### go-ci.yml

| Input | Type | Default | |
| --- | --- | --- | --- |
| `go-version-file` | string | `go.mod` | file setup-go reads the version from |
| `os` | string (JSON) | `["ubuntu-latest", "macos-latest", "windows-latest"]` | test matrix |
| `race` | boolean | `true` | `go test -race` |
| `gofmt` | boolean | `true` | fail on `gofmt -l .` output |
| `lint` | string | `auto` | golangci-lint: `true`, `false`, or `auto` (only if `.golangci.yml`/`.yaml` exists) |
| `golangci-lint-version` | string | `v2.13.2` | pinned linter release |
| `configure-git` | boolean | `false` | throwaway git identity for suites that commit |
| `timeout-minutes` | number | `15` | per test job |

### go-release.yml

| Input | Type | Default | |
| --- | --- | --- | --- |
| `binary` | string | **required** | executable and asset name |
| `main` | string | `.` | main package, e.g. `./cmd/shenv` |
| `targets` | string (JSON) | 6 targets: linux/darwin/windows x amd64/arm64 | GOOS/GOARCH list |
| `artifact-style` | string | `archive` | `archive`: `<bin>_<ver>_<os>_<arch>.tar.gz`/`.zip`; `raw`: `<bin>-<os>-<arch>[.exe]` |
| `archive-extra-files` | string | `''` | space-separated files added to each archive |
| `ldflags` | string | `-s -w -X {module}/internal/buildinfo.Version={version} ...Commit={commit} ...Date={date}` | placeholders `{module}` `{version}` `{commit}` `{date}` |
| `go-version-file` | string | `go.mod` | |
| `test` | boolean | `true` | vet + test before building |
| `test-os` | string (JSON) | `["ubuntu-latest"]` | runners for those tests |
| `race` | boolean | `false` | `-race` for those tests |
| `configure-git` | boolean | `false` | throwaway git identity on the test runners |
| `verify-tag` | boolean | `true` | `stamp verify` the tag against the committed version |
| `changelog` | string | `stamp` | `stamp` (annotated tag message) or `git-cliff` |
| `cliff-config` | string | `cliff.toml` | |
| `sign` | boolean | `false` | run `sign-command` after checksums |
| `sign-command` | string | `''` | gets `$CHECKSUMS`, `$VERSION`, `$RELEASE_SIGNING_KEY`; files written to `dist/` are published |
| `move-major-tag` | boolean | `false` | force-move `v<major>` after a non-prerelease |

Secrets: `signing-key` (-> `$RELEASE_SIGNING_KEY`), `major-tag-token` (Contents + Workflows read/write).
Output: `version`.

### rust-ci.yml

| Input | Type | Default | |
| --- | --- | --- | --- |
| `os` | string (JSON) | `["ubuntu-latest", "macos-latest", "windows-latest"]` | matrix |
| `toolchain` | string | `stable` | installed with the runner's rustup |
| `locked` | boolean | `true` | fail if Cargo.lock is stale |
| `fmt` | boolean | `true` | `cargo fmt --all --check` (Linux leg) |
| `clippy` | boolean | `true` | clippy with `-D warnings` |
| `clippy-args` | string | `--workspace --all-targets` | |
| `test` | boolean | `true` | `cargo test` |
| `test-args` | string | `--workspace` | |

### rust-release.yml

| Input | Type | Default | |
| --- | --- | --- | --- |
| `name` | string | **required** | archive prefix: `<name>-<ver>-<target>.tar.gz`/`.zip` |
| `binaries` | string | **required** | space-separated binaries packed into every archive |
| `targets` | string (JSON) | x86_64/aarch64 linux-musl, x86_64 windows-msvc, aarch64/x86_64 apple-darwin | `[{"os": ..., "target": ...}]` |
| `extra-files` | string | `README.md LICENSE` | added when present (globs allowed) |
| `cargo-args` | string | `--workspace --locked` | |
| `toolchain` | string | `stable` | |
| `verify-tag` | boolean | `true` | stamp verify |
| `changelog` | string | `stamp` | `stamp` or `git-cliff` |
| `cliff-config` | string | `cliff.toml` | |
| `images` | string (JSON) | `[]` | Dockerfile targets pushed as `ghcr.io/<owner>/<target>` |
| `image-platforms` | string | `linux/amd64,linux/arm64` | |
| `dockerfile` | string | `Dockerfile` | |

Output: `version`.

### container-release.yml

| Input | Type | Default | |
| --- | --- | --- | --- |
| `image` | string | repository name | `ghcr.io/<owner>/<image>` |
| `context` | string | `.` | |
| `dockerfile` | string | `Dockerfile` | |
| `target` | string | `''` | Dockerfile stage |
| `platforms` | string | `linux/amd64` | QEMU is set up for anything else |
| `build-args` | string | `''` | newline-separated; `VERSION=<version>` is always passed |
| `cache` | string | `gha` | `gha`, `registry` (inline cache on `:latest`) or `none` |
| `verify-tag` | boolean | `true` | stamp verify |
| `changelog` | string | `stamp` | `stamp`, `git-cliff` or `none` (no GitHub Release) |
| `cliff-config` | string | `cliff.toml` | |

Image tags: `<version>`, `<major>.<minor>` and `latest`; pre-releases (`v1.2.0-rc.1`) get only `<version>`.
Outputs: `version`, `image`, `digest`.

### sveltekit-ci.yml

| Input | Type | Default | |
| --- | --- | --- | --- |
| `node-version` | string | `24` | |
| `working-directory` | string | `.` | |
| `env` | string | `''` | newline-separated `KEY=VALUE` build placeholders (not secrets) |
| `setup-command` | string | `''` | runs after install (e.g. Paraglide compile) |
| `check` / `check-command` | boolean / string | `true` / `pnpm check` | |
| `lint` / `lint-command` | boolean / string | `true` / `pnpm lint` | |
| `test` / `test-command` | boolean / string | `true` / `pnpm test:unit --run` | |
| `build` / `build-command` | boolean / string | `false` / `pnpm build` | |
| `timeout-minutes` | number | `20` | |

pnpm's version is read from `packageManager` in `package.json`; the project must set it.

## Calling a workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  release:
    uses: p-arndt/.github/.github/workflows/go-release.yml@v1
    permissions:
      contents: write
    with:
      binary: hop
      changelog: git-cliff
```

More in [`examples/`](examples/): `hop-ci.yml`, `hop-release.yml`, `ccu-release.yml`
(raw binaries, 9 targets), `shenv-release.yml` (signing), `stamp-release.yml`
(major tag), `minisuite-ci.yml`, `minisuite-release.yml` (GHCR images),
`uprox-ci.yml`, `uprox-release.yml` (container, gated on the repo's own CI).

Notes:

- A called workflow never gets more permissions than the calling job grants, so
  release callers have to grant `contents: write` (and `packages: write` for
  images) on the calling job.
- Secrets are not inherited implicitly. Pass the ones a workflow declares
  (`secrets: signing-key: ${{ secrets.RELEASE_SIGNING_KEY }}`) rather than
  `secrets: inherit`, so each workflow sees only what it needs.
- The workflows set their own `concurrency` groups (`go-release-<repo>`,
  `go-ci-<repo>-<ref>`, ...). Do not give the caller a concurrency group with
  the same name, or the caller and the called workflow wait for each other.
- Release workflows only run for tag pushes and fail otherwise.
- `stamp` verification and notes need a `VERSION` file, a `package.json` or a
  `.stamp.yml` in the calling repository. Set `verify-tag: false` for
  repositories that do not use stamp.

## Versioning

Callers reference a major tag (`@v1`), never `@main`:

1. Tag every release with a full version: `git tag -a v1.2.0 -m "v1.2.0"`.
2. Move the major tag onto it: `git tag -f -a v1 -m "v1.2.0" v1.2.0 && git push -f origin v1`.
3. Only breaking changes (removed/renamed inputs, changed defaults that alter
   output names) get a new major (`v2`); `v1` keeps working for existing callers.

A caller that wants the same guarantee as the pinned actions inside can pin a
full commit SHA instead: `uses: p-arndt/.github/.github/workflows/go-ci.yml@<sha> # v1.2.0`.

Keep the action pins current with Dependabot (`package-ecosystem: github-actions`).

## Visibility: make this repository public

Reusable workflows in a **private** repository are only callable from other
repositories if that repository allows it: Settings -> Actions -> General ->
Access -> "Accessible from repositories owned by the user 'p-arndt'". Even then
public repositories cannot call it.

This repository contains no secrets (every secret is passed in by the caller),
so the recommended setup is to make it **public**. Then every repository, public
or private, can call the workflows without further configuration.
