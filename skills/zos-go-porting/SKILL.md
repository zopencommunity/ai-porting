---
name: zos-go-porting
description: Load before porting any Go project to z/OS via zopen. Contains required buildenv patterns, Go workspace setup, patch workflow, z/OS-specific Go fixes, and binary encoding requirements. Covers pure-Go and CGO-enabled ports.
---

# z/OS Go Porting

Use this skill for porting Go projects to z/OS using the zopen build system.

## Core Rules

1. Use `.gopatch` extension for all Go patches — zopen's automatic patch applicator skips them, avoiding double-apply. Apply manually via `apply_port_patch` in `zopen_config`.
2. Generate patches **after** the build succeeds using `git -C <dir> diff HEAD`. Stage new files first with `git -C <dir> add` before diffing.
3. Prefer `CGO_ENABLED=0` (pure Go). Only enable CGO if the project explicitly requires it.
4. Always set `ZOPEN_CONFIGURE_MINIMAL=1` — the Go build does not run a `./configure` script.
5. Use `ZOPEN_MAKE="skip"` and `ZOPEN_INSTALL="zopen_install"` — the Go build and install are done entirely in shell functions.
6. Set `GOMAXPROCS=1` in `zopen_init` for build-time stability on z/OS.
7. Use `PORT_ROOT="$(dirname "${GOPATH}")"` to navigate safely — GOPATH lives one level below the port root, and the CWD may be deleted by zopen during clean cycles.
8. Never run `zopen-build` in the background — run in foreground with an appropriate timeout so errors are captured immediately.

## CRITICAL: Continuous Skill Improvement

If the user provides a correction or tip that resolves a z/OS Go porting issue, update this `SKILL.md` in the repository immediately so future agents do not repeat the same mistake.

## Buildenv Structure

A minimal Go `buildenv` looks like this:

```bash
# bump: <name>-version /<NAME>_VERSION="(.*)"/ https://github.com/org/project.git|semver:*
<NAME>_VERSION="x.y.z"

export ZOPEN_NAME="<name>"
export ZOPEN_PROJECT_NAME="<name>"
export ZOPEN_BUILD_LINE="STABLE"
export ZOPEN_STABLE_URL="https://github.com/org/project.git"
export ZOPEN_STABLE_TAG="v${<NAME>_VERSION}"
export ZOPEN_STABLE_DEPS="make coreutils git check_go"

export ZOPEN_CATEGORIES="utilities"
export ZOPEN_RUNTIME_DEPS=""

export ZOPEN_COMP="GO"
export ZOPEN_CONFIGURE="zopen_config"
export ZOPEN_CONFIGURE_MINIMAL=1
export ZOPEN_MAKE="skip"
export ZOPEN_INSTALL="zopen_install"
export ZOPEN_CHECK="skip"
export ZOPEN_CLEAN="zopen_clean"

zopen_init()
{
  export CGO_ENABLED=0
  export GOMAXPROCS=1
  export GOPATH="$(cd .. && pwd)/go-path"
  export GOMODCACHE="${GOPATH}/pkg/mod"
  export GOTMPDIR="/tmp/<name>port-gotmp-${USER:-unknown}"
  mkdir -p "${GOTMPDIR}" "${GOMODCACHE}"

  if [ -n "${GOROOT:-}" ] && [ -d "${GOROOT}/bin" ]; then
    export PATH="${PATH}:${GOROOT}/bin"
  fi

  export GOBIN="${ZOPEN_INSTALL_DIR}/bin"
  mkdir -p "${GOBIN}"
}

zopen_config()
{
  PORT_ROOT="$(dirname "${GOPATH}")"
  cd "${PORT_ROOT}/<name>"

  go mod download
}

zopen_install()
{
  PORT_ROOT="$(dirname "${GOPATH}")"
  cd "${PORT_ROOT}"

  go build -buildvcs=false \
    -ldflags="-s -w -X main.version=v${<NAME>_VERSION}" \
    -o "${GOBIN}/<name>" \
    ./<name>/cmd/<name>

  # Tag the binary as ASCII so z/OS AUTOCVT does not double-convert UTF-8 output
  chtag -tc ISO8859-1 "${GOBIN}/<name>" 2>/dev/null || true
}

zopen_check_results()
{
  echo "actualFailures:0"
  echo "totalTests:1"
  echo "expectedFailures:0"
  echo "expectedTotalTests:1"
}

zopen_get_version()
{
  echo "${<NAME>_VERSION}"
}

zopen_clean()
{
  PORT_ROOT="$(dirname "${GOPATH}")"
  cd "${PORT_ROOT}"
  chmod -R u+w "${PORT_ROOT}/go-path" 2>/dev/null || true
  rm -rf \
    "${PORT_ROOT}/go.work" \
    "${PORT_ROOT}/go.work.sum" \
    "${PORT_ROOT}/go-path" \
    "${PORT_ROOT}/<name>"
}
```

