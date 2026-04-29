---
name: inv-repo
description: Use when the user runs `inv` commands, mentions the side repo / extras repo / inveniam-extras, or wants to stage personal/dev files (Cursor rules, Claude config, local Makefile, per-developer docker-compose overrides). Also use whenever you'd otherwise suggest `git add` for a file under `.cursor/`, `.claude/`, `.history/`, `.idea/`, `.github/README.md`, the root `Makefile`, or `apps/ai-service/containers/*/docker-compose.yml` — those belong to the side repo, not the main repo.
---

# inv-repo (inveniam.io side repo)

This project uses a **dotfiles-style side repo** alongside the main `inveniam.io` repo to track personal/dev files without polluting the main repo.

- **Bare git dir**: `~/.inveniam-extras.git`
- **Work tree**: project root (shared with main repo)
- **Wrapper alias**: `inv` (defined in user's shell rc), expands to `git -C <project-root> --git-dir=$HOME/.inveniam-extras.git --work-tree=<project-root>`
- **Remote**: `git@github.com:ankur-dutta-jktech/inveniam-extras.git`
- **Authoritative file list**: `inv ls-files`

## When to use `inv` instead of `git`

Files that belong to the **side repo** (use `inv`):
- `.cursor/` (Cursor IDE rules + skills)
- `.claude/` (Claude Code settings, skills, agents)
- Root `Makefile`
- `apps/ai-service/containers/*/docker-compose.yml` (per-developer overrides)
- `.github/README.md` (the side repo's own README, rendered by GitHub)
- Other personal/IDE state if added (`.history/`, `.idea/`)

Run `inv ls-files` if unsure — it lists every path the side repo tracks.

Everything else uses plain `git` against the main repo.

## Rules — follow these exactly

1. **Never run `inv add .` or `inv add -A`.** The work tree is shared with the main repo, so a wildcard add would try to stage the entire project. Always pass explicit paths: `inv add path/to/file`.
2. **First-time staging of an ignored path needs `-f`.** The side repo reads the main repo's `.gitignore`, so paths like `graphify-out/`, `.history/`, `.idea/` need `inv add -f <path>` on the *initial* add. Subsequent edits to already-tracked files don't need `-f`.
3. **Never put secrets in the side repo.** No `.env*`, no credentials, no tokens. Treat it as code, not a vault. Refuse if the user asks.
4. **Don't suggest `git add` for side-repo paths.** If the user asks to commit something under `.cursor/`, `.claude/`, `Makefile`, etc., use `inv` — never `git`.
5. **Don't suggest `inv` for main-repo paths.** Source code, tests, configs that aren't in the side-repo list belong in the main repo and use plain `git`.
6. **Untrack with `--cached`.** To stop tracking without deleting from disk: `inv rm --cached <path>` then `inv commit`. Plain `inv rm` would delete the file.
7. **Pathspecs resolve from the project root** because the alias has `-C` baked in. Tell the user this if they hit a "pathspec did not match" error from a subdirectory — they don't need to `cd`.
8. **One initial README exclude.** When setting up a new machine/worktree, the user adds `.github/README.md` to the main repo's `.git/info/exclude` (local-only) so main `git status` doesn't flag it (the side repo's README sits under `.github/`, which the main repo otherwise tracks).

## Common operations

| Goal | Command |
| --- | --- |
| See side-repo status | `inv status` |
| Stage a tracked file | `inv add <path>` |
| Stage a `.gitignore`d path (first time only) | `inv add -f <path>` |
| Commit | `inv commit -m "..."` |
| Push | `inv push` |
| Pull on another machine/worktree | `inv pull` |
| List everything tracked | `inv ls-files` |
| Stop tracking a file | `inv rm --cached <path>` then commit |
| Inspect history | `inv log --oneline` |

## Worktrees

Each `git worktree add` of the main repo can have its own side-repo bare clone (e.g. `~/.inveniam-extras-feature-x.git`) with a per-worktree alias (`inv-fx`). Side-repo state is independent per worktree — sync via the GitHub remote with `inv push` / `inv pull`. See `.github/README.md` at the project root for the bootstrap commands.

## If the user hasn't set up `inv` yet

If `inv` isn't available (alias missing, bare repo missing), point them to `.github/README.md` → "First-time setup on a new machine". Don't try to set it up silently — walk through the steps so they know what's happening.
