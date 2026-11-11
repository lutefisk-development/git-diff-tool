# gdt — git diff tool

A small, read-only visual diff viewer for git, inspired by the changes view in GitHub Desktop.

Run `gdt` inside any git repository and a window opens showing every changed file on the left and a
colored, syntax-highlighted diff on the right. It updates live as you edit files.

It only *shows* changes — it never stages, commits, discards or modifies anything.

## Features

- File list with status icons: modified, added, deleted, renamed (`old → new`)
- Unified diff with old/new line numbers, hunk headers and syntax highlighting
- Intra-line highlighting of the exact characters that changed
- `+added −removed` totals per file and overall
- Live updates: the window refreshes about a second after a file is saved, created, deleted, staged or committed
- View a single commit or a range of revisions
- **History** tab: browse the branch's commits (author, co-authors, age, `↑` for not-yet-pushed) and click one
  to see its changed files and diff
- Light and dark theme (follows the OS)
- Keyboard navigation and a resizable sidebar
- No dependencies beyond Python 3 and git

## Requirements

- **Python 3.7+** (macOS ships with one; check with `python3 --version`)
- **git**
- **Google Chrome**, Brave, Microsoft Edge or Chromium (recommended): the viewer opens as a standalone app window
  without tabs or an address bar. Without one of these, it opens in your default browser.

No Docker, no `pip install`, no build step.

## Installation

```sh
git clone https://github.com/lutefisk-development/git-diff-tool.git ~/dev/git-diff-tool
chmod +x ~/dev/git-diff-tool/gdt
ln -s ~/dev/git-diff-tool/gdt ~/.local/bin/gdt
```

`~/.local/bin` must be on your `PATH`. If it isn't, add this to `~/.zshrc` (or `~/.bashrc`):

```sh
export PATH="$HOME/.local/bin:$PATH"
```

Any other directory on your `PATH` works too, such as `/usr/local/bin`. Keep it a **symlink**: `gdt` finds
`template.html` by following the link back to the repository.

### oh-my-zsh users

The oh-my-zsh `git` plugin defines an alias `gdt` (`git diff-tree ...`). An alias always wins over a command
on your `PATH`, so add this line at the **end** of `~/.zshrc`, after oh-my-zsh is loaded:

```sh
unalias gdt 2>/dev/null
```

Then open a new terminal. Check with `whence -v gdt` (zsh) or `type gdt`; it should point to the file and not
an alias. Alternatively, link it under another name, e.g. `ln -s ~/dev/git-diff-tool/gdt ~/.local/bin/gdiff`.

## Usage

Run it from anywhere inside a repository, including subdirectories:

```sh
gdt                  # all uncommitted changes: staged + unstaged + untracked files, vs HEAD
gdt <commit>         # what a commit changed (vs its first parent), e.g. gdt HEAD or gdt b6bf7b1
gdt <a>..<b>         # changes between two revisions, e.g. gdt main..HEAD
gdt --help
```

`gdt` returns your terminal right away; the window runs on its own.

### History

Click **History** (top right) to see the commits on the current branch (`git log HEAD`), newest first, the
same list GitHub Desktop shows. Selecting a commit shows its changed files in the middle column and the diff on the
right. Merge commits are shown against their first parent. More commits load as you scroll. A `↑` marks commits
that aren't on the upstream branch yet. In live mode the list refreshes when `HEAD` moves (commit, amend, checkout,
pull).

### Keyboard shortcuts

| Key | Action |
| --- | --- |
| `↓` / `j` | Next file |
| `↑` / `k` | Previous file |
| `w` | Toggle line wrapping |
| `Ctrl+1` / `Ctrl+2` | Changes / History tab |
| `←` / `→` (or `h` / `l`) | History: move between the commit list and the file list |

### Closing

Close the window as usual: the red button or `⌘W`. The background process exits by itself about
30 seconds later. If the browser was started only for gdt, it may keep running in the Dock afterwards (`⌘Q`
quits it).

### Options

| Option | Description |
| --- | --- |
| `--no-open` | Print the URL instead of opening a window |
| `--foreground` | Don't detach; runs in the terminal until `Ctrl+C` (useful for debugging) |

| Environment variable | Description |
| --- | --- |
| `GDT_BROWSER=default` | Open in the default browser instead of a Chromium app window |

## How it works

```
gdt ──► git diff / git ls-files ──► JSON ──► local HTTP server (127.0.0.1, random port)
                                                   │
             app window (template.html) ◄── fetch ─┤
                                         ◄── SSE ──┘  "something changed, refetch"
```

1. `gdt` finds the repository root and starts a tiny HTTP server on `127.0.0.1` with a random free port.
   It then forks into the background and opens the page in a browser app window.
2. The page fetches the diff as JSON. In working-tree mode, the diff is `git diff HEAD` (staged and
   unstaged changes together) plus untracked files that aren't ignored by `.gitignore`.
3. **Live updates:** once a second, the server computes a cheap fingerprint of the working tree: the `HEAD` commit,
   `git status --porcelain` and the modification time and size of each changed file. When it changes, the
   server pushes an event over Server-Sent Events and the page refetches, keeping the selected file and
   scroll position. Commit and range views are static and don't poll.
4. When no window has been connected for 30 seconds, the server shuts itself down.

The project has two files:

- `gdt`: Python script (standard library only) that collects and parses diffs and serves the page
- `template.html`: the viewer (HTML/CSS/JS, no build step)

## Good to know

- **Read-only.** gdt never writes to your repository. Status polling uses `git --no-optional-locks`,
  so it doesn't take `index.lock` and won't collide with git commands you run at the same time.
- **Local only.** The server listens on `127.0.0.1` only and rejects requests with an unexpected `Host` header.
- **Syntax highlighting** uses [highlight.js](https://highlightjs.org/), loaded from cdnjs. Offline, everything
  still works, just without colors. Highlighting is turned off for very large diffs (over 20,000 lines in a file).
- **Binary files** are listed but not displayed. Untracked files larger than 2 MB are listed with a note.
- **Merge commits:** `gdt <merge-commit>` shows the diff against the first parent.
- **Large repositories:** in live mode, `git status` runs once a second while the window is open. This is
  usually fast. If it's slow in a very large repo, increase the interval in `watch()` in `gdt`.
- Git repositories only (no SVN or Mercurial).
- Tested on macOS. Linux should work (`google-chrome`/`chromium` app mode, falling back to `xdg-open`). Windows
  isn't supported, because the script uses `fork`.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| `gdt` runs `git diff-tree` | The oh-my-zsh alias is active; see [oh-my-zsh users](#oh-my-zsh-users) |
| `command not found: gdt` | The symlink's directory isn't on `PATH`, or the script isn't executable (`chmod +x gdt`) |
| `gdt: not inside a git repository` | Run it inside a git working tree |
| Window shows **● Disconnected** | The server stopped; run `gdt` again |
| Opens in a normal browser tab | No Chromium-based browser was found in `/Applications`; that's the fallback |
| No syntax colors | No internet connection (highlight.js is loaded from a CDN) |
