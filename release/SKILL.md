---
name: release
description: Create a git tag and GitHub release based on changes since the last release. Use when the user asks to make a release, create a tag/release, cut a release, or publish a new version.
---

# Release Skill

Create a semantically versioned git tag and GitHub release with auto-generated release notes based on changes since the last release.

## Workflow

### Step 0: Ensure on Main Branch

Run `git branch --show-current` to check the current branch. If not on `main`, switch to it:
```
git checkout main && git pull origin main
```

Also run `git status --short` to check for uncommitted changes. If there are uncommitted changes, inform the user and **stop**.

### Step 1: Gather Current State

Run these commands in parallel:

1. `git tag --sort=-creatordate | head -5` — get the latest tag
2. `gh release list --limit 1` — confirm the latest GitHub release
3. `git fetch origin main` — ensure main is up to date with remote

### Step 2: Analyze Changes Since Last Release

Using the latest tag (e.g. `v1.0.0`):

1. `git log <last_tag>..HEAD --oneline` — list all commits since last release
2. `git log <last_tag>..HEAD --format="%h %s%n%b" --no-merges` — get detailed commit messages (skip merge commits)

If there are **no commits** since the last tag, inform the user and stop.

### Step 3: Determine Version Bump

Analyze the changes and determine the appropriate semver bump:

- **Major** (X.0.0): Breaking changes, major API changes, incompatible migrations
- **Minor** (x.Y.0): New features, new functionality, significant enhancements
- **Patch** (x.y.Z): Bug fixes, CI/CD changes, documentation updates, dependency bumps, refactors with no user-facing changes

### Step 4: Confirm with User

Use `AskUserQuestion` to present:
- The proposed version number
- A summary of changes grouped by category
- Ask if they want to proceed or adjust the version

### Step 5: Update Version File

Look for a version file in the project root and update it to the new version:
- `pyproject.toml` → update `version = "X.Y.Z"`
- `package.json` → update `"version": "X.Y.Z"`
- `Cargo.toml` → update `version = "X.Y.Z"`

If the user passed "update toml too", "update version", or similar as an argument, this step is required. Otherwise, ask the user if they want the version file updated.

### Step 6: Commit and Push Version Bump

Since `main` is protected and does not allow direct pushes, the version bump must go through a PR:

1. Create a branch: `git checkout -b chore/bump-<version>`
2. Commit the version file change: `git commit -m "chore: bump version to <version>"`
3. Push the branch: `git push -u origin chore/bump-<version>`
4. Create a PR: `gh pr create --title "chore: bump version to <version>" --body "Bump version for release"`
5. Wait for CI checks: `gh pr checks <pr_number> --watch`
6. Merge the PR: `gh pr merge <pr_number> --merge --delete-branch`
7. Switch back to main and pull: `git checkout main && git pull origin main`

### Step 7: Create Tag and Release

1. Create an annotated git tag:
   ```
   git tag -a <version> -m "<version>: <short summary>"
   ```

2. Push the tag:
   ```
   git push origin <version>
   ```

3. Create the GitHub release with `gh release create`:
   ```
   gh release create <version> --title "<version>" --notes "<release notes>"
   ```

### Step 8: Verify

Run in parallel:
- `git tag -l <version>` — confirm tag exists
- `gh release view <version>` — confirm release is published

Report the release URL to the user.

## Release Notes Format

Group changes into categories. Only include categories that have changes:

```
## What's Changed

### Breaking Changes
- ...

### Features
- ...

### Bug Fixes
- ...

### CI/CD Improvements
- ...

### Documentation
- ...

### Maintenance
- ...

**Full Changelog**: https://github.com/<owner>/<repo>/compare/<prev_tag>...<new_tag>
```

Derive the owner/repo from `gh repo view --json nameWithOwner -q .nameWithOwner`.

## Rules

- **Only release from the `main` branch** — refuse to create tags/releases from any other branch
- Always use annotated tags (`git tag -a`), not lightweight tags
- Always confirm the version with the user before creating anything
- Use the conventional `v` prefix for tags (e.g. `v1.2.3`)
- If the user passes an argument (e.g. `/release v2.0.0`), use that as the version but still confirm
- Write concise release notes — one line per change, no commit hashes in notes
- Group related commits (e.g. multiple commits from one PR) into a single bullet point
