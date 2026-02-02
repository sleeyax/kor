# Manual Helm Chart Release Instructions

This document provides detailed instructions for manually releasing a Helm chart for the kor project on a fork. These instructions are useful when you need to create a release independently from the automated chart-releaser-action workflow.

## Prerequisites

Before starting the release process, ensure you have the following tools installed:

- **Helm CLI** (`helm`) - For packaging Helm charts
- **GitHub CLI** (`gh`) - For creating GitHub releases
- **Git** - For version control operations
- **sha256sum** (or equivalent) - For calculating chart digests

Verify installations:
```bash
helm version
gh --version
git --version
sha256sum --version  # or 'shasum -a 256' on macOS
```

## When to Use Manual Releases

Use this manual process when:
- Working on a fork and need to test chart changes before upstreaming
- The automated chart-releaser-action workflow is not available or configured
- You need to create a quick patch release for testing
- Iterating on chart development and need rapid testing cycles

## Release Process Overview

The manual release process consists of:
1. Update chart version in Chart.yaml
2. Commit the version bump
3. Package the Helm chart
4. Create a GitHub release with the chart package
5. Update the Helm repository index (index.yaml)
6. Commit and push the updated index

## Detailed Step-by-Step Instructions

### Step 1: Determine Version Numbers

Follow semantic versioning for both chart and app versions:

- **Patch version** (0.2.X): Bug fixes, documentation updates, minor improvements
- **Minor version** (0.X.0): New features, non-breaking changes
- **Major version** (X.0.0): Breaking changes

**Current versions** (as of this document):
- Chart version: `0.2.14`
- App version: `0.6.8`

### Step 2: Switch to Your Feature Branch

Ensure you're on the correct branch with your changes:

```bash
# Check current branch
git branch --show-current

# Switch to your feature branch if needed
git checkout feat/role-customizations

# Pull latest changes
git pull origin feat/role-customizations

# Verify you have the latest commits
git log --oneline -5
```

### Step 3: Update Chart.yaml

Edit `charts/kor/Chart.yaml` and update the version fields:

```yaml
apiVersion: v2
name: kor
description: A Kubernetes Helm Chart to discover orphaned resources using kor
type: application
version: 0.2.X  # <- Increment this (chart version)
appVersion: "0.6.X"  # <- Update if app code changed
maintainers:
  - name: "sleeyax"  # <- Update to your GitHub username for fork releases
    url: "https://github.com/sleeyax/kor"  # <- Update to your fork URL
annotations:
  "artifacthub.io/license": MIT
  "artifacthub.io/links": |
    - name: Application Source
      url: https://github.com/sleeyax/kor  # <- Update to your fork
    - name: Chart Source
      url: https://github.com/sleeyax/kor/tree/feat/role-customizations/charts/kor  # <- Update branch/path
    - name: Grafana Dashboard
      url: https://grafana.com/grafana/dashboards/19863-kor-dashboard/
```

**Important**: For fork releases, update:
- `maintainers.name` to your GitHub username
- `maintainers.url` to your fork URL
- Chart Source URL to point to your fork and branch

### Step 4: Commit Version Bump

```bash
# Stage the changes
git add charts/kor/Chart.yaml

# Commit with a descriptive message
git commit -m "chore: bump chart version to 0.2.X"

# Push to your fork
git push origin feat/role-customizations
```

### Step 5: Package the Helm Chart

Package the chart to a temporary directory:

```bash
# Package the chart
helm package charts/kor -d /tmp/

# Verify the package was created
ls -lh /tmp/kor-0.2.X.tgz
```

This creates `/tmp/kor-0.2.X.tgz` where X is your version number.

### Step 6: Calculate the SHA256 Digest

Calculate the digest for the chart package (needed for index.yaml):

```bash
# Linux
sha256sum /tmp/kor-0.2.X.tgz | cut -d' ' -f1

# macOS
shasum -a 256 /tmp/kor-0.2.X.tgz | cut -d' ' -f1
```

**Save this digest** - you'll need it for updating index.yaml in Step 8.

Example output:
```
9b2c8932744bd64d2bc2c7b0c309dbb897f8751f2b6721f09de8d83f65253de4
```

### Step 7: Create GitHub Release

Create a GitHub release with the packaged chart:

```bash
gh release create kor-0.2.X \
  /tmp/kor-0.2.X.tgz \
  --title "kor-0.2.X" \
  --notes "Brief description of changes in this release" \
  -R <your-github-username>/kor
```

**Example**:
```bash
gh release create kor-0.2.14 \
  /tmp/kor-0.2.14.tgz \
  --title "kor-0.2.14" \
  --notes "Release with skip namespace validation option for RBAC filters" \
  -R sleeyax/kor
```

**Verify the release**:
```bash
gh release view kor-0.2.X --repo <your-github-username>/kor
```

### Step 8: Update index.yaml on gh-pages Branch

The Helm repository index must be updated to include the new release.

#### 8.1 Switch to gh-pages Branch

```bash
# Stash any uncommitted changes if needed
git stash

# Switch to gh-pages branch
git checkout gh-pages

# Pull latest changes
git pull origin gh-pages
```

#### 8.2 Get Current Timestamp

Generate timestamps for the index entry:

```bash
# Get current timestamp in ISO 8601 format with nanoseconds
date -u +"%Y-%m-%dT%H:%M:%S.%NZ"
```

Example output:
```
2026-01-28T21:58:58.302278074Z
```

**Save this timestamp** - you'll need it for both the `created` field and the `generated` field.

#### 8.3 Edit index.yaml

Add a new entry at the top of the `kor:` entries list (after line 3 in `index.yaml`):

