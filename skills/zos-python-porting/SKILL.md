---
name: zos-python-porting
description: Load before starting any z/OS python porting work. Contains required procedures, known gotchas, and build patterns for zopen ports. It provides a command-first workflow for metadata collection, dependency mapping to exact zopen package names, project generation, build/fix iteration, patch creation, bump validation, and optional repo/CI setup.
---

# z/OS Porting

Use this skill for end-to-end zopen porting work with local `zopen-*` commands.

## Core Rules

1. Use local CLI commands directly (`zopen-generate`, `zopen-build`, `zopen-info`, etc.).
2. Use `--help` as source of truth for flags/syntax in installed tooling.
3. Prefer Homebrew formula metadata and upstream project metadata first; use web search only as fallback.
4. Do not create files in `patches/` until build succeeds.
5. **Always run `zopen-build` in foreground with appropriate timeout, never as background process.** This ensures proper error capture and debugging.

## CRITICAL: Continuous Skill Improvement

**If the user provides a course correction, tip, or alternative approach that successfully resolves an issue, you MUST update this `SKILL.md` file in the `https://github.com/IgorTodorovskiIBM/zos-porting` repository.** This ensures the skill evolves and prevents future agents from repeating the same mistakes. Treat this as a core mandate of the porting workflow.

## Preflight

Run:

```bash
command -v zopen-generate zopen-build zopen-info zopen-query zopen-version jq git bump
zopen-generate --help
zopen-build --help
zopen-info --help
zopen-version || zopen-version --help
```

If a command reports `Source the zopen-config prior to running ...`, source zopen config first.

## MCP Integration

**For porting issues, leverage the `zos-porting-rag` MCP server** which provides access to 600+ z/OS porting patches and documentation. 

