# tombi

Container images with [Tombi](https://tombi-toml.github.io/tombi/), a formatter, linter and language server for TOML. `tombi format` rewrites TOML files in one consistent style, and for files with a JSON Schema, such as `Cargo.toml` and `pyproject.toml`, it also sorts keys and tables in the order the schema gives. `tombi lint` reports syntax errors and values that do not match the schema. The images build Tombi from the source of each Tombi release with a pinned Rust release. They are built on Ubuntu and Alpine for `linux/amd64` and `linux/arm64`, and rebuilt for a new Tombi release at most once a week (see [Updates](#updates)) and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the Tombi project. Report problems with the image in this repository and problems with Tombi itself in the [Tombi issue tracker](https://github.com/tombi-toml/tombi/issues).

## Quick start

Format every TOML file in the current directory and below:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/tombi format
```

The image's default arguments are `--help`, so name a subcommand: `format`, `lint`, `lsp`, `completion` or `cache`. Without paths, `format` and `lint` take every `*.toml` file below the directory of the project configuration they find (see [Configuration](#configuration)), or below the working directory, `/work`, when there is none. They skip the files that `.ignore` excludes and, inside a git repository, those that `.gitignore` excludes. Pass files or directories to work on only those.

Check the formatting without changing anything, as in CI; `--diff` also prints the changes:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/tombi format --check
```

Lint the files:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/tombi lint
```

Both exit with 0 when every file passes and with 1 when one does not: a file that `format --check` would change, or a file in which `lint` finds an error. Warnings do not change the exit code of `lint` unless you pass `--error-on-warnings`. `--diagnostics-format` writes the findings as `json`, `json-lines`, `sarif`, `junit`, `gitlab` (Code Quality) or `github` (workflow commands); all but `pretty` and `github` also need `--diagnostics-file <path>`.

Format text from standard input, without mounting anything. `--stdin-filename Cargo.toml` makes Tombi apply the schema for that file name:

```sh
echo 'a=1' | docker run --rm -i ghcr.io/randomcontainers/tombi format -
```

## Schemas and network access

Tombi validates and sorts a file by a JSON Schema when it finds one for it:

- The schemas for `Cargo.toml`, `pyproject.toml` and `tombi.toml` are built into the binary, so they work without network access. For some tables they refer to further SchemaStore schemas that are not built in: the `[tool.*]` tables of `pyproject.toml`, such as `[tool.ruff]` and `[tool.uv]`, and the `[lints]` and `[workspace.lints]` tables of `Cargo.toml`. Tombi fetches those schemas or takes them from the cache. With `--offline` and no cached copy, Tombi neither validates nor sorts those tables, and prints no warning.
- For other files Tombi reads the [SchemaStore](https://www.schemastore.org/) catalog from `https://www.schemastore.org/api/json/catalog.json` and fetches the schema that the catalog names for the file.
- A `#:schema <url>` comment at the top of a file, or a `[[schemas]]` entry in `tombi.toml`, names a schema by URL or by a path relative to the file or the configuration.

Fetched catalogs and schemas are cached for 24 hours in `/cache/tombi`. Mount a volume there to keep them between runs, for example `-v tombi-cache:/cache`. `--offline` (or `TOMBI_OFFLINE=true`) uses only the built-in schemas, the cache and local files, and fetches nothing.

A TOML file can name any URL in a `#:schema` comment, and Tombi fetches it. Tombi refuses URLs that resolve to private or special-use addresses unless `TOMBI_SCHEMA_TRUSTED_HOSTS` lists the host, but for files you did not write, run with `--offline` or with `docker run --network none`. When a schema on `raw.githubusercontent.com` answers 401, 403 or 404, Tombi retries with the token in `GH_TOKEN` or `GITHUB_TOKEN`, so pass one of them into the container only when you need schemas from private repositories. The image has no `gh`, so Tombi cannot take a token from it.

## Configuration

Tombi reads `.tombi.toml`, `tombi.toml` or `.config/tombi.toml`, or a `[tool.tombi]` table in `pyproject.toml`, from the working directory or the nearest parent that has one. In the container the parents of `/work` are the image's own directories, so mount the root of the project.

```toml
toml-version = "v1.0.0"

[files]
exclude = ["vendor/**"]

[format.rules]
indent-width = 4

[[schemas]]
path = "schemas/app.schema.json"
include = ["app.toml"]
```

Upstream's [configuration reference](https://tombi-toml.github.io/tombi/docs/configuration/) lists every key.

## CI and pre-commit

The image has a shell, so CI systems that run their scripts inside the job's image can use it. In GitLab CI, clear the entrypoint:

```yaml
tombi:
  image:
    name: ghcr.io/randomcontainers/tombi:<version>
    entrypoint: [""]
  script:
    - tombi format --check --offline
    - tombi lint --offline
```

Tombi releases often, and a minor release can format files differently or report new problems, so a job on a moving tag can start to fail without a change in your files. Replace `<version>`, here and below, with the newest exact version tag (see [Tags](#tags)), and update it by hand.

pre-commit runs a `docker_image` hook with the repository mounted at `/src` as the working directory and, with rootful Docker, as your user:

```yaml
repos:
  - repo: local
    hooks:
      - id: tombi-format
        name: tombi format
        language: docker_image
        entry: ghcr.io/randomcontainers/tombi:<version> format --offline
        types: [toml]
```

Docker reuses an image it has already pulled, so a moving tag such as `latest` would stay on the first version it pulled.

## Language server

`tombi lsp` speaks the Language Server Protocol on standard input and output. An editor that starts the server in a container needs the files at the same paths inside it as on the host, because the protocol names files by path:

```sh
docker run --rm -i --user "$(id -u):$(id -g)" -v "$PWD:$PWD" -w "$PWD" ghcr.io/randomcontainers/tombi lsp
```

## What is in the image

- `tombi` in `/usr/local/bin`, built from the release tag with cargo's default release profile. Upstream's own builds add link-time optimization and strip symbols, so this binary is larger. It is linked against the distro's C library and libgcc_s: glibc on Ubuntu, musl on Alpine.
- The schemas for `Cargo.toml`, `pyproject.toml` and `tombi.toml`, compiled into the binary.

`tombi --version` reports the release, which the build takes from the release tag. Upstream's release workflow also writes the version into the workspace's `Cargo.toml`, which this build leaves at its placeholder, `0.0.0-dev`. That is the version `tombi lsp` reports to editors, and the links it offers to the built-in schemas open Tombi's `main` branch instead of the release tag.

Copying `/usr/local/bin/tombi` into another image with `COPY --from` works only on a base with the same C library, at the same or a newer version, and libgcc_s. For a single file that runs on any Linux system, use the statically linked builds that Tombi attaches to each [release](https://github.com/tombi-toml/tombi/releases), `tombi-cli-<version>-x86_64-unknown-linux-musl.tar.gz` and `tombi-cli-<version>-aarch64-unknown-linux-musl.tar.gz`; `gh attestation verify <file> --repo tombi-toml/tombi` checks the build provenance that upstream records for them.

`/usr/local/share/randomcontainers/tombi/buildinfo` has the commit, the Rust release with the URL and SHA-256 of each archive it was installed from, the output of `rustc -vV`, the cargo-auditable version, the version tombi reports, and every crate the binary records, with its version, whether it is linked in or used only during the build, and its license.

## Default or slim

tombi's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a Tombi release such as `1.9.1`. `<minor>` and `<major>` are its shorter forms, `1.9` and `1`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Tombi version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time. In CI, pin at least the exact version (see [CI and pre-commit](#ci-and-pre-commit)).

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Each image contains tombi built for its own architecture and is built natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. `lint` and `format --check` only read files, so for them the mount can be read-only (`-v "$PWD:/work:ro"`). `format` writes the files it changes, and only those its UID may write, so run it as your own user:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Tombi release and are rebuilt when the base image changes. Switch to root to install more, then back. This adds `git`, for a script that checks only the files changed on a branch:

```dockerfile
FROM ghcr.io/randomcontainers/tombi:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends git \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache git`. The entrypoint is `["tini", "--", "tombi"]`; set your own `ENTRYPOINT` if your image runs a script. To pick up new Tombi releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/tombi/` holds the version, the source URLs, the build details, the license files, `crate-sources/` with the crates under MPL-2.0, and `runtime-deps`, the list of distro packages tombi needs at run time: empty on Ubuntu, and on Alpine musl and libgcc_s by their `so:` names.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/tombi:latest \
  --repo randomcontainers/tombi --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/tombi:latest --format '{{ json .SBOM }}'
```

tombi is built with [cargo-auditable](https://github.com/rust-secure-code/cargo-auditable), which stores the list of crates in the binary's `.dep-v0` section. Syft, Grype and Trivy read that list, so SBOM tools and image scanners see those crates. The list comes from the whole Tombi workspace, so it also names crates that tombi does not compile, and Tombi's own crates appear there as version `0.0.0-dev`. The Rust standard library is not in the list, and scanners do not report its advisories; `buildinfo` names the Rust release that built tombi, and [Updates](#updates) says when it moves. Follow Tombi's [security advisories](https://github.com/tombi-toml/tombi/security) as well.

Tombi publishes no source tarball, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [tombi-toml/tombi](https://github.com/tombi-toml/tombi) and stops unless the tag points to the commit recorded in `package.yml`. cargo downloads the crates in the tag's `Cargo.lock` and checks each against the checksum there, and the build that follows has no network access. It also stops when upstream's `.cargo/config.toml`, which applies to the build, differs from the SHA-256 pinned in `cargo-config.sha256`.

Rust comes from static.rust-lang.org. The release manifest named in `package.yml` is checked against the SHA-256 recorded there and against its signature, made with the Rust release key (fingerprint `108F 6620 5EAE B0AA A8DD 5E1C 85AB 96E6 FA1B E5FE`, kept in [keys/rust.gpg](keys/rust.gpg)). Each archive installed from it is checked against the SHA-256 that the manifest lists. cargo-auditable, which runs only during the build, is compiled from its crates.io crate, checked against the SHA-256 in `package.yml`, with the crates listed in its own `Cargo.lock`.

## Updates

The project checks the releases of [tombi-toml/tombi](https://github.com/tombi-toml/tombi/releases) every 15 minutes. A release is picked up once it is 48 hours old, and at most once a week: while the release in `package.yml` is less than 7 days old, newer releases wait, and then the newest release that is 48 hours old is taken and those in between are skipped. The new version and the commit of its tag are committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The Rust release that builds tombi follows Rust's stable channel. Each new stable release is committed to `package.yml` once it is a day old, and the images are rebuilt with it. cargo-auditable is pinned by hand. Tombi declares no minimum Rust version and builds its releases with the newest stable Rust, so a Tombi release can need a newer Rust than the pinned one. It then does not build until the pin moves.

A release also waits for a person to update this repository when it:

- adds a crate under a license that the image's label does not cover
- adds a crate without a license file, or a crate with a `links` key, which builds or links native code
- moves ring to another version
- changes the license of a crate that has files in [licenses](licenses), such as the license psl declares
- drops a crate that has files or a line in [licenses](licenses)
- changes upstream's `.cargo/config.toml`
- stops embedding the SchemaStore schemas

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags. While a release waits for a person, the images of the current version are not rebuilt either: the new version is already in `package.yml`, and every rebuild builds it.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  --build-arg RUST_TOOLCHAIN_VERSION=<rust-toolchain version from package.yml> \
  --build-arg RUST_TOOLCHAIN_SHA256=<rust-toolchain sha256 from package.yml> \
  --build-arg CARGO_AUDITABLE_VERSION=<cargo-auditable version from package.yml> \
  --build-arg CARGO_AUDITABLE_SHA256=<cargo-auditable sha256 from package.yml> \
  -t tombi:local .
```

Use `Dockerfile.alpine`, with the same arguments, for the Alpine image. The build downloads 120 to 135 MB of Rust archives on Ubuntu and about 170 MB on Alpine, and the crates in `Cargo.lock` for the build platform, up to about 40 MB. `--build-arg JOBS=<n>` limits the number of crates cargo compiles in parallel.

## Licenses

Tombi is available under the MIT license. The binary also contains the Rust crates Tombi depends on. Most of them are under `MIT OR Apache-2.0` or MIT, and the image's label takes MIT wherever a crate offers it. The others are:

- Apache-2.0: similar, sync_wrapper and unicode-linebreak, which are Apache-2.0 only, and pep440_rs, pep508_rs, ryu and self_cell, whose other choices are not MIT.
- `Apache-2.0 AND ISC`: ring, which also contains code from BoringSSL and fiat-crypto under their own notices. ISC: rustls-webpki and untrusted.
- BSD-3-Clause: subtle. Unicode-3.0: the ICU4X crates and unicode-ident. `Apache-2.0 WITH LLVM-exception AND MIT`: async-codec-lite.
- MPL-2.0: option-ext and version-ranges, and Mozilla's Public Suffix List, which psl compiles in although the crate declares `MIT OR Apache-2.0`.

The list of crates in the binary also names crates that other parts of the Tombi workspace enable, such as the gloo crates of its WebAssembly build and zlib-rs (Zlib), whether or not tombi compiles them; the label covers them because the build checks it against the whole list. The Rust standard library and the crates it uses are linked in too. They are under `MIT OR Apache-2.0` or another choice that includes MIT, except compiler-builtins, which is under `MIT AND Apache-2.0 WITH LLVM-exception`. The built-in schemas for `Cargo.toml` and `pyproject.toml` are copies of SchemaStore's schemas, which SchemaStore publishes under the Apache License 2.0.

The image's license label is `MIT AND Apache-2.0 AND Apache-2.0 WITH LLVM-exception AND BSD-3-Clause AND ISC AND Unicode-3.0 AND Zlib AND MPL-2.0`. The build stops when a crate in the binary's list has a license that this label does not cover.

The license files are in `/usr/local/share/randomcontainers/tombi/licenses/`:

- Tombi's `LICENSE` from the release tag.
- `schemastore/` with SchemaStore's `LICENSE` and `NOTICE`.
- `crates/<crate>-<version>/` with the license files of every crate that the binary records, including those used only while it is built, and `crates/CRATES.tsv`, which lists each of them with its license.
- `rust/` with the license files of rust-lang/rust, the standard library and the crates it vendors, from the `rust-src` archive of the Rust release that built tombi. LLVM's libunwind is left out, because the dynamically linked binary does not contain it.

Three crates publish no license file: defmt-parser, gloo-net and gloo-utils. This repository keeps their license texts in [licenses](licenses), together with SchemaStore's files, the MPL-2.0 text and a notice for the Public Suffix List in psl. The build copies them into the image. The Ubuntu and Alpine packages in the image keep their own licenses.

The source for each image:

- tombi: built from the `v<version>` tag of [tombi-toml/tombi](https://github.com/tombi-toml/tombi). `/usr/local/share/randomcontainers/tombi/source` names the commit of that tag, the Rust release manifest, the `rust-src` archive with the source of the Rust standard library, the cargo-auditable crate and the crates under MPL-2.0. The crates are on [crates.io](https://crates.io/) at the versions in that commit's `Cargo.lock`, and `buildinfo` lists those the binary records.
- The crates under MPL-2.0, option-ext, version-ranges and psl: their `.crate` files from crates.io, checked against `Cargo.lock`, are in `/usr/local/share/randomcontainers/tombi/crate-sources/`.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
