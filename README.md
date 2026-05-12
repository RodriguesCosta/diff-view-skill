# diff-view

> 🌎 [Português (pt-br)](README.pt-br.md)

A [Claude Code](https://claude.com/claude-code) skill that renders the current git repository's changes as a side-by-side HTML diff and opens it in a [Maestri](https://www.themaestri.app/) portal next to your terminal.

Powered by [diff2html-cli](https://github.com/rtfpessoa/diff2html-cli), run through [Bun](https://bun.com/) via `bunx --bun`.

## What it does

Instead of squinting at `git diff` in the terminal, ask Claude Code to show you a visual diff. The skill:

1. Picks the right diff scope (working tree vs HEAD by default, but honors "compare to main", "last commit", "staged only", etc.).
2. Pipes that into `bunx --bun diff2html-cli` to produce a self-contained side-by-side HTML page.
3. Includes **untracked files** too, by feeding `git diff --no-index /dev/null <file>` for each one — without mutating your index.
4. Strips the `diff2html` promo header via the native `-t "git diff"` flag.
5. Opens the result in a Maestri portal called **Diff** (reuses the portal on subsequent runs, with a cache-busting query string so the new content loads).

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
{
  git diff HEAD
  git ls-files --others --exclude-standard | while IFS= read -r f; do
    git diff --no-index --binary -- /dev/null "$f" || true
  done
} | bunx --bun diff2html-cli -i stdin -s side -t "git diff" -F /tmp/claude-diff.html

maestri portal create "file:///tmp/claude-diff.html" "Diff"
# or, if Diff portal already exists:
maestri portal navigate "Diff" "file:///tmp/claude-diff.html?$(date +%s)"
```

The cache-bust query string forces the portal to reload since the file path stays the same across runs.

## Maestri integration

The HTML is fully self-contained (CSS + JS inlined) and served via `file://`, so no local web server is needed. The skill keeps a single portal named **Diff** around and points it at the latest HTML on each run, so the diff view sits in a predictable spot on your canvas.

## License

MIT