## Workflow

### 1. Collect Metadata

Same as the C/C++ skill. Additionally determine:
- Go version required (check `go.mod`)
- Whether the project uses CGO
- Whether the project has dependencies that themselves need z/OS patches (needing a go.work workspace)
- The exact import path for the main binary (`go.mod` module name + `cmd/<name>` path)

### 2. Map Dependencies

- Use `check_go` (not `go`) in `ZOPEN_STABLE_DEPS`
- Always include `git` — `go mod download` needs it for module fetching
- Include `make` and `coreutils` for build utilities
- For projects with C extensions: add `make` and the relevant C library

### 3. Patch Workflow

Go patches are applied manually. Use `.gopatch` extension to prevent zopen's automatic applicator from touching them.

#### Setup local clone for patch development

```bash
# Clone the upstream project at the exact tag
git clone https://github.com/org/project.git /tmp/<name>
git -C /tmp/<name> checkout v<version>

# Clone a dependency if it also needs patching
git clone https://github.com/org/dep.git /tmp/deps/<dep>
git -C /tmp/deps/<dep> checkout v<dep-version>
```

#### Apply changes directly in the clone

Add new z/OS-specific files and edit existing files directly in `/tmp/<name>`. Then:

```bash
# Stage new files (git diff only shows tracked changes)
git -C /tmp/<name> add pkg/foo/bar_zos.go

# Generate the patch
git -C /tmp/<name> diff HEAD > /path/to/port/patches/<name>-zos.gopatch

# For a dependency patch
git -C /tmp/deps/<dep> add y/file_zos.go
git -C /tmp/deps/<dep> diff HEAD > /path/to/port/patches/<name>-<dep>-zos.gopatch
```

#### Apply patches in buildenv

```bash
apply_port_patch()
{
  dir="$1"
  patch="$2"

  if ! git -C "${dir}" apply --check "../patches/${patch}"; then
    echo "Patch ${patch} did not apply cleanly; checking if already applied..."
    git -C "${dir}" apply --reverse --check "../patches/${patch}" >/dev/null 2>&1 || return 1
    return 0
  fi
  git -C "${dir}" apply "../patches/${patch}"
}
```

Call it in `zopen_config`:

```bash
apply_port_patch <name> <name>-zos.gopatch || return 1
apply_port_patch deps/<dep> <name>-<dep>-zos.gopatch || return 1
```

### 4. Go Workspace (go.work) for Multi-Module Patches

When a dependency also needs z/OS patches, use a go.work workspace so both modules are built from local clones rather than the module cache.

#### In zopen_config

```bash
clone_module()
{
  repo="$1"; dir="$2"; ref="$3"
  if [ ! -d "${dir}/.git" ]; then git clone "${repo}" "${dir}"; fi
  git -C "${dir}" fetch --tags --force
  git -C "${dir}" checkout --force "${ref}"
  git -C "${dir}" clean -fdx
}

zopen_config()
{
  PORT_ROOT="$(dirname "${GOPATH}")"
  cd "${PORT_ROOT}"

  clone_module https://github.com/org/project.git     <name>  "v${<NAME>_VERSION}"
  clone_module https://github.com/org/dep.git         <dep>   "v${DEP_VERSION}"

  apply_port_patch <name> <name>-zos.gopatch   || return 1
  apply_port_patch <dep>  <name>-<dep>-zos.gopatch || return 1

  rm -f go.work go.work.sum
  go work init ./<name> ./<dep>

  cd <name>
  go mod download
  cd "${PORT_ROOT}"
}
```

#### In zopen_clean — remove both clones

```bash
zopen_clean()
{
  PORT_ROOT="$(dirname "${GOPATH}")"
  cd "${PORT_ROOT}"
  chmod -R u+w "${PORT_ROOT}/go-path" 2>/dev/null || true
  rm -rf \
    "${PORT_ROOT}/go.work" "${PORT_ROOT}/go.work.sum" \
    "${PORT_ROOT}/go-path" \
    "${PORT_ROOT}/<name>" \
    "${PORT_ROOT}/<dep>"
}
```

### 5. Build and Iterate

```bash
cd <name>port
zopen-build -f -v
```

Inspect `log.STABLE` on failure. Common Go build commands to test locally on z/OS:

```bash
# From the port root directory (where go.work lives)
go build -buildvcs=false -o /tmp/test-binary ./cmd/<name>
go vet ./...
```

### 6. Finalize

1. Regenerate patches from the local clones **after** the build succeeds.
2. Run `bump check buildenv` to validate the version bump config.
3. Add `.gitignore` entries for any cloned source directories:
   ```
   <name>/
   <dep>/
   go-path/
   go.work
   go.work.sum
   ```
