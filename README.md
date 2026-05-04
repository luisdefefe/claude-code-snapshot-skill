# /snapshot

A Claude Code skill for preserving versions of project pages on a static site.

If you build web projects with Claude Code and want to write about how they evolved — *"version 1 was this, version 2 changed that, version 3 is what's live now"* — `/snapshot` saves the current state of a project as a frozen, browsable copy at a stable URL like `yoursite.com/projects/ProjectName/snapshot/v1/`. Each snapshot also gets a `SUMMARY.md` written by Claude, capturing what decisions were made in that version while the context is still fresh.

The point is to create anchor points as you make design decisions so that it is easy to reference multiple version side by side. This is especially useful if you are writing about how a project developed over time. By the time you sit down to write *how* a project evolved, the *why* behind each version has usually faded. This skill captures both — a working version you can link to, plus a note from Claude written when the thinking was still in the room.

## What it does

You're working on a project page in Claude Code. You type `/snapshot` (or just say "take a snapshot"). Claude:

1. Figures out which page you're snapshotting (defaults to the current directory).
2. Picks the next version number (`v1`, `v2`, `v3`...).
3. Copies the page's files to `<page>/snapshot/vN/`.
4. Injects `noindex` and a `canonical` link into the copied HTML, so the snapshot doesn't compete with your live page in search.
5. Writes a `SUMMARY.md` in the snapshot folder describing what version this is, what decisions got made, and what's likely next — based on your conversation.
6. Commits and pushes.

Within a minute (depending on your host), the snapshot is live. You can link to it from an article like:

```markdown
- [v1](/projects/ProjectName/snapshot/v1/) — first layout, felt static
- [v2](/projects/ProjectName/snapshot/v2/) — added parallax, ran into perf issues
- [v3](/projects/ProjectName/) — current, cut the parallax
```

## Install

In your project repo:

```bash
mkdir -p .claude/skills
git clone https://github.com/<you>/snapshot-skill .claude/skills/snapshot
```

Or just download this folder and drop it at `.claude/skills/snapshot/` inside your repo. No install step, no dependencies, no build.

To make the skill available across all your projects, install it at `~/.claude/skills/snapshot/` instead.

## How to use

Inside a project page directory, in Claude Code:

```
/snapshot
```

Or just say "take a snapshot of this." Claude will figure out which page you mean (or ask if it's ambiguous) and run through the steps.

## URL convention

Snapshots live at `<page-path>/snapshot/v{N}/`, mirroring the page's location in your repo:

| Page is at... | Snapshot v1 lives at... |
|---|---|
| `/projects/ProjectName/` | `/projects/ProjectName/snapshot/v1/` |
| `/writing/some-article/` | `/writing/some-article/snapshot/v1/` |
| `/` (the homepage) | `/snapshot/v1/` |

Stable, predictable, and the same convention works on any site structure.

## What gets injected into snapshots

Each snapshot's HTML gets two additions in `<head>`:

```html
<meta name="robots" content="noindex, nofollow">
<link rel="canonical" href="/projects/forest/">
```

`noindex` keeps snapshots out of search engines. The canonical link points back to the live page, so if a snapshot does end up in front of Google somehow, it knows the live page is the real version of this content.

The original (canonical) page is never modified. Only the copy inside the snapshot folder gets these tags.

## Hosts

Works with any host that deploys when you push to your git branch:

- Vercel
- Netlify
- Cloudflare Pages
- GitHub Pages
- anything similar

The skill itself is host-agnostic — it just writes files and commits. Whatever happens after the push is your host's problem. For hosts with manual deploy steps, run your deploy after the snapshot commit lands.

## Things to know

**Snapshots accumulate in your repo.** Each one duplicates the page's files. For pages that are mostly text, HTML, and CSS, this is negligible. If your page has heavy assets (images, video, fonts), the skill will warn you when a snapshot is large (> 5 MB) and let you decide.

**Relative paths going up (`../`) will break in snapshots.** A snapshot lives two folders deeper than the original page (`<page>/snapshot/vN/`), so a path like `../shared/foo.png` no longer points to the same place. The skill scans for these before copying and warns you. Most pages use root-relative paths (`/styles/main.css`) or same-folder paths and aren't affected — but if your page does reach outside its own folder, you'll need to either copy those assets into the page folder first, or accept that the link will break in the snapshot.

**`SUMMARY.md` lives inside the snapshot folder.** It's a Markdown file, not deployed as a webpage by default, but reachable at the URL if someone knows where to look. Treat it as semi-public notes — it'll become the basis for an article anyway.

**The skill commits and pushes by default.** Claude Code's permission system is the actual gate — if you haven't granted commit/push permissions for the session, you'll be prompted. To disable auto-commit entirely, edit step 7 in `SKILL.md` (it's literally a paragraph of English instructions; replace it with "Stop here. Tell the user what was created and let them commit when ready").

## Customizing

This skill is a single Markdown file (`SKILL.md`) that Claude reads as instructions. To change its behavior, edit the file. A few common tweaks:

- **Disable auto-commit:** see above.
- **Rename the URL convention** from `snapshot/vN` to `versions/vN` or `archive/vN`: do a find-and-replace on `snapshot/` in `SKILL.md`. The URL convention is the only thing the skill genuinely cares about.
- **Change what goes in `SUMMARY.md`:** edit `templates/summary.md` and the corresponding step in `SKILL.md`.

## Why this exists

I'm not a programmer, but I build little web things with Claude Code and I like to write about those projects. Reverse-engineering "what was this version like, and why?" from git history is painful. The *why* behind a design decision lives in the conversation, not in the diff. By the time I sit down to write the article, the *why* is gone.

This skill captures both: a working, clickable version of each iteration, plus a note from Claude written when the thinking was still in the room.

## License

MIT
