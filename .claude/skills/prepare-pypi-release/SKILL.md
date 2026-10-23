---
name: prepare-pypi-release
description: Bump version, update CHANGELOG, validate the build, and open a release PR for abench-speckz. Merging the PR publishes to PyPI via GitHub Actions. Run when the user asks to prepare, cut, or publish a release.
---

Prepare abench-speckz for a PyPI release. Follow these steps in order.

Publishing is automated: `.github/workflows/release.yml` runs on every push to `main`, and when the
version in `pyproject.toml` has no matching `vX.Y.Z` tag it re-runs CI, builds, publishes to PyPI
(Trusted Publishing, `pypi` environment), then creates the `vX.Y.Z` tag and a GitHub Release whose
notes are the version's CHANGELOG section. This skill's job ends at opening the release PR —
never `twine upload`, tag, or push to `main` directly.

## Step 1 — Determine the new version

Check what has changed since the last release tag:

```bash
# What version is current?
grep '^version' pyproject.toml

# What commits are unreleased?
git log $(git describe --tags --abbrev=0 2>/dev/null || git rev-list --max-parents=0 HEAD)..HEAD --oneline
```

Apply semantic versioning to the unreleased commits:
- **patch** (0.x.Z) — bug fixes only
- **minor** (0.X.0) — any new feature, new field, or new behavior (backward-compatible)
- **major** (X.0.0) — breaking changes to the YAML schema, CLI flags, or public API

If the user specifies a version explicitly, use that.

## Step 2 — Update the three version locations

Version is stored in three places — all must match:

| File | Field |
|---|---|
| `pyproject.toml` | `version = "X.Y.Z"` |
| `src/abench_speckz/__init__.py` | `__version__ = "X.Y.Z"` |
| `CHANGELOG.md` | Replace `## [Unreleased]` with `## [Unreleased]\n\n## [X.Y.Z] - YYYY-MM-DD` |

Today's date for the CHANGELOG entry:

```bash
date +%Y-%m-%d
```

### CHANGELOG format

Move all content from the `## [Unreleased]` section into the new `## [X.Y.Z]` entry. If `[Unreleased]` is empty, summarise the unreleased commits yourself under the appropriate headings (`### Added`, `### Fixed`, `### Changed`, `### Removed`). Keep `## [Unreleased]` as an empty placeholder above the new entry.

## Step 3 — Lint, format, and type checks

```bash
source .venv/bin/activate
ruff check src/ tests/
ruff format --check src/ tests/
mypy
```

All three must exit 0 before proceeding. Fix any issues:

- `ruff check --fix src/ tests/` auto-fixes safe lint violations
- `ruff format src/ tests/` auto-formats code
- Mypy type errors must be fixed manually

Re-run the checks after fixing to confirm clean. If `ruff` is not installed, install it:

```bash
pip install ruff -q
```

## Step 4 — Run tests

```bash
source .venv/bin/activate
pytest -x -q
```

All tests must pass before proceeding. Fix any failures first.

## Step 5 — Build the distribution

```bash
source .venv/bin/activate
pip install build twine -q
python -m build
```

Expected output ends with:
```
Successfully built abench_speckz-X.Y.Z.tar.gz and abench_speckz-X.Y.Z-py3-none-any.whl
```

## Step 6 — Validate the build

```bash
source .venv/bin/activate
twine check dist/abench_speckz-X.Y.Z*
```

Both the `.whl` and `.tar.gz` must show `PASSED`. If either fails, fix the issue (usually a malformed README or missing metadata) before committing.

## Step 7 — Commit on a release branch

Stage only the three version files (do not stage `dist/`), on a new branch:

```bash
git switch -c release/vX.Y.Z
git add pyproject.toml src/abench_speckz/__init__.py CHANGELOG.md
git commit -m "Bump version to X.Y.Z, update CHANGELOG"
```

## Step 8 — Push and open the PR

```bash
git push -u origin release/vX.Y.Z
gh pr create --base main --title "Release vX.Y.Z" --body "<the new CHANGELOG section>"
```

## Step 9 — Report to the user

Tell the user:
- The new version and the PR link
- That merging the PR publishes to PyPI: the `release` workflow tests, builds, uploads, then creates the
  `vX.Y.Z` tag and GitHub Release (the publish job may wait for approval on the `pypi` environment)
- They can watch it with `gh run watch` or under the repo's Actions tab after merging

## Gotchas

- **`dist/` is not in `.gitignore`** — do not `git add dist/` or it will end up in the repo.
- **Version must match in all three files.** A mismatch between `pyproject.toml` and `__init__.py` will confuse users importing the package.
- **Do not amend published commits.** Once a version tag is pushed, create a new patch release for any follow-up fixes.
- **Do not create the `vX.Y.Z` tag yourself.** The release workflow treats an existing tag as "already released" and skips publishing.
- **Do not leave `## [X.Y.Z]` missing from CHANGELOG.md.** The workflow uses that section as the GitHub Release notes.
- **`python -m build` uses the installed setuptools**, not the venv's source tree. Make sure `pip install -e '.[dev]'` was run in the active venv.
- **PyPI releases are permanent.** Once a version is uploaded it cannot be replaced — only yanked. Double-check `twine check` passes before opening the PR (the release workflow runs `twine check --strict`).