**Setup**: See [zos-porting-rag repository](https://github.com/ZOSOpenTools/zos-porting-rag) for MCP server configuration.

## Workflow

### 1. Collect Metadata

Gather:
- project name (lowercase)
- one-line description
- repo URL (prefer the Git clone repository location over an archive file)
- SPDX license
- categories
- build system
- stable URL + dependencies

Use:
- `zopen-generate --list-licenses`
- `zopen-generate --list-categories`
- `zopen-generate --list-build-systems`

## Python Porting

### Metadata Sources (in order)

1. `pyproject.toml`
2. `setup.py` or `setup.cfg`
3. PyPI metadata
4. upstream docs / README
5. Homebrew formula metadata if available - `https://formulae.brew.sh/api/formula/${PROJECT}.json`

Rules:
- Prefer upstream Python packaging metadata over Brew.
- Use `check_python` for Python dependency mapping, not `python`.
- Distinguish build requirements from runtime requirements.
- If the package uses `pyproject.toml`, inspect the build backend first
  (e.g. setuptools, hatchling, poetry-core, meson-python, maturin).
- `python -m build --wheel` handles all project types (pyproject.toml, setup.py, setup.cfg) — no need for separate code paths.

### C Extensions Detection

Check for `.c` files in the project source tree. If present:
- Add `--c-extensions` flag to `zopen-generate` — this omits `ZOPEN_COMP="skip"` so the C compiler is available during the build.
- The compiler variables (CC, CFLAGS, LDFLAGS) are exported as environment variables by `zopen-build` and Python's build system (setuptools, etc.) picks them up automatically.
- `ZOPEN_MAKE_MINIMAL="yes"` is set for all Python ports — this keeps compiler flags in the environment rather than passing them as make arguments (which would break Python builds).
- **C extensions compile successfully on z/OS with `-fvisibility=default` flag in `ZOPEN_EXTRA_CFLAGS`.** Modern Python packages often don't need additional symbol visibility patches - the compiler flag is usually sufficient for all extensions to work correctly.
- **CRITICAL for C extension linking**: Must append `LIBS` to `LDFLAGS` in `zopen_init()` function for proper linking. Example: `export LDFLAGS="$LDFLAGS $LIBS"`. This ensures zoslib and other dependency libraries are linked correctly during Python wheel build.

If no C extensions:
- `ZOPEN_COMP="skip"` is set (default for Python ports without `--c-extensions`).

### Git Submodules

**For Python ports with git submodules** (e.g., bundled C libraries): Set `ZOPEN_CLONE_SUBMODULES="yes"` in `buildenv` to ensure submodules are cloned during build. Example: python-xxhash bundles xxHash C library as submodule in `deps/xxhash/`.

### Generate Command

Pure Python:
```bash
zopen-generate \
  --name <name> \
  --description "<description>" \
  --categories "<cat1 cat2>" \
  --license <spdx_or_unknown> \
  --type BUILD \
  --build-system Python \
  --stable-url "<Git clone url or Archive url>" \
  --stable-deps "check_python <other_deps>" \
  --dev-url "<Git clone url or Archive url>" \
  --dev-deps "check_python <other_deps>" \
  --build-line stable \
  --runtime-deps "check_python" \
  --non-interactive
```

Python with C extensions:
```bash
zopen-generate \
  --name <name> \
  --description "<description>" \
  --categories "<cat1 cat2>" \
  --license <spdx_or_unknown> \
  --type BUILD \
  --build-system Python \
  --c-extensions \
  --stable-url "<Git clone url or Archive url>" \
  --stable-deps "check_python <other_deps>" \
  --dev-url "<Git clone url or Archive url>" \
  --dev-deps "check_python <other_deps>" \
  --build-line stable \
  --runtime-deps "check_python" \
  --non-interactive
```

### Build Workflow

The generated `buildenv` sets `ZOPEN_BUILD_SYSTEM="Python"`. All build logic is handled **natively by `zopen-build`** — no custom functions needed in the buildenv.

`zopen-build` automatically:
- Creates a venv, installs `setuptools build installer wheel`
- Runs `python -m build --wheel`
- Runs `pytest -v` for testing (installs wheel with `--force-reinstall` first)
- Installs the wheel into `$ZOPEN_INSTALL_DIR/lib/python` and symlinks scripts into `bin/`
- Preserves the `.whl` file in `$ZOPEN_INSTALL_DIR/dist/` for publishing
- Sets up `PYTHONPATH` in `.env`
- Merges `LIBS` into `LDFLAGS` for C extension ports (Python's build system ignores `LIBS`)
- Parses pytest output for `zopen_check_results`
- Adds `check_python` and `grep` as implicit dependencies

### Overriding Defaults

If a project needs custom behavior (e.g., a different test runner), define the function in `buildenv` and it takes precedence over the built-in. For example, pycryptodome uses its own test suite:

```sh
zopen_custom_check() {
  . .venv/bin/activate
  pip install --force-reinstall --no-deps dist/*.whl
  pip install pycryptodome-test-vectors==1.0.22
  python -m Crypto.SelfTest
  zopen_check_result=$?
  deactivate
  return "${zopen_check_result}"
}

zopen_check_results() {
  dir="$1"; pfx="$2"; chk="$1/$2_check.log"
  totalTests=$(grep -oE "Ran [0-9]+ tests" "${chk}" | awk '{print $2}')
  totalTests=${totalTests:-0}
  if grep -q "^OK$" "${chk}"; then
    actualFailures=0
  else
    actualFailures=$(grep -oE "failures=[0-9]+" "${chk}" | cut -d= -f2)
    actualFailures=${actualFailures:-1}
  fi
  echo "actualFailures:${actualFailures}"
  echo "totalTests:${totalTests}"
  echo "expectedFailures:0"
}
```

Also set `ZOPEN_MAKE="zopen_custom_check"` and `ZOPEN_CHECK="zopen_custom_check"` in buildenv to use custom functions instead of built-ins.

**CRITICAL: When customizing Python test execution:**
- Keep pytest invocation verbose (`-v`) to preserve full check log output
- Ensure `zopen_check_results` parses summary text patterns like `N passed`, `N failed`, and `N errors`
- Stable log formatting makes CI diagnosis and result extraction reliable
- Always provide a matching `zopen_check_results` parser for custom check flows so zopen-build records accurate totals

### Publishing

#### GitHub (pax)
Standard `zopen-publish` for pax files (same as any port):
```bash
zopen-publish -f \
  -p install/<name>.pax.Z \
  -m metadata.json \
  -g <TAG> \
  -t <github_token>
```

#### Pulp PyPI (wheel)
Publish the preserved wheel from `$ZOPEN_INSTALL_DIR/dist/` to a Pulp PyPI repository:
```bash
zopen-publish \
  --whl <name>port/install/<name>/dist/<name>-<version>-*.whl \
  --pulp-url http://<host>:<port>/pypi/<repo>/ \
  --pulp-password <password>
```

Environment variables `PULP_URL`, `PULP_USER`, `PULP_PASSWORD` can be used instead of flags.

#### Both at once
```bash
zopen-publish -f \
  -p install/<name>.pax.Z \
  -m metadata.json \
  -g <TAG> \
  -t <github_token> \
  --whl install/<name>/dist/*.whl \
  --pulp-url http://<host>:<port>/pypi/<repo>/ \
  --pulp-password <password>
```

Consumers install from Pulp with:
```bash
pip install --index-url http://<host>:<port>/pypi/<repo>/simple/ <package>
```

### Testing Python Packages

**Python packages with built-in test runners (e.g., `python -m Crypto.SelfTest`) often work better than pytest on z/OS.** Always check for native test runners before forcing pytest, especially when encountering test class compatibility issues. If the package provides its own test runner:
1. Check the package documentation or README for test instructions
2. Modify `zopen_custom_check()` in `buildenv` to use the native test runner instead of pytest
3. Update `zopen_check_results()` to parse the native test runner's output format

**Python test import shadowing**: When pytest runs from source directory with local package folder (e.g., `xxhash/`), it shadows the installed wheel in site-packages, causing `ModuleNotFoundError` for C extensions. This is expected Python behavior. Tests pass when run from outside source dir or after proper installation.

**CRITICAL: Handling C Extension Test Failures**

For Python C-extension ports, if pytest in zopen-build imports the source tree instead of the installed wheel and fails with `ModuleNotFoundError` for the extension:

1. **Before changing test logic**, inspect the newest `*_check.log` to confirm whether failures come from test execution or import-path shadowing
2. A pattern like "collected tests" followed by `ModuleNotFoundError` for `package._extension` usually means pytest is running from the source tree and not exercising the installed wheel
3. **Define `zopen_custom_check`** to:
   - Install `dist/*.whl` into `.venv` with `pip install --force-reinstall --no-deps`
   - Copy only the test suite to a temporary directory outside the source tree (e.g., under `/tmp`)
   - Run pytest from that temporary directory so imports resolve to the installed package
4. **Provide matching `zopen_check_results`** parser to extract passed/failed/error counts from the pytest summary line

Example for ports with bundled or generated tests:
```sh
zopen_custom_check() {
  . .venv/bin/activate
  pip install --force-reinstall --no-deps dist/*.whl
  
  # Copy tests to temporary directory outside source tree
  test_dir="/tmp/${ZOPEN_PKGNAME}_tests_$$"
  mkdir -p "${test_dir}"
  cp -r tests/* "${test_dir}/"
  
  cd "${test_dir}"
  pytest -v
  zopen_check_result=$?
  
  cd -
  rm -rf "${test_dir}"
  deactivate
  return "${zopen_check_result}"
}
```

This approach:
- Validates the packaged extension exactly as CI/install will use it
- Ensures the check phase explicitly stages tests into a temporary execution directory
- Confirms the built wheel contains the extension module
- Runs tests against the installed wheel from outside the source tree
- Validates extracted test content separately from the source checkout

**A successful local import test alone is not enough to validate CI**; always confirm the package passes through the zopen-build check phase with the installed wheel plus extracted/copied tests, because pytest collection context can differ from ad hoc manual testing.

### 2. Map Dependencies (Strict)

Dependency source of truth:
- `https://raw.githubusercontent.com/zopencommunity/meta/refs/heads/main/docs/api/zopen_releases_latest.json`

Rules:
1. Map each required brew dependency to exact zopen package name.
2. Keep required dependencies only.
3. If required dependency is unavailable in zopen package list, fail and explain.
4. **Always add `coreutils`** if the project uses `make install` and the system `install` is insufficient.

Special cases:
- use `check_python` (not `python`)
- use `check_go` (not `go`)
- if `flex` is required, add `m4` before `flex`
- if `cmake` is required, add `make`
- if configure fails with "requires GNU bison", add `bison` to `ZOPEN_STABLE_DEPS`
- if build fails with "envsubst: FSUM7351 not found", add `gettext` to `ZOPEN_STABLE_DEPS` (envsubst is provided by gettext and is commonly used in Makefiles for template variable substitution)

### 3. Generate Project

Before generating, ensure `<name>port` does not exist (or use `--force` intentionally).

Non-interactive template:

```bash
zopen-generate \
  --name <name> \
  --description "<description>" \
  --categories "<cat1 cat2>" \
  --license <spdx_or_unknown> \
  --type BUILD \
  --build-system "<GNU Make|CMake|Go|Gradle|Maven|Meson|Python>" \
  --stable-url "<url>" \
  --stable-deps "<dep1 dep2>" \
  --build-line stable \
  --dev-deps "<dep1 dep2>" \
  --non-interactive
```

Notes:
- Use `--build-system Go` for Go projects.
- Keep upstream source URLs (`--stable-url`, `--dev-url`) as `https://` URLs.
- **CRITICAL: Sanitize `buildenv` variables**: Shell variables CANNOT contain hyphens. Always use underscores (e.g., `PYTHON_XXHASH_VERSION`, not `PYTHON-XXHASH_VERSION`). Hyphenated names break shell parsing and can surface later as invalid version or loadBuildEnv failures during install/finalization.
- **For git-based ports**: Specify the HTTPS git URL (e.g., `https://github.com/org/project.git`) rather than a tarball URL unless absolutely necessary. This is appropriate for projects that require git history or submodules.
- **For CMake projects**: Always reference existing working examples like `github.com/zopencommunity/llamacppport/blob/main/buildenv` before starting.
- **Dependency Home Variables**: `zopen-build` automatically provides the root directory of each dependency as an environment variable named `<DEPNAME>_HOME` (e.g., `BLIS_HOME`, `ZOSLIB_HOME`). Reference these in `buildenv` as `\${DEPNAME_HOME}` to ensure they are evaluated correctly during the build process.

### 4. Build and Iterate

```bash
cd <name>port
zopen-build -v
```

**CRITICAL: Always run `zopen-build` in foreground with appropriate timeout (e.g., 300s for typical builds, longer for complex projects). Never use background processes.** This ensures proper error capture and allows immediate debugging of build failures.

If build fails:
1. inspect latest `log.STABLE`/`log.DEV`
2. identify root cause
3. modify source or `buildenv`
4. rerun `zopen-build -v`

#### Handling Patch Conflicts

When patches fail to apply cleanly (common with line ending or format issues on z/OS):

**Best Practice: Individual File Resolution**
1. **Fix conflict locally**: Manually resolve the conflict in the extracted source file. Ensure the fix is correct for z/OS.
2. **Create replacement patch**:
   ```bash
   cd <extracted-source-dir>
   git add <file>
   git diff HEAD -- <file> > ../patches/<file>.patch
   ```
3. **Reset and Reapply**:
   ```bash
   git reset --hard
   cd ..
   zopen-build -v
   ```
   This ensures that the build starts from a clean state with your corrected patch correctly applied by `zopen-build`.

**Alternative: Force Apply**
1. **Force apply patches to generate rejection files**:
```bash
zopen-build -v --forcepatchapply
```
This applies patches where possible and creates `.rej` files for rejected hunks.

2. **Manually resolve conflicts**:
   - Locate `.rej` files in the extracted source directory
   - Apply rejected changes manually to the corresponding source files
   - Use the `.rej` file content as a guide for what needs to be changed

3. **Create corrected patches**:
```bash
cd <extracted-source-dir>
git diff HEAD > ../patches/PR1.patch
```

4. **Clean and rebuild**:
```bash
cd ..
zopen-build -v --clean
zopen-build -v
```

Common fixes:
- missing configure: set `ZOPEN_BOOTSTRAP` or `ZOPEN_CONFIGURE="skip"`
- missing macros/functions: add `-D__XPLAT` in `ZOPEN_EXTRA_CPPFLAGS`, rebuild with `-f` if needed
- platform differences: guard with `#ifdef __MVS__`
- **Missing symbols in `.so`**: Add `-fvisibility=default` to `ZOPEN_EXTRA_CFLAGS` or patch headers with `__attribute__((visibility("default")))`.
- **Read-only `/usr/local` errors**: Use `ZOPEN_EXTRA_MAKE_OPTS` to override install paths (e.g., `export ZOPEN_EXTRA_MAKE_OPTS="INSTALL_LIB_DIR=\${ZOPEN_INSTALL_DIR}/lib"`).
- **Big Endian issues**: Check for bit-packing or binary format assumptions. Disable `mmap` if byte-swapping is needed in memory.
- **u_int*_t typedef conflicts**: Add `#ifndef __MVS__` guards around `u_int8_t`, `u_int16_t`, `u_int64_t` typedefs (z/OS uses standard `uint*_t` types).
- **Missing symbols in `.so` (CRITICAL for extensions)**: 
  1. Add `-fvisibility=default` to `ZOPEN_EXTRA_CFLAGS` in `buildenv`
  2. Patch header files: change `#define API_MACRO` to `#define API_MACRO __attribute__((visibility("default")))` for non-Windows platforms
  3. For template headers (`.h.tmpl`), add platform check: `#ifndef _WIN32 ... #else ... __attribute__((visibility("default"))) ... #endif`
- **Read-only `/usr/local` errors**: 
  1. Use `ZOPEN_EXTRA_MAKE_OPTS` to override install paths: `export ZOPEN_EXTRA_MAKE_OPTS="INSTALL_LIB_DIR=\${ZOPEN_INSTALL_DIR}/lib INSTALL_INCLUDE_DIR=\${ZOPEN_INSTALL_DIR}/include"`
  2. Set `ZOPEN_INSTALL_OPTS="install \${ZOPEN_EXTRA_MAKE_OPTS}"` to ensure overrides are passed to `make install`
  3. Patch Makefile to use `?=` instead of `=` for install directory variables (e.g., `INSTALL_LIB_DIR ?= /usr/local/lib`)
- **Big Endian issues**: Check for bit-packing or binary format assumptions. Disable `mmap` if byte-swapping is needed in memory.
- **posix_memalign missing declaration**: Ensure `#define _XOPEN_SOURCE 600` is at the VERY TOP of the C file, before any includes.
- **thread_local support**: z/OS Clang may not support `thread_local`. Use thread-specific storage or remove if safe.
- **poll() conflicts**: `#define __poll 1` in `poll.h` can conflict with variables named `__poll`. `#undef __poll` after including `<poll.h>` on z/OS.
- Some Python packages (e.g., psutil) check `sys.platform` but lack z/OS support. Patch the pyton script to add z/OS handling similar to existing platforms like AIX. Add `elif sys.platform.startswith('zos'):` cases with appropriate z/OS configuration.


### 5. Finalize After Success

1. Create patch from extracted source tree:
```bash
git diff HEAD > ../patches/PR1.patch
```
2. Verify package/binary:
```bash
zopen-info <name>
```
3. If exporting headers/libs for downstream ports, update `zopen_append_to_env` in `buildenv`.
4. Validate bump config:
```bash
bump --help
bump current buildenv
bump check buildenv
```
**CRITICAL for bump configuration**:
- Ensure version variable (e.g., `USEARCH_VERSION`) is defined BEFORE `ZOPEN_STABLE_URL` references it
- **Use git repository URL pattern** for GitHub projects: `https://github.com/org/project.git|*` (NOT the releases page URL `https://github.com/org/project/releases|semver:*`, which fails with "no version found" error)
- Follow existing port examples like `gitport` for correct bump pattern syntax

5. Add source-dir ignore pattern to `.gitignore`:
```bash
echo "" >> .gitignore
echo "# Ignore source directories created by zopen-build" >> .gitignore
echo "<package-name>-*/" >> .gitignore
echo "<package-name>/" >> .gitignore
```
6. **NEVER check in the extracted source directory.** Verify with `git status` before committing.
7. Document changes in `patches/README.md`.

## Optional Repo/CI

Only for users with required org permissions.

1. Ask user whether to create repo now:
```bash
zopen-create-repo --help
zopen-create-repo -n <name> -d "zopen port of <name>"
```
Fallback for token issues: `unset GITHUB_TOKEN; gh repo create zopencommunity/<name>port --public --description "..."`.

2. Push using SSH remote:
```bash
git remote add origin git@github.com:zopencommunity/<name>port.git
git push origin main
```

3. Ask user whether to create CI job now:
```bash
zopen-create-cicd-job --help
zopen-create-cicd-job -n <name> -b stable -s cicd-stable.groovy -r yes
```

## Completion Criteria

Port is complete when:
1. `zopen-build` succeeds.
2. dependencies are exact zopen names.
3. patches are generated after success.
4. bump checks pass.
5. `.gitignore` includes source-dir pattern.
6. `patches/README.md` is updated.