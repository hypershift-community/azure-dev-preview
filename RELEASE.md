# Release Process for Maintainers

This document describes how to create a new release of the HyperShift Azure Dev-Preview.

## Overview

Releases consist of:
- Multi-architecture CLI binaries (linux/darwin × amd64/arm64)
- ARTIFACTS.yaml manifest (tracks versions and compatibility)
- SHA256 checksums for verification
- GitHub release with all assets

## Prerequisites

- [ ] Access to push to this repository
- [ ] `gh` CLI authenticated with GitHub
- [ ] Go 1.24+ installed
- [ ] `task` CLI installed
- [ ] Access to upstream HyperShift repository (for building CLI)

## Release Steps

### 1. Determine New Version

Check the upstream HyperShift release you want to base this dev-preview on:

```bash
# Check available upstream tags
cd /path/to/upstream/hypershift
git fetch --tags
git tag | grep ^v0.1 | sort -V | tail -5
```

### 2. Update VERSION File

```bash
# Update VERSION file with new version (without 'v' prefix)
echo "0.1.70" > VERSION
```

### 3. Update ARTIFACTS.yaml

Edit `ARTIFACTS.yaml` to update:

```yaml
version: v0.1.70  # Update version
releaseDate: "2025-11-19"  # Update date

cli:
  version: v0.1.70  # Update
  sourceCommit: <new-commit-hash>  # Get from: git rev-parse v0.1.70

operators:
  hypershift:
    image: quay.io/redhat-user-workloads/.../hypershift-operator-main:v0.1.70  # Update if changed

openshift:
  version: "4.22.0-ec.1"  # Update recommended OCP version
  releaseImage: quay.io/openshift-release-dev/ocp-release@sha256:...  # Update
```

To get the recommended OCP release for a HyperShift version, check the upstream docs or release notes.

### 4. Build CLI Binaries

```bash
# Option A: Build from public upstream
task build:cli

# Option B: Build from local upstream clone (for testing)
UPSTREAM_URL=file:///path/to/local/hypershift task build:cli
```

This will:
- Clone the upstream repository
- Checkout the version tag
- Build for all 4 platforms (takes ~5-10 minutes)
- Place binaries in `bin/{os}/{arch}/hypershift`

### 5. Create Tarballs

```bash
task build:tarballs
```

This creates:
- `release/hypershift-linux-amd64.tar.gz`
- `release/hypershift-linux-arm64.tar.gz`
- `release/hypershift-darwin-amd64.tar.gz`
- `release/hypershift-darwin-arm64.tar.gz`

Each tarball contains:
- `hypershift` binary
- `RELEASE-INFO.txt` with version info and links

### 6. Generate Checksums

```bash
task build:checksums
```

This creates `release/SHA256SUMS.txt` with checksums for all tarballs.

### 7. Verify Artifacts

```bash
# Check all files exist
ls -lh release/

# Verify a binary works
tar -xzf release/hypershift-linux-amd64.tar.gz
./hypershift version

# Verify checksums
cd release && sha256sum -c SHA256SUMS.txt
```

### 8. Commit and Push Changes

Commit the version files and push to main. The commit you push will be tagged in the next step.

```bash
git add VERSION ARTIFACTS.yaml
git commit -s -m "chore: Release v0.1.70

Update to HyperShift v0.1.70 with recommended OCP 4.22.0-ec.1

Signed-off-by: Your Name <your.email@example.com>"

git push origin main
```

!!! important "This commit will be tagged"
    The commit you just pushed will be automatically tagged as `v0.1.70` when you create the GitHub release in the next step. Make sure this is the commit you want to release.

### 9. Create GitHub Release and Tag

Run the release task to create both the Git tag and GitHub release:

```bash
task release:cli
```

**What this does:**

1. **Creates a Git tag**: Automatically creates tag `v{VERSION}` (e.g., `v0.1.70`) pointing to the current HEAD on main (the commit you just pushed)
2. **Creates GitHub release**: Creates a GitHub release with that tag
3. **Uploads assets**:
   - All 4 CLI tarballs (`hypershift-{platform}-{arch}.tar.gz`)
   - `SHA256SUMS.txt` checksum file
   - `ARTIFACTS.yaml` manifest

**You do NOT need to:**
- Manually create tags with `git tag`
- Manually push tags with `git push --tags`

The `gh release create` command handles tag creation automatically.

The release will be available at: `https://github.com/hypershift-community/azure-dev-preview/releases/tag/v<VERSION>`

### 10. Verify Release

- [ ] Visit the GitHub release page
- [ ] Verify all 6 assets are uploaded (4 tarballs + checksum + manifest)
- [ ] Download and test a binary:
  ```bash
  curl -LO https://github.com/hypershift-community/azure-dev-preview/releases/latest/download/hypershift-linux-amd64.tar.gz
  tar -xzf hypershift-linux-amd64.tar.gz
  ./hypershift version
  ```

### 11. Update Documentation (if needed)

If the new release requires documentation changes:
- Update any version-specific examples
- Update compatibility notes
- Create a changelog entry if significant changes

### 12. Announce Release

- Update any tracking issues or project boards
- Notify customers via appropriate channels
- Update any related repositories that depend on this version

## Troubleshooting

### Build fails with vendoring errors

The upstream repository may have vendoring inconsistencies. This is expected - the build task clones fresh and uses the vendored dependencies as-is.

### Release already exists

If you need to recreate a release:

```bash
# Delete the existing release and tag
gh release delete v0.1.70 --yes
git push origin :refs/tags/v0.1.70

# Then recreate
task release:cli
```

### Task says "up to date" but I want to rebuild

```bash
# Clean and rebuild
task clean
task build:cli
```

## Idempotent Tasks

All build tasks are idempotent:

- `task build:cli` - Skipped if `bin/` binaries exist
- `task build:tarballs` - Skipped if `release/*.tar.gz` exist
- `task build:checksums` - Skipped if `release/SHA256SUMS.txt` exists
- `task release:cli` - Skipped if GitHub release exists

Run `task clean` to reset state and force rebuild.

## Quick Reference

```bash
# Full release workflow
echo "0.1.70" > VERSION
# Edit ARTIFACTS.yaml (version, releaseDate, cli.version, cli.sourceTag, cli.sourceCommit, openshift.*)
task clean                    # Optional: clean previous build
task build:cli               # ~5-10 minutes
task build:tarballs          # ~1 minute
task build:checksums         # Instant
git add VERSION ARTIFACTS.yaml && git commit -s -m "chore: Release v0.1.70"
git push origin main
task release:cli             # Creates Git tag v0.1.70 + GitHub release with assets

# Verify
gh release view v0.1.70
```

## Files Generated (gitignored)

These files are created during the build but are gitignored:

- `bin/` - Compiled binaries
- `release/` - Tarballs and checksums
- `.build/` - Temporary build directory

Only `VERSION` and `ARTIFACTS.yaml` are version controlled.
