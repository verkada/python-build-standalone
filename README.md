# Verkada Python Build Standalone

This is Verkada's fork of [astral-sh/python-build-standalone](https://github.com/astral-sh/python-build-standalone), customized to build FIPS-enabled Python distributions for our specific target platforms.

## Supported Targets

We build **3 targets** with FIPS-enabled OpenSSL:

- **aarch64-apple-darwin** (macOS ARM64)
- **aarch64-unknown-linux-gnu** (Linux ARM64)
- **x86_64-unknown-linux-gnu** (Linux x86_64)

**Build variants:**
- `pgo+lto` - Optimized build (base for install_only artifacts)
- `freethreaded+pgo+lto` - Free-threaded builds (Python 3.13+)

**Python versions:** 3.10, 3.11, 3.12, 3.13, 3.14

---

## 0. How to Release

### Prerequisites

1. **Sync from upstream** (if needed):
   ```bash
   git fetch upstream
   git checkout main
   git merge upstream/main
   git push origin main
   ```

2. **Ensure all workflows succeed on main:**
   - Check that **vlinux** workflow completed successfully
   - Check that **vmacos** workflow completed successfully
   - Verify all target/Python version combinations built

### Triggering a Release

#### Option 1: GitHub UI

1. Go to [Actions > vRelease](https://github.com/verkada/python-build-standalone/actions/workflows/vrelease.yml)
2. Click "Run workflow"
3. Fill in:
   - **tag**: Release version (e.g., `20260105`)
   - **sha**: Full commit SHA from main branch where builds succeeded
   - **dry-run**: Check to test without publishing
4. Click "Run workflow"

#### Option 2: GitHub CLI

```bash
# Get the commit SHA from main
SHA=$(git rev-parse main)

# Dry run first (recommended)
gh workflow run vrelease.yml \
  --repo verkada/python-build-standalone \
  --ref main \
  -f tag=20260105 \
  -f sha=$SHA \
  -f dry-run=true

# Wait for dry run to complete and verify
gh run watch

# Run actual release
gh workflow run vrelease.yml \
  --repo verkada/python-build-standalone \
  --ref main \
  -f tag=20260105 \
  -f sha=$SHA \
  -f dry-run=false
```

### What the Release Does

1. Downloads all build artifacts from the commit's workflow runs
2. Creates `install_only.tar.gz` and `install_only_stripped.tar.gz` archives
3. Creates a GitHub release with the tag
4. Uploads all artifacts (full `.tar.zst` and install_only `.tar.gz` files)
5. Generates build provenance attestations

### Release Artifacts

Each Python version produces:
```
cpython-{version}-{target}-pgo+lto-{date}.tar.zst
cpython-{version}-{target}-install_only-{date}.tar.gz
cpython-{version}-{target}-install_only_stripped-{date}.tar.gz
```

Plus freethreaded variants for Python 3.13+.

---

## 1. FIPS Setup

### Overview

We build OpenSSL with FIPS 140-2 module support to meet compliance requirements. The FIPS module is included in all builds except musl-based targets.

### Implementation

**Files modified:**
- `cpython-unix/build-openssl-3.5.sh` - Enable FIPS during OpenSSL build
- `cpython-unix/build-cpython.sh` - Copy FIPS modules to Python installation
- `src/validation.rs` - Skip FIPS module files in distribution validation

**What gets built:**
1. OpenSSL configured with `enable-fips` flag
2. FIPS provider module (`fips.so` on Linux, `fips.dylib` on macOS)
3. FIPS configuration file (`fipsmodule.cnf`)

**Where FIPS files are located in the distribution:**
```
python/
  share/
    ssl/
      ossl-modules/
        fips.so          # FIPS provider module
      fipsmodule.cnf     # FIPS configuration
```

### Enabling FIPS Mode at Runtime

To use FIPS mode with these distributions:

1. Set the OpenSSL configuration to load the FIPS provider:
   ```bash
   export OPENSSL_CONF=/path/to/openssl-fips.cnf
   export OPENSSL_MODULES=/path/to/python/share/ssl/ossl-modules
   ```

2. Create an `openssl-fips.cnf` file:
   ```ini
   [openssl_init]
   providers = provider_sect

   [provider_sect]
   fips = fips_sect

   [fips_sect]
   activate = 1
   module = /path/to/python/share/ssl/ossl-modules/fips.so
   ```

3. Verify FIPS mode is active:
   ```python
   import ssl
   import hashlib

   # Should show FIPS provider loaded
   print(ssl.OPENSSL_VERSION)
   ```

### FIPS Limitations

#### 1. Symbol Visibility Trade-off

**Issue:** The FIPS provider module requires the `OSSL_provider_init` symbol to be exported. To achieve this, we remove `-fvisibility=hidden` from OpenSSL compilation flags.

**Impact:**
- Increases OpenSSL's exported symbol surface
- Internal symbols that should be hidden become visible
- Not a security vulnerability, but reduces defense in depth

**Code Review Note:** This is an acceptable trade-off for FIPS compliance, but be aware that it differs from standard OpenSSL builds.

#### 2. musl libc Not Supported

FIPS module is **only available on glibc-based Linux** and **macOS**. Not available for musl-based targets because:
- OpenSSL FIPS requires async support (`no-async` flag needed for musl)
- musl lacks atomic primitives required by FIPS module
- FIPS module fundamentally incompatible with musl

#### 3. FIPS Compliance vs FIPS-Enabled

**Important:** Building with `enable-fips` creates a FIPS-capable distribution, but does **not** guarantee FIPS 140-2/140-3 compliance. Full compliance requires:
- Formal validation by NIST/CMVP
- Specific runtime configuration
- Approved cryptographic operations only
- Security policy adherence

Consult your security team for FIPS compliance requirements.

---

## 2. Workflows

### vlinux.yml - Linux Builds

**Triggers:** Push to main, Pull requests

**Runners:**
- **x86_64**: `namespace-profile-ubuntu-22-04-amd64-x86-64-large-caching`
- **aarch64**: `namespace-profile-ubuntu-22-04-amd64-arm-large-caching` (native ARM64)

**Jobs:**
1. **crate-build** - Builds the pythonbuild Rust tool on both architectures
2. **image** - Builds Docker images for build environments:
   - `build`, `build.cross`, `gcc` (x86_64)
   - `build.debian9`, `gcc.debian9` (aarch64)
3. **build** - Builds Python distributions for both targets

**Key Features:**
- Native ARM64 builds (no cross-compilation)
- Docker layer caching via GitHub Container Registry
- Download caching for build dependencies
- Build provenance attestations on main branch
- Distribution validation with runtime tests

**Build Matrix:**
- 2 targets × 5 Python versions × 2 build options = 20 builds
- Total with freethreaded: 24 builds

### vmacos.yml - macOS Builds

**Triggers:** Push to main, Pull requests

**Runner:** `namespace-profile-mac-small-tahoe` (native ARM64)

**Jobs:**
1. **crate-build** - Builds the pythonbuild Rust tool
2. **build** - Builds Python distributions for macOS

**Key Features:**
- Native aarch64 builds
- macOS SDK validation
- Build provenance attestations
- Distribution validation with runtime tests

**Build Matrix:**
- 1 target × 5 Python versions × 2 build options = 10 builds
- Total with freethreaded: 12 builds

### vrelease.yml - Release Workflow

**Trigger:** Manual (`workflow_dispatch`)

**Runner:** `ubuntu-latest`

**Inputs:**
- **tag**: Release tag (e.g., `20260105`)
- **sha**: Commit SHA to release (must have successful vlinux/vmacos runs)
- **dry-run**: Boolean to test without publishing

**What it does:**
1. Fetches build artifacts from the specified commit's workflow runs
2. Creates `install_only` archives (removes static libs, test modules)
3. Creates `install_only_stripped` archives (also removes debug symbols)
4. Creates a GitHub release with the tag
5. Uploads all artifacts to the release
6. Generates build provenance attestations

**Important:** The release workflow uses the `just` build automation tool and the `pythonbuild` Rust CLI to orchestrate the release process.

---

## 3. Workflow Design Decisions

### Why Separate Workflows?

We created `vlinux.yml`, `vmacos.yml`, and `vrelease.yml` as **separate workflows** instead of modifying the upstream workflows:

**Benefits:**
- Won't conflict with upstream when syncing
- Clear separation of Verkada-specific configs
- Easier to maintain and understand

**Trade-off:**
- Upstream workflows still exist (disabled via `workflow_dispatch` only)
- Duplicate code between upstream and Verkada workflows

### Why Hardcoded Matrices?

Instead of using `ci-matrix.py` with `ci-targets.yaml`, we **hardcode the build matrices** in the workflow files:

**Benefits:**
- No dependency on upstream config files
- Self-contained and easier to understand
- Won't break when upstream changes ci-targets.yaml

**Trade-off:**
- Adding new Python versions requires workflow updates
- Less flexible than dynamic matrix generation

### Why Namespace Runners?

We use Verkada's namespace runners instead of GitHub-hosted runners:

**Benefits:**
- Native ARM64 builds (faster, more reliable than cross-compilation)
- Larger disk space (avoid "no space left" errors)
- Caching support for better performance
- Cost control within Verkada infrastructure

**Trade-off:**
- Requires namespace runner infrastructure
- Less portable than GitHub-hosted runners

---

## 4. Creating Pull Requests

**CRITICAL:** Always create PRs against `verkada/python-build-standalone`, NOT the upstream repo.

```bash
# Correct - targets our fork
gh pr create --repo verkada/python-build-standalone --title "Title" --body "Body"

# WRONG - would target upstream astral-sh repo
gh pr create --title "Title" --body "Body"
```

Since this is a fork, the default `gh pr create` may target the upstream repository. Always explicitly specify `--repo verkada/python-build-standalone`.

---

## 5. Syncing from Upstream

When syncing from upstream, these files **will conflict**:

| File | Why |
|------|-----|
| `src/release.rs` | We removed most targets |
| `src/validation.rs` | Added FIPS file skipping |
| `cpython-unix/build-*.sh` | Added FIPS support |
| `.github/workflows/*.yml` | Modified upstream workflows |

**Resolution strategy:**
1. Keep our FIPS changes in build scripts
2. Keep our target reduction in release.rs
3. Keep our workflow trigger changes
4. Merge other upstream changes normally

The `v*` workflow files won't conflict (they're unique to our fork).

---

## 6. Troubleshooting

### Docker Image Loading Issues

If you see `ImageNotFound` errors, check:
1. Image job completed successfully
2. Image artifacts were uploaded
3. Build job downloaded the artifacts
4. Docker images were loaded (check debug output)

**Known issue:** Docker Buildx with containerd snapshotter creates different image IDs. Our workflows capture the actual loaded ID from `docker load` output to fix this.

### FIPS Build Failures

If OpenSSL FIPS builds fail:
1. Check that target is glibc-based (not musl)
2. Verify `enable-fips` and `install_fips` are in build commands
3. Check that `fipsmodule.cnf` was generated

### Namespace Runner Issues

If builds fail with runner errors:
1. Verify runner names in namespace configuration
2. Check that runners have Docker installed
3. Ensure runners have sufficient disk space (50GB+ recommended)

---

## 7. Additional Resources

- **Upstream repo:** https://github.com/astral-sh/python-build-standalone
- **Our fork:** https://github.com/verkada/python-build-standalone
- **OpenSSL FIPS:** https://www.openssl.org/docs/fips.html
- **Python release schedule:** https://peps.python.org/pep-0719/
