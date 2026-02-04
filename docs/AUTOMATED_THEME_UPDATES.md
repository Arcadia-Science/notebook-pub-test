# Automated Theme Updates for Notebook Pubs

## Rationale

### The Problem

We have a growing number of notebook pubs (7+ and counting) that all vendor the same Quarto theme extension (`notebook-pub-theme`). When the theme is updated, each pub must be manually updated with `quarto update extension`. This doesn't scale.

### Design Constraints

1. **Theme updates are orthogonal to scientific content.** The versioning system (git tags) exists to track changes to the science—notebooks, figures, conclusions. Theme updates (CSS, fonts, layout) shouldn't create new content versions.

2. **Minimal maintenance burden.** A "push model" where the theme repo dispatches to all pubs requires maintaining a list of repos, a PAT with access to all of them, and updating both whenever a new pub is created. A "pull model" where each pub checks for updates independently requires only adding a workflow to the template—new pubs inherit it automatically.

3. **Single PR for review.** Scientists should review one PR that updates the theme on `main`. After merging, the site should rebuild automatically without further intervention.

4. **Don't touch existing workflows.** The content release flow (`build.yml` → `publish.yml`) works well. Theme updates should be a parallel path that doesn't interfere.

5. **Theme releases are intentional.** The workflow only picks up tagged releases from the theme repo, not arbitrary commits. This gives theme maintainers control over when updates propagate and simplifies development—you can push work-in-progress without triggering updates downstream.

### The Solution

Two new workflows that form a parallel path to publishing:

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONTENT RELEASE PATH (existing)                                     │
│                                                                     │
│   Tag push (v*) → build.yml → _build.py → PR → publish.yml → gh-pages │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ THEME UPDATE PATH (new)                                             │
│                                                                     │
│   Daily check → check-theme-updates.yml → PR → publish-theme-change.yml → gh-pages │
└─────────────────────────────────────────────────────────────────────┘
```

These paths are completely independent. Theme updates sync only `_extensions/` to the `publish` branch and re-render—they don't run `_build.py` or create version tags.

---

## Workflow Specifications

### Workflow 1: `check-theme-updates.yml`

**Purpose:** Daily check for theme updates; create a PR if one is available.

**Trigger:**
- Schedule: daily at midnight PST (`0 8 * * *` UTC)
- Manual: `workflow_dispatch`

**Steps:**
1. Checkout repository (`main` branch)
2. Setup Quarto
3. Read current theme version from `_extensions/Arcadia-Science/arcadia-pub-theme/_extension.yml`
4. Fetch latest theme version from GitHub API (`/repos/Arcadia-Science/notebook-pub-theme/releases/latest`)
5. Compare versions; exit early if already up to date
6. Run `make update-theme`
7. Create PR with:
   - Branch: `auto/theme-update`
   - Title: `chore(deps): update arcadia-pub-theme X.X → Y.Y`
   - Labels: `dependencies`, `quarto`
   - Body: links to release notes and changelog

**Permissions:**
- `contents: write`
- `pull-requests: write`

**Behavior Notes:**
- Uses a fixed branch name so subsequent runs update the existing PR rather than creating duplicates
- `delete-branch: true` cleans up after merge

---

### Workflow 2: `publish-theme-change.yml`

**Purpose:** When a theme update merges to `main`, sync the theme to `publish` and re-render the site.

**Trigger:**
- Push to `main` branch where `_extensions/**` changed
- Manual: `workflow_dispatch`

**Steps:**
1. Check for open PRs targeting `publish` branch; if any exist, exit early (a content release is in progress—the theme update will go live when that merges)
2. Checkout `main` branch (into `main-branch/`)
3. Checkout `publish` branch (into `publish-branch/`)
4. Delete `publish-branch/_extensions/`
5. Copy `main-branch/_extensions/` to `publish-branch/_extensions/`
6. Commit and push to `publish` (exit early if no changes)
7. Setup Quarto
8. Run `quarto publish gh-pages --no-prompt` from `publish-branch/`

**Permissions:**
- `contents: write`

**Behavior Notes:**
- **Guards against race condition:** If a build PR is open (content release in progress), the workflow exits early. This prevents publishing a half-built state. The theme update isn't lost—it's already on `main` and will be included when the content release eventually merges and publishes.
- Works directly on `publish` branch without creating a PR
- Does NOT run `_build.py`—the versioned notebooks already exist on `publish` from the last content release
- Self-contained: does its own render/publish rather than triggering `publish.yml`
