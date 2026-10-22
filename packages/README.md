Central declarative congfigurations for artifacts delivering.
===

> We use go template format to control them.
> Ref for SemVer Constraint: https://github.com/Masterminds/semver#checking-version-constraints

## Prerequire tools

- [gomplate](https://github.com/hairyhenderson/gomplate)
- [yq](https://github.com/mikefarah/yq)
- [jq](https://jqlang.github.io/jq/download/)

## Profiles

- `release`: community release profile.
- `enterprise`: enterprise release profile, it will not publish any tiup pkgs.
- `failpoint`: enable failpoint switch on community profile.
- `fips`: fips feature release without enterprise plugins.
- `rust`: builds only the standalone Rust SQL node (`tidb-server`) for
  `pingcap/tidb@hparser-integration` on `linux/amd64` and `linux/arm64`, and publishes the
  `tidb` tiup package to the **staging** mirror only. Uses the official Rust image
  (`docker.io/rustlang/rust:nightly-2026-08-22`, glibc) and builds no container images.

## For component binaries packages and container images

Configuration template: [packages.yaml.tmpl](./packages.yaml.tmpl)

### What's the golang version the builders are using

- `(~ 6.0]`: golang `v1.18.x`
- `[6.1 ~ 7.0)`: golang `v1.19`
- `[7.0 ~ 7.3]`: golang `v1.20.x`
- `[7.4 ~ 8.3]`: golang `v1.21.x`
- `[8.4 ~ 8.5.4]`: golang `v1.23.x`
- `[8.5.5 ~ )`: golang `v1.25.x`

### What's the minimum OS of the packages

The OS floor of a package is **not** declared per component — it is inherited from the builder image that
produced it and, for darwin, from the SDK of the macOS build machine. Use this table to answer
"can my machine run this package".

| Platform | Minimum OS requirement | Determined by |
|---|---|---|
| `linux/amd64`, `linux/arm64` | depends on the builder image: **glibc 2.28** for the Rocky Linux 8.10 builders (most components), **glibc 2.17** for the `centos7` builders; components built on other base images (e.g. `tidb-operator` uses the official `golang` image) can be higher | `dockerfiles/cd/builders/<component>/Dockerfile` vs `.../centos7/Dockerfile`; routing in [packages.yaml.tmpl](./packages.yaml.tmpl) |
| `darwin/amd64`, `darwin/arm64` | **macOS 13.0** | SDK of the macOS build machine (Command Line Tools 14.3 / SDK 13.3) |

Checking a package:

```console
# darwin: minimum macOS version recorded in the Mach-O load commands
$ otool -l <binary> | grep -A3 LC_BUILD_VERSION      # minos 13.0 => needs macOS 13+

# linux: highest glibc symbol version required by the binary
$ objdump -T <binary> | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -1
```

Change history:

| Date | Change |
|---|---|
| 2026-10-22 | `darwin/amd64` and `darwin/arm64` packages moved from **macOS 12.0** to **macOS 13.0**. The macOS build machines were upgraded from macOS 12.7.6 to 13.7.8 (Command Line Tools 14.3 / SDK 13.3) in preparation for the Go 1.27 toolchain, which requires macOS 13+. Because `LC_BUILD_VERSION.minos` follows the SDK of the build machine, the new floor took effect immediately, before any Go version bump. `linux` packages are unaffected. |

> The floor is a property of the produced binaries, not of this repository, so it can change without any
> change to `go.mod` or `packages.yaml.tmpl`.

### Required context

You can get them by run:
```console
$ grep -oE "{{\s*\..*?}}" packages/packages.yaml.tmpl | grep -oE "\.\w+(\.\w+)*" | sort -u
.Git.ref
.Git.sha
.Git.url
.Release.arch
.Release.os
.Release.version
```

## For offline deploy packages

### Required context

You can get them by run:
```console
$ grep -oE "{{\s*\..*?}}" packages/offline-packages.yaml.tmpl | grep -oE "\.\w+(\.\w+)*" | sort -u
.Release.arch
.Release.version
```

## How to verify the template

Please run `./.github/scripts/ci.sh` locally before commit.
