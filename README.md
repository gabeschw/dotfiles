# dotfiles

Personal dotfiles and coding-agent configuration, managed with [GNU Stow](https://www.gnu.org/software/stow/).

Each top-level folder is a Stow **package** whose internal structure mirrors `$HOME`; Stow symlinks each file into place under your home directory.

```
dotfiles/
├── opencode/   # → ~/.config/opencode/  (AGENTS.md, opencode.jsonc, agent/, skills/)
├── claude/     # → ~/.claude/           (CLAUDE.md, settings.json)
├── zsh/        # → ~/.zshrc
├── brew/       # → ~/Brewfile
├── ghostty/    # → ~/.config/ghostty/config
├── zed/        # → ~/.config/zed/settings.json
└── btop/       # → ~/.config/btop/
```

`AGENTS.md` is the single source of truth for agent instructions; `~/.claude/CLAUDE.md` just `@`-includes it, so Claude Code and opencode share one set of preferences.

> **The `-t ~` flag is required** on every Stow command. This repo doesn't live under `$HOME`, so without `--target` Stow would target the repo's parent directory.

## Setup on a new machine

```sh
brew install stow
git clone <repo-url> ~/Projects/repos/dotfiles
cd ~/Projects/repos/dotfiles
stow -t ~ opencode claude zsh brew ghostty zed btop   # or: stow -t ~ */
```

External dependencies these configs assume (install separately):

- **oh-my-zsh** (and the `gis` theme) — `.zshrc` sources it

## Stow commands

Run from the repo root:

| Command | Action |
|---|---|
| `stow -n -v -t ~ <pkg>` | Dry run — show what would happen, change nothing |
| `stow -t ~ <pkg>`       | Create symlinks for a package |
| `stow -R -t ~ <pkg>`    | Restow — pick up files added/renamed in the package |
| `stow -D -t ~ <pkg>`    | Remove a package's symlinks |

## Adding files

Stow only symlinks **files**; it creates intermediate directories as real directories and leaves existing ones (e.g. `~/.oh-my-zsh/`) untouched. To track a new file, move it into the matching path inside its package, then **restow** so Stow picks it up:

```sh
# track ~/.oh-my-zsh/custom/themes/gis.zsh-theme
mkdir -p zsh/.oh-my-zsh/custom/themes
mv ~/.oh-my-zsh/custom/themes/gis.zsh-theme zsh/.oh-my-zsh/custom/themes/
stow -R -t ~ zsh
```

The same works for directory trees. After stowing, editing the repo file and editing its symlinked location are identical; they're the same file.

### Folded vs. unfolded directories

How Stow links a package's directory depends on whether the target dir already exists when you stow:

- **Folded** — target dir is absent → Stow symlinks the *whole directory* (e.g. `~/.config/opencode/skills -> .../dotfiles/opencode/.config/opencode/skills`). Any file later written there lands **inside the repo automatically**, already tracked.
- **Unfolded** — target dir already exists (or you pass `--no-folding`, or two packages share it) → Stow makes a real directory and symlinks each file individually. New files written there are real files *outside* the repo until claimed.

To convert: unfold with `stow --no-folding -R -t ~ <pkg>`; fold by emptying the target (`stow -D`, then `rm -rf` the now-empty dir) and re-stowing. Dry-run with `stow -n -v` first.

Fold dirs where everything belongs in the repo (config-only). Keep dirs unfolded when they mix tracked config with untracked runtime — `opencode` stays unfolded so `node_modules/` isn't captured, while its `agent/` and `skills/` subdirs fold on their own and self-track.

### Claiming files written into an unfolded directory

In an unfolded target dir, files a tool writes are real files living outside the repo. `bin/get-new-dotfiles` finds these untracked files, moves them into the repo, and restows:

```sh
bin/get-new-dotfiles --dry-run   # preview what would be claimed
bin/get-new-dotfiles             # move into repo + restow
```

It scans the packages with directory targets (`opencode`, `claude`, `btop`), skipping the node runtime and other non-tracked files. Folded subdirs need no claiming (they self-track), and the script's `find` doesn't descend into them anyway. When you add a package with a directory target, add it to the `PACKAGES`/`TARGETS` arrays in the script.

## Brewfile

`brew/Brewfile` tracks Homebrew packages, casks, Go tools, and npm packages, managed by [Homebrew Bundle](https://github.com/Homebrew/homebrew-bundle) and symlinked to `~/Brewfile`.

| Command | Action |
|---|---|
| `brew bundle --file ~/Brewfile` | Install everything in the file (additive — never deletes) |
| `brew bundle check --file ~/Brewfile` | Show what's missing or needs updating |
| `brew bundle dump --force --no-vscode --file ~/Brewfile` | Regenerate from what's actually installed |

After a `dump`, review with `git diff` and commit.

## Removing a package (keeping its config on another machine)

When you delete a package on machine A but want machine B to keep that config as ordinary (untracked) files, **freeze the symlinks into real files before pulling** — otherwise the pull deletes the repo files and B's symlinks dangle. On machine B, before `git pull`:

```sh
stow -D -t ~ <pkg>                 # remove symlinks while repo files still exist
cp -R <pkg>/<path> ~/<path>        # copy the content back as real files
git pull --ff-only                 # now the repo deletion can't orphan anything
```

Doing this dance often is a signal the config probably shouldn't have been removed — or that those tools belong in a per-host arrangement (e.g. `Brewfile.common` + `Brewfile.<host>`) rather than fully dropped.

## Not tracked

opencode's node runtime (`node_modules/`, `package.json`, `bun.lock`) is intentionally left in place under `~/.config/opencode/` and not version-controlled here.
