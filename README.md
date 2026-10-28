> **Moved.** This plugin now lives in [heliowap/opencode-plugins](https://github.com/heliowap/opencode-plugins/tree/main/plugins/base-check). Install: `opencode plugin add 'github:heliowap/opencode-plugins#main::path:plugins/base-check'`. This repository is archived.

# opencode-v2-base-check

An OpenCode v2 plugin that tells a session how stale its git base is **before the first prompt
runs**, so no session starts blind on top of an outdated branch.

It never changes your work. The only command it runs that is not read-only is
`git fetch --prune`, which updates remote-tracking refs and touches neither the working tree, the
index nor any local ref. Every git call passes `--no-optional-locks`, because without it
`git status` rewrites `.git/index`. It does not pull, rebase, merge, stash or check out anything,
and it never asks a model to.

## What a session sees

The report is appended to the session's first prompt:

```
Base check: branch dev at 7c597b5.
Base: 1 commit behind origin/dev.
Upstream: 1 commit behind origin/dev.
Worktree: 2 uncommitted changes, so a rebase here would need a stash.
No files were changed: this check only ran git fetch --prune and read-only git commands. Do not pull, rebase or merge on your own initiative.
```

- **Base** is how far behind your integration branch you are. This is the "stale base" the plugin
  exists for.
- **Upstream** is how far your branch diverged from what it tracks, so unpushed commits and commits
  someone else pushed to your branch both show up.
- **Worktree** counts `git status --porcelain` entries, because they decide whether a rebase is even
  possible without a stash.
- A line starting with `Fetch:` appears only when `git fetch --prune` failed, and says that the
  counts came from local refs and may be stale.

The first prompt is waiting on this, so the whole check has a fifteen-second deadline, with ten of
them for the fetch. Past the deadline the prompt goes through with no report rather than stalling,
and a slow network degrades the report instead of the session.

A detached HEAD is reported as such and gets no upstream line. A directory that is not a repository
gets no report at all. Anything git declined to answer is reported as unread rather than filled in
with a plausible value: `Base check: branch dev at an unknown commit.`, `Worktree: could not be
read.`, `Base: the distance to origin/dev could not be counted.`

The last line is a prohibition, not an invitation: it tells the model not to pull, rebase or merge
on its own initiative, and it does not override a rebase you ask for yourself.

## Install

```jsonc title="opencode.jsonc"
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": [{ "package": "github:heliowap/opencode-v2-base-check", "options": { "base": "origin/dev" } }]
}
```

### `base`

The ref the session's base is compared against. Without it the plugin falls back to the remote's own
default branch (`refs/remotes/origin/HEAD`), which is often **not** the branch you work from: a
repository whose `origin/HEAD` points at `main` while everyone branches off `dev` needs
`"base": "origin/dev"` to report anything useful. A fallback says so in the report, so a count
against the wrong branch is never silent:

```
Base: 37 commits behind origin/main (the remote's default branch; set this plugin's "base" option to compare against another).
```

A configured ref the repository does not have is reported as such, and when nothing resolves at all
the report asks for the option instead of guessing.

### Loading it from a local checkout

```jsonc
{
  "plugins": [{ "package": "/absolute/path/to/opencode-v2-base-check", "options": { "base": "origin/dev" } }]
}
```

Relative paths, absolute paths, `file://` URLs and symlinks all work, verified on opencode 2.1.4.
What matters is where the directory's entry lives: OpenCode resolves a local plugin directory as a
path, trying `<dir>/server` and then `<dir>/index`, and never reads that directory's `package.json`
(`packages/plugin/src/host.ts`). A directory whose entry is declared only through `exports` or
`main`, pointing into `src/`, therefore loads nothing and logs nothing at all — upstream
[#52300](https://github.com/anomalyco/opencode/issues/52300). That is why this package keeps
`index.ts` at its root, re-exporting `src/index.ts`. An installed package does resolve `exports`, so
the `github:` form above was never affected.

## Why it works this way

OpenCode has no session-start hook. The available pieces are the `session.created` event, the
`prompt` hook, and plugin `setup`, and each alone is not enough:

- `session.created` plus `ctx.session.synthetic` does not guarantee ordering. Submit a prompt before
  the check finishes and the model's first turn runs without it.
- The `prompt` hook alone is deterministic but makes `git fetch` block the admission of your first
  prompt, which costs seconds in a large repository.

So the plugin uses both. `session.created` starts the check as a warm-up, which in an interactive
session means the fetch runs while you are still typing. The `prompt` hook then waits for that
result and appends it to the session's first prompt, starting the check itself if the warm-up did
not arrive in time, as happens with one-shot `opencode run`. Subagent sessions are skipped: they
inherit their parent's base. Later prompts in the same session are left alone, for as long as the
plugin runtime lives; after an OpenCode restart, a session you resume is checked once more, by then
against a base that may well have moved.

The report is **appended**, never prepended, because file and skill attachments carry `mention`
offsets into the prompt text and prefixing would shift every one of them.

## Why it does not rebase for you

Starting a session on a stale base is worth knowing about; fixing it automatically is not worth the
risk:

- A rebase rewrites commits. With uncommitted work it needs a stash, and on a conflict the session
  starts inside a broken repository, with the agent resolving conflicts before it knows the task.
- Detached HEADs are usually deliberate baselines for comparison. Moving them destroys the thing
  they exist to measure.
- Rewriting a branch that has an open pull request changes the review baseline and forces a push.
- A review fixed point recorded at the start of a task (`git rev-parse HEAD`) is invalidated by a
  rebase that happens underneath it.

Discovering staleness is cheap and safe. Integrating it is a decision with consequences, so it stays
an explicit step you ask for.

## Development

```sh
bun install
bun test
bun run typecheck
bun script/e2e.ts   # needs a configured provider; costs one model call per prompt
```

`src/facts.ts` and `src/describe.ts` are the tested seams. `src/index.ts` is the wiring that unit
tests cannot see, and `bun script/e2e.ts` covers it against a real OpenCode session: it serves a
throwaway clone from a private `opencode serve`, sends two prompts, and exits non-zero unless the
first carries the base check and the second does not.
