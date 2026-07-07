# dotfiles

Personal dotfiles and coding-agent configuration, managed with [GNU Stow](https://www.gnu.org/software/stow/).

Each top-level folder is a Stow **package** whose internal structure mirrors `$HOME`; Stow symlinks each file into place under your home directory.

```
dotfiles/
├── opencode/   # → ~/.config/opencode/  (AGENTS.md, opencode.jsonc, agent/)
├── agents/     # → ~/.agents/            (.skill-lock.json)
├── claude/     # → ~/.claude/           (CLAUDE.md, settings.json; skills/ bridged by bin/restore-skills)
├── zsh/        # → ~/.zshrc
├── brew/       # → ~/Brewfile
├── ghostty/    # → ~/.config/ghostty/config
├── zed/        # → ~/.config/zed/settings.json
├── btop/       # → ~/.config/btop/
└── linearmouse/ # → ~/.config/linearmouse/
```

`AGENTS.md` is the single source of truth for agent instructions; `~/.claude/CLAUDE.md` just `@`-includes it, so Claude Code and opencode share one set of preferences.

> **The `-t ~` flag is required** on every Stow command. This repo doesn't live under `$HOME`, so without `--target` Stow would target the repo's parent directory.

## Setup on a new machine

```sh
brew install stow
git clone <repo-url> ~/Projects/repos/dotfiles
cd ~/Projects/repos/dotfiles
stow -t ~ opencode zsh brew ghostty zed btop linearmouse
stow -t ~ --no-folding agents   # keep ~/.agents a real dir so npx-installed skills stay outside the repo
stow -t ~ claude                # .stow-no-folding marker prevents directory folding
bin/restore-skills              # installs skills + bridges ~/.claude/skills -> ~/.agents/skills
```

`agents` is stowed with `--no-folding` on purpose: on a fresh machine `~/.agents` doesn't exist, so a plain `stow` would fold the whole directory into the repo and `npx` would then write installed skills *inside* the repo. `--no-folding` forces a real `~/.agents/` with only `.skill-lock.json` symlinked.

`claude` contains a `.stow-no-folding` marker file for the same reason: without it, Stow would fold `~/.claude` into a single directory symlink and Claude CLI's auto-generated files (history, cache, daemon logs, etc.) would land in the repo. The marker forces individual file symlinks so only `CLAUDE.md` and `settings.json` are tracked.

(Don't use the `stow -t ~ */` shorthand for the same reason.)

External dependencies these configs assume (install separately):

- **oh-my-zsh** (and the `gis` theme) — `.zshrc` sources it
- **Node.js** — `bin/restore-skills` uses `npx`

### Updating an existing machine

After pulling, just run all of these every time. Re-running a step when nothing changed is safe — it simply does nothing.

```sh
git pull --ff-only
stow -R -t ~ opencode zsh brew ghostty zed btop linearmouse
stow -R -t ~ --no-folding agents
stow -R -t ~ claude
brew bundle --file ~/Brewfile   # installs any new packages
bin/restore-skills              # installs any new skills + ensures the bridge link
```

`stow -R` re-syncs each package's symlinks to match the repo; `brew bundle` and `bin/restore-skills` install whatever the pull added.

### Agent skills

Skills are managed by [`npx skills`](https://github.com/vercel-labs/skills) and installed globally into `~/.agents/skills/`. The lockfile at `agents/.agents/.skill-lock.json` is the source of truth (symlinked to `~/.agents/.skill-lock.json` via Stow). `~/.claude/skills` is a symlink to `~/.agents/skills/`, so opencode and Claude Code share the same skill set.

That `~/.claude/skills` symlink is **not** Stow-managed: a relative symlink inside a Stow package resolves relative to the repo, not `$HOME`, so it can never reach the `npx`-managed `~/.agents/skills`. `bin/restore-skills` creates it directly instead.

After `stow`, restore skills on a new machine:

```sh
bin/restore-skills
```

This first links `~/.claude/skills -> ../.agents/skills`, then reads the lockfile and re-runs `npx skills add -g` for every entry, preserving the lockfile unchanged. Add or update skills as usual with `npx skills add <source> -a opencode -g`; the lockfile updates through the symlink and can be committed.

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

- **Folded** — target dir is absent → Stow symlinks the *whole directory* (e.g. `~/.config/zed -> .../dotfiles/zed/.config/zed`). Any file later written there lands **inside the repo automatically**, already tracked.
- **Unfolded** — target dir already exists (or you pass `--no-folding`, or two packages share it) → Stow makes a real directory and symlinks each file individually. New files written there are real files *outside* the repo until claimed.

To convert: unfold with `stow --no-folding -R -t ~ <pkg>`; fold by emptying the target (`stow -D`, then `rm -rf` the now-empty dir) and re-stowing. Dry-run with `stow -n -v` first.

Fold dirs where everything belongs in the repo (config-only). Keep dirs unfolded when they mix tracked config with untracked runtime — `opencode` stays unfolded so `node_modules/` isn't captured, while its `agent/` subdir folds on its own and self-tracks. `agents` is likewise kept unfolded (stow it with `--no-folding`) so the `npx`-installed `~/.agents/skills/` stays outside the repo; only `.skill-lock.json` is tracked.

In an unfolded target dir, files a tool writes are real files living outside the repo. To track one, move it into the matching path inside its package and **restow** (see [Adding files](#adding-files) above) — same `mv` + `stow -R` flow.

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
