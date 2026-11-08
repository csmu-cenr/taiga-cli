# Makefile reference

The root [Makefile](Makefile) builds, installs, tests, and packages Taiga CLI.
Run these commands from the repository root. Running `make` without a target
runs `build`.

## Requirements

- Make, a POSIX shell, and standard utilities such as `mkdir` and `install`.
- The Go toolchain required by [go.mod](go.mod) (currently Go 1.25.13).
- Git for automatic commit and release timestamp metadata.
- `golangci-lint` on `PATH` for `lint` and `verify`.
- Docker with Compose, a running Docker daemon, and `curl` for `test-integration`.
- macOS signing tools and credentials when signing release binaries.

## Targets

All targets are phony: Make runs their recipes even if a file with the same
name exists.

| Command | What it does |
| --- | --- |
| `make build` | Creates `bin/` and builds `./cmd/taiga` as `bin/taiga`, using `-trimpath` and the version linker flags described below. |
| `make install` | Runs `build`, creates the installation directory, and copies the binary there with mode `0755`. Defaults to `~/.local/bin/taiga`. |
| `make test` | Runs `go test ./...`. |
| `make test-race` | Runs `go test -race ./...`. Requires a platform and toolchain that support Go's race detector. |
| `make lint` | Runs `go vet ./...`, then `golangci-lint run` using the repository configuration. |
| `make test-integration` | Runs `scripts/test-integration.sh` against a dedicated Docker Compose Taiga instance. |
| `make release` | Validates release metadata and runs `./cmd/releasepack` to generate local release artifacts. |
| `make verify` | Runs the `test`, `test-race`, `lint`, and `build` prerequisites. Integration tests and release packaging are separate. |

Without `-j`, `verify` runs its prerequisites in the order listed. With `make -j`,
they may run concurrently.

The integration harness uses Compose project `taiga-cli-e2e` and
`test/e2e/docker-compose.yml`, waits for Taiga on `localhost:19000`, builds a
temporary CLI binary, and runs `go test -tags=integration -count=1 ./test/e2e`.
Its cleanup removes that Compose project's containers and volumes and the
temporary build directory.

## Variables

Override defaults on the command line, for example `make build VERSION=v1.2.3`.
Variables defined with `?=` also accept environment values.

| Variable | Default | Purpose |
| --- | --- | --- |
| `VERSION` | `dev` | Embedded version for local builds; release version for packaging. |
| `COMMIT` | `git rev-parse --short=12 HEAD`, or `unknown` | Embedded commit identifier. |
| `BUILD_DATE` | Current UTC time in `YYYY-MM-DDTHH:MM:SSZ` format | Embedded build date for `build` and `install`. |
| `SOURCE_DATE_EPOCH` | Commit timestamp from `git show -s --format=%ct HEAD` | Unix timestamp passed to the release packager. |
| `PREFIX` | `$(HOME)/.local` | Base installation prefix. |
| `BINDIR` | `$(PREFIX)/bin` | Installation directory for the CLI. |
| `DESTDIR` | Empty (not assigned in the Makefile) | Staging prefix prepended to `BINDIR` during installation. |
| `RELEASE_DIR` | `dist/$(VERSION)` | Release output directory; must be absent or empty. |
| `TARGETS` | Empty | Comma-separated `GOOS/GOARCH` release targets; empty selects the packager defaults. |
| `SIGN_IDENTITY` | Empty | macOS Developer ID Application identity; empty disables signing. |
| `SIGN_KEYCHAIN` | Empty | Keychain searched for the signing identity. |
| `NOTARY_KEY` | Empty | Path to the App Store Connect API key used for notarization. |
| `NOTARY_KEY_ID` | Empty | App Store Connect API key ID. |
| `NOTARY_ISSUER` | Empty | App Store Connect issuer ID, when required by the key. |

`LDFLAGS` is assigned with `:=`. It strips symbol/debug information with `-s -w`
and sets `Version`, `Commit`, and `BuildDate` in
`github.com/KoukeNeko/taiga-cli/internal/version`. A command-line `LDFLAGS`
override replaces this entire value, including the metadata flags.

The `release` target passes `VERSION`, `COMMIT`, and `SOURCE_DATE_EPOCH` to the
packager, which constructs its own build flags and timestamp. It does not use
the Makefile's `BUILD_DATE` or `LDFLAGS`.

## Common workflows

Build and check the local CLI:

```sh
make build
./bin/taiga version
make verify
make test-integration
```

Install under a custom prefix or stage an installation for packaging:

```sh
make install PREFIX=/usr/local
make install BINDIR=/usr/bin DESTDIR=/tmp/taiga-stage
# Staged binary: /tmp/taiga-stage/usr/bin/taiga
```

The destination must be writable. Ensure the installed binary's directory is on
`PATH` to invoke it as `taiga`.

Create release artifacts with explicit metadata:

```sh
make release \
  VERSION=v1.2.3 \
  COMMIT="$(git rev-parse HEAD)" \
  SOURCE_DATE_EPOCH="$(git show -s --format=%ct HEAD)"
```

The Makefile rejects an empty or `dev` version, an empty or `unknown` commit,
and an empty epoch before invoking the packager. The packager performs further
validation. Default targets cover macOS, Linux, and Windows on `amd64` and
`arm64`; artifacts include platform archives, an SPDX SBOM, and `SHA256SUMS`.

To package a subset into a fresh directory:

```sh
make release VERSION=v1.2.3 \
  TARGETS=linux/amd64,linux/arm64 \
  RELEASE_DIR=dist/v1.2.3-linux
```

Signing requires a macOS host and a usable Developer ID identity. Supply
`SIGN_IDENTITY` and optionally `SIGN_KEYCHAIN`; provide `NOTARY_KEY` and
`NOTARY_KEY_ID` together to enable notarization, with `NOTARY_ISSUER` as needed.
Signed macOS artifacts contain Apple's secure timestamp and are not byte
reproducible. See [RELEASING.md](RELEASING.md) for release preparation,
reproducibility details, and publishing steps. `make release` only creates local
artifacts; it does not publish them.
