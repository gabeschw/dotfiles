# dotfiles

Personal dotfiles and coding-agent configuration, managed with [GNU Stow](https://www.gnu.org/software/stow/).

## Layout

Each top-level folder is a Stow **package** whose internal structure mirrors `$HOME`.
Stow symlinks each file to the matching path under your home directory.

```
dotfiles/
├── opencode/   # → ~/.config/opencode/  (AGENTS.md, opencode.jsonc, agent/, skills/)
└── claude/     # → ~/.claude/           (CLAUDE.md, settings.json)
```

`AGENTS.md` is the single source of truth for agent instructions; `~/.claude/CLAUDE.md`
just `@`-includes it, so Claude Code and opencode share one set of preferences.

## Setup on a new machine

```sh
brew install stow
git clone <repo-url> ~/Projects/repos/dotfiles
cd ~/Projects/repos/dotfiles
stow -t ~ opencode claude
```

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

## Adding a file

Put it inside the right package at the path it should have under `$HOME`, then restow:

```sh
# e.g. track ~/.gitconfig
mkdir -p git
mv ~/.gitconfig git/.gitconfig
stow -t ~ git
```

## Not tracked

opencode's node runtime (`node_modules/`, `package.json`, `bun.lock`) is intentionally
left in place under `~/.config/opencode/` and not version-controlled here.
