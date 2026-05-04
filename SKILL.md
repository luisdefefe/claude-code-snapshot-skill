---
name: snapshot
description: Save a versioned snapshot of the current project page at <page>/snapshot/v{N}/, alongside a SUMMARY.md describing the decisions in this version. Use when the user types /snapshot or asks to "snapshot," "take a snapshot," or "save this version" of a project page they're working on. Designed for static portfolio sites where past versions of a project are referenced from articles.
---

# Snapshot

Preserves the current state of a project page as a frozen, linkable version at `<page>/snapshot/v{N}/`. Each snapshot is a self-contained copy of the page that stays browsable at a stable URL, plus a `SUMMARY.md` describing what this version is and why.

## Site URL (optional)

To make the skill's report include a clickable URL instead of just a path, fill in your site's base URL below (e.g., `https://example.com`). Leave blank to skip — the skill will fall back to auto-detection, then to a relative path.

Site URL: 

## When to invoke

Invoke when the user explicitly asks for a snapshot — "snapshot this," "take a snapshot," "/snapshot," "save this version." Do not invoke proactively or as a side effect of other work.

## What to do

Follow these steps in order. Keep status messages short — one line per step is plenty.

### 1. Identify the page

Infer which page is being snapshotted from the conversation context — what project or page the user has been working on or discussing in this session, recently edited files, the page they're focused on. The current working directory is one signal among many; in typical Claude Code usage cwd is the repo root, not a page directory, so don't lean on it as a default.

A page directory is one that contains an `index.html` (and usually some CSS/JS/assets that belong to that page). If context doesn't make the page clear, or points to several candidates, ask the user. Don't guess.

### 2. Pick the next version number

Look in `<page>/snapshot/`. If it doesn't exist yet, this snapshot is `v1`. Otherwise, find the highest existing `v{N}` (where N is an integer) and use `v{N+1}`. Ignore folders inside `snapshot/` that don't match the `v{integer}` pattern.

### 3. Pre-flight checks

Before copying anything, run two checks against the source files:

**a. Relative-path check.** Scan the page's HTML, CSS, and JS files for paths starting with `../`. If any are found, surface them to the user:

> Heads up: `index.html` references `../shared/foo.png`. The snapshot copy will live two folders deeper than the original page, so this link will break in the snapshot. Continue anyway?

Wait for confirmation before continuing. Root-relative paths (`/styles/main.css`) and same-folder paths (`./foo.png`, `foo.png`) are safe — no warning needed.

**b. Size check.** Estimate the total size of files about to be copied (excluding the existing `snapshot/` folder). If above 5 MB, surface it:

> Heads up: this snapshot is ~14 MB. Each snapshot duplicates the page's assets, so the repo will grow with each one. Continue?

Wait for confirmation. Below 5 MB, skip silently.

### 4. Create and populate the snapshot folder

Create `<page>/snapshot/v{N}/`. Copy everything from the page directory into it, **excluding the `snapshot/` subfolder itself**. Preserve any nested directory structure. Skip `.DS_Store` and other OS junk.

### 5. Inject snapshot meta tags

For every HTML file inside the new snapshot folder (top-level and nested), inject the following into `<head>`, immediately after `<meta charset>` if present, otherwise at the start of `<head>`:

```html
<meta name="robots" content="noindex, nofollow">
<link rel="canonical" href="/<canonical-url>/">
```

`<canonical-url>` is the URL path where the live (canonical) version of *this specific HTML file* lives. For a page at `projects/forest/index.html`, that's `/projects/forest/`. For a nested page like `projects/forest/about/index.html`, the canonical is `/projects/forest/about/`. For the site root (`index.html` at the repo root), the canonical is `/`.

The `noindex` keeps snapshots out of search results; the canonical link tells search engines the live page is the authoritative version of this content.

### 6. Generate `SUMMARY.md`

Write `SUMMARY.md` inside the snapshot folder. Use `templates/summary.md` (in this skill's folder) as a structural starting point and fill it in based on the current conversation:

- **What this version is** — one short paragraph describing the project's state at this snapshot. What it is, what it does, the overall shape.
- **Decisions in this version** — design or product choices made, things tried and rejected, constraints worked around. Pull from the conversation.
- **Changes from v{N-1}** — what's different vs. the prior version, visually/functionally/structurally. Skip this section entirely for `v1`.
- **Likely next moves** — based on the conversation, what feels like the next iteration: problems hit, things parked, ideas in flight.

Date it at the top.

If the conversation doesn't have enough context to fill a section meaningfully, write one honest sentence (e.g., "Light context for this snapshot — see article for details") rather than padding. The summary is meant to capture *real* decisions while they're fresh, not to look comprehensive.

### 7. Commit and push

Stage the new snapshot folder. Commit with a message in this shape:

```
Snapshot <page-name> v{N}

<one-line summary of what's being preserved>
```

Push to the current branch.

If the user's permission setup blocks commit or push, the standard Claude Code permission prompt will handle it — don't try to work around it.

If the user has indicated (in CLAUDE.md or earlier in the session) that they prefer to commit manually, stop after writing files and tell them what to commit instead.

### 8. Report back

Three to five lines, no more:

- Where the snapshot lives in the repo: `<page>/snapshot/v{N}/`
- The URL it'll be live at once the host finishes deploying. Build it by:
  1. Reading the **Site URL** value near the top of this `SKILL.md`. If filled in, use it.
  2. Otherwise, auto-detect from `vercel.json`, `package.json`'s `homepage` field, or the `origin` remote in `.git/config`.
  3. If neither yields a domain, give the path instead: "Live at `/<page>/snapshot/v{N}/` once your deployment finishes."
- Any warnings that came up during pre-flight (relative paths, size).

## What this skill does not do

- Doesn't modify the canonical (live) version of the page. The original `<page>/index.html` is untouched.
- Doesn't create branches or pull requests. Snapshots commit directly to the current branch.
- Doesn't delete or prune old snapshots. Manage cleanup manually.
- Doesn't handle hosts without auto-deploy on git push. For those, the user runs their normal deploy step after the commit lands.

## Customization notes for the user

- **To disable auto-commit**, replace step 7 with: "Stop here. Tell the user what was created and let them commit when ready."
- **To rename the convention** from `snapshot/vN` to `versions/vN` or similar, change every reference to `snapshot/` in this file. The URL convention is the only thing the skill cares about.
