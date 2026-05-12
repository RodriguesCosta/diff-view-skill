---
name: diff-view
description: Generate a visual HTML diff of changes in the current git repository using `bunx --bun diff2html-cli` and open it in a Maestri portal browser for side-by-side review. Use this skill whenever the user wants to visually inspect git changes, see a diff in the browser, review what's about to be committed, compare branches or commits, or asks for "diff view", "visual diff", "ver alterações visualmente", "mostrar diff", "show me the changes", "open diff in browser", "abrir diff no maestri", or anything similar — even if they don't explicitly mention "diff2html". Prefer this over plain `git diff` output whenever the user wants something visual or shareable.
---

# Diff View

Generate a self-contained HTML diff of the current git repository's changes using `diff2html-cli` (run via `bunx --bun` to force the Bun runtime) and display it in a Maestri portal browser for side-by-side inspection.

## Prerequisites

- Current working directory is a git repository (`git rev-parse --is-inside-work-tree`).
- `bun` is installed and on PATH. We deliberately use Bun over Node — don't silently fall back to `npx` without asking.
- `maestri` CLI is available (used to open the result on the canvas).

## Workflow

### 1. Pick the diff scope

Default to **all uncommitted changes vs HEAD** (`git diff HEAD`) — that's the most common review scenario. If the user specifies otherwise, honor it:

| Intent | Git command |
|---|---|
| Default — uncommitted (staged + unstaged) vs HEAD | `git diff HEAD` |
| Staged only | `git diff --cached` |
| Unstaged only (working tree vs index) | `git diff` |
| Branch comparison | `git diff main..HEAD` (substitute branch names) |
| Last commit | `git show HEAD` |
| Specific range | `git diff <from>..<to>` |
| Specific commit | `git show <sha>` |

Quickly verify there's something to show before going further:

```bash
git diff HEAD --quiet && echo "no changes" || echo "has changes"
```

If empty, tell the user there's nothing to diff and stop — don't open an empty page.

### 2. Generate the HTML

Compose a single diff stream that includes both tracked changes (`git diff HEAD`) **and** every untracked file (rendered as a new-file diff via `git diff --no-index /dev/null <file>`), then pipe it through `diff2html-cli` via `bunx --bun`. The `--bun` flag forces the Bun runtime (rather than letting bunx fall back to Node):

```bash
{
  git diff HEAD
  git ls-files --others --exclude-standard | while IFS= read -r f; do
    git diff --no-index --binary -- /dev/null "$f" || true
  done
} | bunx --bun diff2html-cli -i stdin -s side -t "git diff" -F /tmp/claude-diff.html
```

Why the loop for untracked files: `git diff HEAD` ignores files git doesn't yet know about. Using `git diff --no-index /dev/null <file>` produces a standard "new file" unified diff for each untracked file **without mutating the index** (no `git add -N`, no cleanup needed). The `|| true` is required because `git diff --no-index` exits 1 whenever files differ — which is always for a new file vs `/dev/null`.

Flags worth knowing:
- `-i stdin` — read the diff from stdin (the piped git output)
- `-s side` — side-by-side layout (`line` for unified/inline if the user prefers it)
- `-t "git diff"` — page title. diff2html-cli's `--title` overrides **both** the `<title>` element and the visible `<h1>` page header, replacing the default `"Diff to HTML by rtfpessoa"` promo header. Pick any title that fits — `"git diff"` is a clean default.
- `-F /tmp/claude-diff.html` — write a standalone HTML file to this path. With `-F` set, diff2html-cli won't open its own preview browser, which is what we want.

The HTML is fully self-contained (CSS + JS inlined), so `file://` works without running a local server.

### 3. Open in a Maestri portal

Check existing portals first to avoid cluttering the canvas with duplicates:

```bash
maestri list
```

If a portal named **Diff** already exists, reuse it. Append a cache-busting query string so the portal definitely reloads the new file content (the file path stays the same across runs and browsers may serve the cached version otherwise):

```bash
maestri portal navigate "Diff" "file:///tmp/claude-diff.html?$(date +%s)"
```

Otherwise create a new one (it'll be placed next to the terminal automatically):

```bash
maestri portal create "file:///tmp/claude-diff.html" "Diff"
```

### 4. Summarize what's open

Tell the user briefly what they're looking at — e.g., "Opened diff in the 'Diff' portal: 4 files changed (+87/-23)". Useful one-liners that match the scope you used:

```bash
git diff HEAD --shortstat   # files changed, insertions, deletions
git diff HEAD --stat        # per-file breakdown
```

## Notes

- `/tmp/claude-diff.html` is overwritten on each run — that's intentional; it's a temporary artifact.
- The first invocation of `bunx --bun diff2html-cli` may take a few seconds while Bun fetches the package; subsequent runs are fast (cached).
- If `bun` is not installed, surface the error and ask the user before falling back to `npx` — we chose Bun on purpose.
- If `diff2html-cli` writes anything to stderr, surface it rather than swallowing it silently.
- The portal name "Diff" is a convention so subsequent runs reuse the same portal node. If the user wants a fresh one (e.g., to compare two diffs side by side), they can ask for it.