4. Update `patches/README.md`.

## z/OS-Specific Go Fixes

### Missing platform files (most common)

Go projects provide `_linux.go`, `_darwin.go`, etc. but no `_zos.go`. Create one:

```go
//go:build zos

package foo

// implement the platform-specific interface using z/OS syscalls
```

Build tag syntax — z/OS uses `//go:build zos` (GOOS=zos, GOARCH=s390x).

### syscall.Stat_t field differences

z/OS `syscall.Stat_t` differs from Linux:
- Modification time: `stat.Mtim.Sec` / `stat.Mtim.Nsec` (not `stat.Mtimensec`)
- Block size unit: `stat.Blocks * 512` (not `stat.Blksize`)
- Hard-link count for multi-link inode dedup: `stat.Nlink`

```go
//go:build zos
func getPlatformUsage(info os.FileInfo) int64 {
    if stat, ok := info.Sys().(*syscall.Stat_t); ok {
        return stat.Blocks * 512
    }
    return 0
}
```

### unix.O_DSYNC missing

z/OS lacks `unix.O_DSYNC`. Use `syscall.O_SYNC` instead.

Pattern — in the dependency's `file_dsync.go`:
```go
//go:build ... && !zos
```
In `file_nodsync.go`:
```go
//go:build ... || zos
// use syscall.O_SYNC
```

### SQLite unavailable on z/OS/s390x

Add `|| zos` to the build tag of the `sqlite_other.go` stub file that returns "not available":

```go
//go:build ... || zos

package analyze

func GetSQLiteConn(...) error { return errors.New("SQLite not available on this platform") }
```

### Mount point / disk info

z/OS exposes mount points via `/proc/mounts` (same format as Linux). Use `unix.Statfs` for disk sizes. Provide a `_zos.go` implementation using these.

### ASCII terminal default (no-unicode)

z/OS terminals use ASCII, not UTF-8. For tools that have a `--no-unicode` or `--ascii` flag, default it to `true` on z/OS using build-tag files:

`cmd/<name>/defaults_zos.go`:
```go
//go:build zos

package main

const noUnicodeDefault = true
```

`cmd/<name>/defaults_other.go`:
```go
//go:build !zos

package main

const noUnicodeDefault = false
```

Then in `main.go`:
```go
flags.BoolVarP(&af.NoUnicode, "no-unicode", "u", noUnicodeDefault, "...")
```

### Binary output encoding (AUTOCVT double-conversion)

z/OS SSH sessions with `_BPXK_AUTOCVT=ON` auto-convert EBCDIC→UTF-8. Go binaries output UTF-8 natively, so **untagged** binaries get double-converted and produce garbled output.

Fix: tag the binary as ISO8859-1 (ASCII) in `zopen_install`:
```bash
chtag -tc ISO8859-1 "${GOBIN}/<name>" 2>/dev/null || true
```

Without this, the binary will appear to produce garbled output when run over SSH unless the user has sourced a login shell (`bash -l`) that sets the correct locale.

### GOTMPDIR required

z/OS Go builds require a writable temp directory. Always set:
```bash
export GOTMPDIR="/tmp/<name>port-gotmp-${USER:-unknown}"
mkdir -p "${GOTMPDIR}"
```

If omitted, builds may fail with permission or path errors.

### go build -buildvcs=false

Always pass `-buildvcs=false`. Without it, Go tries to embed VCS info from the go.work workspace, which may fail because the workspace root is not a git repo.

### Dependency version in ldflags

Find the correct ldflags import path for version injection:
```bash
grep -r 'Version\|version' <name>/build/ <name>/internal/version/ 2>/dev/null | head -10
```

Common patterns:
- `-X github.com/org/project/build.Version=v${VERSION}`
- `-X github.com/org/project/cmd/<name>/version.Version=v${VERSION}`
- `-X main.version=v${VERSION}`

### CGO-enabled ports

When CGO is required:
```bash
export CGO_ENABLED=1
export CC="${CC:-ibm-clang}"
export CGO_CFLAGS="-I${SOME_DEP_HOME}/include"
export CGO_LDFLAGS="-L${SOME_DEP_HOME}/lib -lsomething"
```

Add `make coreutils` and any required C library zopen packages to `ZOPEN_STABLE_DEPS`.

## Completion Criteria

Port is complete when:
1. `zopen-build -f -v` succeeds.
2. Dependencies use exact zopen names (`check_go` not `go`).
3. Patches are generated after a successful build.
4. `.gopatch` extension used for all patches.
5. `bump check buildenv` passes (use git URL pattern, not releases page).
6. `.gitignore` includes cloned source dirs, `go-path/`, `go.work`, `go.work.sum`.
7. Binary is tagged ISO8859-1 (or `chtag` call is in `zopen_install`).
8. `patches/README.md` describes each patch and why it was needed.
