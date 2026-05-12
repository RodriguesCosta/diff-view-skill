# diff-view

> 🌎 [Português (pt-br)](README.pt-br.md)

A [Claude Code](https://claude.com/claude-code) skill that renders the current git repository's changes as a side-by-side HTML diff and opens it in a [Maestri](https://www.themaestri.app/) portal next to your terminal.

Powered by [diff2html-cli](https://github.com/rtfpessoa/diff2html-cli), run through [Bun](https://bun.com/) via `bunx --bun`.

## What it does

Instead of squinting at `git diff` in the terminal, ask Claude Code to show you a visual diff. The skill:

1. Picks the right diff scope (working tree vs HEAD by default, but honors "compare to main", "last commit", "staged only", etc.).
2. Pipes that into `bunx --bun diff2html-cli` to produce a self-contained side-by-side HTML page.
3. Includes **untracked files** too, by feeding `git diff --no-index /dev/null <file>` for each one — without mutating your index.
4. **Filters out lockfiles by default** on both tracked and untracked sides (`bun.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Podfile.lock`, `Cargo.lock`, `go.sum`, etc.) so the HTML doesn't blow up to tens of MB for a regular review. Tracked files are filtered via git pathspecs (`:(exclude,glob)**/<lockfile>`); untracked files by name. Ask explicitly to "include lockfiles" if you actually want to see them.
5. Strips the `diff2html` promo header via the native `-t` flag (the title becomes `git diff: <repo-name>`).
6. Opens the result in a **per-project** Maestri portal called `Diff: <repo-basename>`. The HTML file and portal name are both keyed off the repo's basename + a short hash of its absolute path, so working on multiple projects at once doesn't clobber any of the open diffs — each project gets its own portal on the canvas.

## Installation

Clone into your user-level skills folder:

```bash
git clone git@github.com:RodriguesCosta/diff-view-skill.git ~/.claude/skills/diff-view
```

Restart Claude Code (or reload skills) and the skill becomes available.

### Prerequisites

- `git` (you already have it).
- [`bun`](https://bun.com/) on your PATH — the skill uses `bunx --bun` to run `diff2html-cli` under the Bun runtime. First run fetches the package; subsequent runs are cached.
- [`maestri`](https://www.themaestri.app/) CLI for opening the result on the canvas. Without it the HTML is still generated at `/tmp/claude-diff.html` and you can open it manually.

## Usage

Just ask. The skill auto-triggers on phrases like:

- "show me the diff" / "visual diff" / "diff view"
- "open the diff in the browser" / "abrir o diff no maestri"
- "review my changes" / "ver alterações visualmente"
- "compare main to HEAD" / "diff between branches"

By default it shows all uncommitted changes (staged + unstaged + untracked) vs `HEAD`. You can specify any scope: "diff between main and feature/x", "show me the last commit", "staged only", etc.

## How it works

The core pipeline is roughly:

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
REPO_NAME=$(basename "$REPO_ROOT")
REPO_HASH=$(printf '%s' "$REPO_ROOT" | shasum | cut -c1-6)
DIFF_FILE="/tmp/claude-diff-${REPO_NAME}-${REPO_HASH}.html"
PORTAL_NAME="Diff: ${REPO_NAME}"

LOCKFILE_RE='(^|/)(bun\.lock|bun\.lockb|package-lock\.json|npm-shrinkwrap\.json|yarn\.lock|pnpm-lock\.yaml|Cargo\.lock|Gemfile\.lock|composer\.lock|Pipfile\.lock|poetry\.lock|uv\.lock|go\.sum|mix\.lock|Podfile\.lock)$'
EXCLUDE_LOCKFILES=()
for f in bun.lock bun.lockb package-lock.json npm-shrinkwrap.json yarn.lock \
         pnpm-lock.yaml Cargo.lock Gemfile.lock composer.lock Pipfile.lock \
         poetry.lock uv.lock go.sum mix.lock Podfile.lock; do
  EXCLUDE_LOCKFILES+=(":(exclude,glob)**/$f")
done

{
  git diff HEAD -- . "${EXCLUDE_LOCKFILES[@]}"
  git ls-files --others --exclude-standard | grep -Ev "$LOCKFILE_RE" | while IFS= read -r f; do
    git diff --no-index --binary -- /dev/null "$f" || true
  done
} | bunx --bun diff2html-cli -i stdin -s side -t "git diff: $REPO_NAME" -F "$DIFF_FILE"

if maestri list 2>&1 | grep -Fq "name: \"$PORTAL_NAME\""; then
  maestri portal navigate "$PORTAL_NAME" "file://${DIFF_FILE}?$(date +%s)"
else
  maestri portal create "file://${DIFF_FILE}" "$PORTAL_NAME"
fi
```

The cache-bust query string forces the portal to reload since the file path stays the same across runs of the same repo. The hash on the file name prevents two repos that happen to share a basename from overwriting each other's HTML.

## Maestri integration

The HTML is fully self-contained (CSS + JS inlined) and served via `file://`, so no local web server is needed. Each project gets its own portal — `Diff: <repo-basename>` — so you can keep diffs open side by side while jumping between projects.

## License

MIT
