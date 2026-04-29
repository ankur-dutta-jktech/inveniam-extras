# inveniam-extras

Personal/dev side repo for `inveniam.io`. Tracks files that don't belong in the main repo — local IDE config, Claude/Cursor settings, custom Makefiles, per-developer `docker-compose.yml` overrides, etc.

Uses the **dotfiles pattern**: a bare git directory lives outside the project, with the work tree pointed at the project root. Files coexist with the main repo's working tree but are tracked by a separate git dir.

> This README lives at `.github/README.md` (rather than the repo root) because the work tree's root `README.md` is owned by the main `inveniam.io` repo. GitHub renders `.github/README.md` as the repo README when no root README is tracked.

## What's tracked

- `.cursor/` — Cursor IDE rules and skills
- `.claude/` — Claude Code settings, agents, skills
- `Makefile` — local task runner
- `apps/ai-service/containers/*/docker-compose.yml` — local-dev compose overrides
- `apps/ai-service/containers/.editorconfig` — personal Python editor rules (overrides not in main repo's root `.editorconfig`)
- `.github/README.md` — this file

(Run `inv ls-files` for the authoritative list.)

## First-time setup on a new machine

Assumes the main `inveniam.io` repo is already cloned at `~/playground/inveniam.io`.

```bash
# 1. Clone the side repo as bare
git clone --bare git@github.com:ankur-dutta-jktech/inveniam-extras.git ~/.inveniam-extras.git

# 2. Add the wrapper alias to ~/.zshrc (or ~/.bashrc)
echo "alias inv='git -C \$HOME/playground/inveniam.io --git-dir=\$HOME/.inveniam-extras.git --work-tree=\$HOME/playground/inveniam.io'" >> ~/.zshrc
source ~/.zshrc

# 3. Quiet untracked-file noise (only show files this repo tracks)
inv config status.showUntrackedFiles no

# 4. Check out the tracked files into the work tree
inv checkout main
# If any tracked file conflicts with an existing local file, back it up and re-run:
#   mv .cursor .cursor.bak && inv checkout main

# 5. Tell the main repo to ignore this README locally (it lives under .github/, which the main repo tracks)
echo ".github/README.md" >> ~/playground/inveniam.io/.git/info/exclude
```

## Setting up an additional worktree

If you use `git worktree add` on the main repo and want side-repo tracking in that worktree too, give each worktree its own bare git dir:

```bash
WORKTREE=~/playground/inveniam.io-feature-x

# 1. Clone a separate bare for this worktree
git clone --bare git@github.com:ankur-dutta-jktech/inveniam-extras.git ~/.inveniam-extras-feature-x.git

# 2. Add a per-worktree alias
echo "alias inv-fx='git -C $WORKTREE --git-dir=\$HOME/.inveniam-extras-feature-x.git --work-tree=$WORKTREE'" >> ~/.zshrc
source ~/.zshrc

inv-fx config status.showUntrackedFiles no
inv-fx checkout main
echo ".github/README.md" >> $WORKTREE/.git/info/exclude
```

Each worktree has independent side-repo state (you can be on different branches per worktree). Pull/push as needed to sync them through the GitHub remote.

## Daily usage

```bash
inv status              # see modified/staged side-repo files
inv add <path>          # stage a tracked file
inv add -f <path>       # force-add a path that's in the main repo's .gitignore
inv commit -m "..."     # commit
inv push                # push to GitHub
inv pull                # pull updates (after editing on another machine)
inv log --oneline       # history
```

The alias has `-C` baked in, so commands work from any subdirectory of the project.

## Pitfalls

- **Never run `inv add .`** — it will try to stage the entire project tree. Always pass explicit paths.
- The side repo reads the main repo's `.gitignore`. To add an ignored path, use `inv add -f <path>`. After the first commit, future edits don't need `-f`.
- Don't put secrets here (`.env*`, credentials). Treat it as code, not a vault.
- The main repo will still show side-repo-only files as untracked unless you add them to `.git/info/exclude` (local) or the main repo's `.gitignore` (committed). Prefer the local exclude for personal-only files.

## Stop tracking a file

```bash
inv rm --cached <path>          # remove from index, keep on disk
inv commit -m "stop tracking <path>"
inv push
```
