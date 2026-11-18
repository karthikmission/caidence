# Caidence — Project Instructions

## Git Worktree Policy

Spec-kit's own scripts (`setup-plan.sh`, `setup-tasks.sh`, etc.) never create a
branch or worktree — they just operate on whatever branch is currently checked
out. That means `main` stays clean only if you make it stay clean yourself.

**Before writing or editing any application source file, or reading it to
decide what to write** (this does not apply to `specs/**` markdown artifacts
— `spec.md`, `plan.md`, `research.md`, `data-model.md`, `contracts/*`,
`quickstart.md`, `tasks.md` are all fine to write directly on `main`, *as
long as nothing reads application source to produce them*. `/speckit-tasks`
qualifies — it only reads the plan. `/speckit-converge` does not: it
compares `tasks.md` against the actual implemented code, so even though its
only write is to `tasks.md`, it needs the worktree too, same as
`/speckit-implement`):

1. Check whether a worktree for the current feature already exists:
   ```bash
   git worktree list
   ```
2. If the current feature (per `.specify/feature.json` / the active
   `specs/<NNN-slug>/` directory) has no worktree yet, create one before doing
   any implementation work, under the shared `worktrees/` parent — never as a
   sibling of `caidence/` directly:
   ```bash
   mkdir -p ../worktrees
   git worktree add ../worktrees/<NNN-slug> -b <NNN-slug>
   ```
3. Do all subsequent file edits and `/speckit-implement` work inside that
   worktree path, not in the original `caidence/` checkout. `main` is for
   finished, merged work only.

If you are a subagent dispatched to implement a specific task or Unit, assume
this policy applies to you too — check `git worktree list` yourself before
writing code rather than trusting that the parent session already did.

## Orient Yourself On Start

The workspace root is the directory that has `caidence/`, `driver/`, and
`worktrees/` as its three direct children. `caidence/` and `worktrees/<slug>/`
sit at **different depths** under it (one level down vs. two), so a fixed
relative path like `../driver` is only correct from one of them — don't
hardcode it. Find the workspace root yourself: walk up from your current
directory until you see all three as siblings, then reference the others by
absolute path from there.

Before touching any file, work out and state back:

1. Your current directory — if it's `<workspace-root>/caidence` on `main`,
   you are in the **REFERENCE** checkout: read-only, never modify it
   directly.
2. If you're in `<workspace-root>/worktrees/<NNN-slug>`, that is the
   **CHANGE** worktree for feature `<NNN-slug>` — this is where
   implementation work belongs.
3. `<workspace-root>/driver/` is **THINK** — prompts, research, scratch
   analysis, temporary notes, meeting notes, anything that isn't a durable
   project artifact. It's its own local-only git repo with no remote;
   nothing in it is ever part of a feature's commit. If you generate
   something for your own reasoning rather than for shipping, it goes there,
   not into REFERENCE or CHANGE.
4. Your current branch (`git branch --show-current`) — it should match the
   `<NNN-slug>` of the worktree you're in. If it doesn't, stop and ask rather
   than guessing which feature you're meant to be working on.

This replaces having a human paste an orientation prompt in by hand every
session — read this file, confirm the above (with real absolute paths, not
the placeholder angle brackets), then proceed.