```yaml
  - annotations:
      artifacthub.io/license: MIT
      artifacthub.io/links: |
        - name: Application Source
          url: https://github.com/<your-username>/kor
        - name: Chart Source
          url: https://github.com/<your-username>/kor/tree/<your-branch>/charts/kor
        - name: Grafana Dashboard
          url: https://grafana.com/grafana/dashboards/19863-kor-dashboard/
    apiVersion: v2
    appVersion: 0.6.X
    created: "2026-01-28T21:58:58.302278074Z"  # <- Use timestamp from step 8.2
    description: A Kubernetes Helm Chart to discover orphaned resources using kor
    digest: 9b2c8932744bd64d2bc2c7b0c309dbb897f8751f2b6721f09de8d83f65253de4  # <- Use digest from step 6
    maintainers:
    - name: <your-username>
      url: https://github.com/<your-username>/kor
    name: kor
    type: application
    urls:
    - https://github.com/<your-username>/kor/releases/download/kor-0.2.X/kor-0.2.X.tgz
    version: 0.2.X
```

**Also update** the `generated:` timestamp at the bottom of the file (last line) to match the timestamp from step 8.2.

#### 8.4 Commit and Push index.yaml

```bash
# Stage the changes
git add index.yaml

# Commit with a descriptive message
git commit -m "chore: update index.yaml for kor-0.2.X"

# Push to gh-pages branch
git push origin gh-pages
```

### Step 9: Verify the Release

#### 9.1 Verify GitHub Release

```bash
gh release view kor-0.2.X --repo <your-username>/kor
```

#### 9.2 Add Helm Repository

```bash
# Add your fork's Helm repository
helm repo add kor-fork https://<your-username>.github.io/kor

# Update repository index
helm repo update

# Search for your chart version
helm search repo kor-fork/kor --versions | head -5
```

#### 9.3 Pull and Inspect Chart

```bash
# Pull the chart
helm pull kor-fork/kor --version 0.2.X

# List contents
tar -tzf kor-0.2.X.tgz | head -20

# Show chart metadata
helm show chart kor-fork/kor --version 0.2.X

# Show default values
helm show values kor-fork/kor --version 0.2.X
```

#### 9.4 Test Installation (Optional)

If you have a Kubernetes cluster available:

```bash
# Dry run to validate
helm install kor-test kor-fork/kor \
  --version 0.2.X \
  --dry-run \
  --debug

# Actual installation (if desired)
helm install kor-test kor-fork/kor --version 0.2.X
```

## Troubleshooting

### Issue: Release Not Found After Creation

**Symptom**: `gh release view` fails or Helm can't find the chart

**Solution**:
1. Verify the release exists on GitHub: `https://github.com/<your-username>/kor/releases`
2. Check that the tag matches the chart version exactly
3. Ensure index.yaml was pushed to gh-pages
4. Wait a few minutes for GitHub Pages to update
5. Clear Helm cache: `helm repo update`

### Issue: SHA256 Digest Mismatch

**Symptom**: Helm reports digest mismatch when pulling chart

**Solution**:
1. Recalculate the digest: `sha256sum /tmp/kor-0.2.X.tgz`
2. Update the `digest` field in index.yaml with the correct value
3. Commit and push the corrected index.yaml

### Issue: Chart Version Already Exists

**Symptom**: Cannot create release because tag already exists

**Solution**:
1. Delete the existing release: `gh release delete kor-0.2.X --repo <your-username>/kor --yes`
2. Delete the tag: `git push origin :refs/tags/kor-0.2.X`
3. Increment the version number and start again

### Issue: GitHub Pages Not Updating

**Symptom**: `helm repo add` works but can't find new chart version

**Solution**:
1. Verify gh-pages branch was pushed: `git log gh-pages --oneline -1`
2. Check GitHub Pages settings in repository settings
3. Wait 5-10 minutes for GitHub Pages to rebuild
4. Force Helm cache refresh: `helm repo remove kor-fork && helm repo add kor-fork https://<your-username>.github.io/kor`

## Differences from Automated Releases

The automated chart-releaser-action workflow (`.github/workflows/chart-release.yml`):

**Automated workflow**:
- Triggers on `v*` or `kor-*` tags
- Automatically packages charts
- Generates index.yaml entries
- Handles timestamps and digests automatically
- No manual gh-pages management needed

**Manual process**:
- Full control over timing and versioning
- Useful for testing on forks
- Requires manual index.yaml maintenance
- More error-prone but more flexible
- Faster iteration for development

## Quick Reference Commands

```bash
# Full release process in sequence
git checkout feat/role-customizations
# Edit charts/kor/Chart.yaml (bump version)
git add charts/kor/Chart.yaml
git commit -m "chore: bump chart version to 0.2.X"
git push origin feat/role-customizations
helm package charts/kor -d /tmp/
sha256sum /tmp/kor-0.2.X.tgz | cut -d' ' -f1  # Save this
gh release create kor-0.2.X /tmp/kor-0.2.X.tgz --title "kor-0.2.X" --notes "..." -R <user>/kor
git checkout gh-pages
# Edit index.yaml (add entry with digest and timestamp)
git add index.yaml
git commit -m "chore: update index.yaml for kor-0.2.X"
git push origin gh-pages
helm repo update
helm search repo kor-fork/kor --versions | head -5
```

## Notes

- Always test chart changes locally with `helm lint` and `helm template` before releasing
- Keep chart versions and app versions aligned with changes
- Document significant changes in release notes
- For upstream contributions, use the automated workflow on the main repository
- This manual process is primarily for fork testing and development
