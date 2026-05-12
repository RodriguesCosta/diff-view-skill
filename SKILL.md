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

### 2. Compute per-project identifiers

Before generating anything, derive a stable-per-project filename and portal name so multiple projects open in parallel **without clobbering each other's diffs**. The repo's directory basename gives a readable label; a short hash of the full absolute path disambiguates two repos that happen to share a basename:

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
REPO_NAME=$(basename "$REPO_ROOT")
REPO_HASH=$(printf '%s' "$REPO_ROOT" | shasum | cut -c1-6)
DIFF_FILE="/tmp/claude-diff-${REPO_NAME}-${REPO_HASH}.html"
PORTAL_NAME="Diff: ${REPO_NAME}"
```

Reruns on the same repo overwrite the same file and reuse the same portal — exactly what you want. Two different repos get distinct files and portals.

### 3. Generate the HTML

Compose a single diff stream that includes both tracked changes (`git diff HEAD`) **and** every untracked file (rendered as a new-file diff via `git diff --no-index /dev/null <file>`), then pipe it through `diff2html-cli` via `bunx --bun`. The `--bun` flag forces the Bun runtime (rather than letting bunx fall back to Node):

```bash
{
  git diff HEAD
  git ls-files --others --exclude-standard | while IFS= read -r f; do
    git diff --no-index --binary -- /dev/null "$f" || true
  done
} | bunx --bun diff2html-cli -i stdin -s side -t "git diff: $REPO_NAME" -F "$DIFF_FILE"
```

Why the loop for untracked files: `git diff HEAD` ignores files git doesn't yet know about. Using `git diff --no-index /dev/null <file>` produces a standard "new file" unified diff for each untracked file **without mutating the index** (no `git add -N`, no cleanup needed). The `|| true` is required because `git diff --no-index` exits 1 whenever files differ — which is always for a new file vs `/dev/null`.

Flags worth knowing:
- `-i stdin` — read the diff from stdin (the piped git output)
- `-s side` — side-by-side layout (`line` for unified/inline if the user prefers it)
- `-t "git diff: $REPO_NAME"` — page title. diff2html-cli's `--title` overrides **both** the `<title>` element and the visible `<h1>` page header, replacing the default `"Diff to HTML by rtfpessoa"` promo header. Including the repo name makes the portal tab self-identifying when multiple are open.
- `-F "$DIFF_FILE"` — write a standalone HTML file to this path. With `-F` set, diff2html-cli won't open its own preview browser, which is what we want.

The HTML is fully self-contained (CSS + JS inlined), so `file://` works without running a local server.

### 4. Open in a Maestri portal

Create the portal if it doesn't exist yet for this project, otherwise navigate the existing one to the new file. Append a cache-busting query string on reuse so the portal definitely reloads (the file path is stable, so the portal would otherwise serve a cached version):

```bash
if maestri list 2>&1 | grep -Fq "name: \"$PORTAL_NAME\""; then
  maestri portal navigate "$PORTAL_NAME" "file://${DIFF_FILE}?$(date +%s)"
else
  maestri portal create "file://${DIFF_FILE}" "$PORTAL_NAME"
fi
```

### 5. Summarize what's open

Tell the user briefly what they're looking at — e.g., "Opened diff in the 'Diff: baas-mobile' portal: 4 files changed (+87/-23)". Useful one-liners that match the scope you used:

```bash
git diff HEAD --shortstat   # files changed, insertions, deletions
git diff HEAD --stat        # per-file breakdown
```

## Notes

- The HTML file at `/tmp/claude-diff-${REPO_NAME}-${REPO_HASH}.html` is overwritten on each run for that repo — that's intentional; it's a per-project temporary artifact. Different repos get different files, so multiple projects can have their diffs open at the same time.
- The portal name is `Diff: <repo-basename>` so you can spot at a glance which project a portal belongs to on the canvas.
- The first invocation of `bunx --bun diff2html-cli` may take a few seconds while Bun fetches the package; subsequent runs are fast (cached).
- If `bun` is not installed, surface the error and ask the user before falling back to `npx` — we chose Bun on purpose.
- Untracked lockfiles (`bun.lock`, `package-lock.json`, etc.) can blow up the HTML size significantly because they're rendered as full new-file diffs. If the user complains about size or slowness, they can either commit/ignore the lockfile or ask the skill to skip lockfiles for that run.
- If `diff2html-cli` writes anything to stderr, surface it rather than swallowing it silently.
