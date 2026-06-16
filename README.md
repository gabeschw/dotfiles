# dotfiles

Personal dotfiles and coding-agent configuration, managed with [GNU Stow](https://www.gnu.org/software/stow/).

## Layout

Each top-level folder is a Stow **package** whose internal structure mirrors `$HOME`.
Stow symlinks each file to the matching path under your home directory.

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

`AGENTS.md` is the single source of truth for agent instructions; `~/.claude/CLAUDE.md`
just `@`-includes it, so Claude Code and opencode share one set of preferences.

To stow everything at once: `stow -t ~ */` from the repo root (list package names
explicitly if your shell doesn't expand the glob).

## Setup on a new machine

```sh
brew install stow
git clone <repo-url> ~/Projects/repos/dotfiles
cd ~/Projects/repos/dotfiles
stow -t ~ opencode claude zsh brew ghostty zed btop
```

## Brewfile

`brew/Brewfile` tracks the Homebrew packages, casks, Go tools, and npm packages you want installed. It's managed by [Homebrew Bundle](https://github.com/Homebrew/homebrew-bundle) and is symlinked to `~/Brewfile`.

| Command | Action |
|---|---|
| `brew bundle --file ~/Brewfile` | Install everything in the file (additive — never deletes) |
| `brew bundle check --file ~/Brewfile` | Show what's missing or needs updating |
| `brew bundle dump --force --no-vscode --file ~/Brewfile` | Regenerate from what's actually installed |

After making changes via `brew bundle dump`, review with `git diff` and commit.

External dependencies these configs assume (install separately):

- **oh-my-zsh** (and the `gis` theme) — `.zshrc` sources it

> **Note:** this repo does not live under `$HOME`, so the `-t ~` (`--target`) flag is
> required — without it Stow targets the repo's parent directory.

## Common commands

Run from the repo root:

| Command | Action |
|---|---|
| `stow -n -v -t ~ <pkg>` | Dry run — show what would happen, change nothing |
| `stow -t ~ <pkg>`       | Create symlinks for a package |
| `stow -R -t ~ <pkg>`    | Restow (after adding/renaming files in the package) |
| `stow -D -t ~ <pkg>`    | Remove a package's symlinks |

## Adding files and directories

Stow only symlinks **files** — intermediate directories under `$HOME` are created as
real directories as needed. If the target directory already exists (e.g. `~/.oh-my-zsh/`),
Stow won't interfere with it; it just adds symlinks for the specific files inside it.

### Adding a single file

```sh
# e.g. track ~/.someconfig
mkdir -p mypkg
mv ~/.someconfig mypkg/.someconfig
stow -t ~ mypkg          # first stow of this package
```

### Adding files to an already-stowed package

Use `stow -R` (restow) so Stow picks up the new files:

```sh
# e.g. track ~/.oh-my-zsh/custom/themes/gis.zsh-theme
mkdir -p zsh/.oh-my-zsh/custom/themes
mv ~/.oh-my-zsh/custom/themes/gis.zsh-theme zsh/.oh-my-zsh/custom/themes/gis.zsh-theme
stow -R -t ~ zsh
```

### Adding a directory tree (e.g. a new agent skill)

Create the full path inside the package directory and place the files there, then
restow. No special handling needed — Stow walks the tree and symlinks each file:

```sh
mkdir -p opencode/.config/opencode/skills/my-skill
# create opencode/.config/opencode/skills/my-skill/SKILL.md
# create opencode/.config/opencode/skills/my-skill/references/...
stow -R -t ~ opencode
```

Editing the repo file or its symlinked original location is the same — they're the
same file once stowed.

## Removing a package (and keeping its config on another machine)

When you delete a package on machine A but want machine B to keep that config as
ordinary (untracked) files, you have to **freeze the symlinks into real files
before pulling** — otherwise the pull deletes the repo files and B's symlinks
dangle. On machine B, before `git pull`:

```sh
stow -D -t ~ <pkg>                 # remove the symlinks while repo files still exist
cp -R <pkg>/<path> ~/<path>        # copy the content back as real files
git pull --ff-only                 # now the repo deletion can't orphan anything
```

Heads-up: if you find yourself doing this "freeze before pull" dance often, that's
a signal the config probably shouldn't have been removed from the repo in the first
place — or that those tools belong in a per-host arrangement (e.g. `Brewfile.common`
+ `Brewfile.<host>`) rather than fully dropped.

## Not tracked

opencode's node runtime (`node_modules/`, `package.json`, `bun.lock`) is intentionally
left in place under `~/.config/opencode/` and not version-controlled here.
