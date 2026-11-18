# Caidence — Project Instructions

## Git Worktree Policy

Spec-kit's own scripts (`setup-plan.sh`, `setup-tasks.sh`, etc.) never create a
branch or worktree — they just operate on whatever branch is currently checked
out. That means `main` stays clean only if you make it stay clean yourself.

**Before writing or editing any application source file** (this does not apply
to `specs/**` markdown artifacts, which are fine to write directly):

1. Check whether a worktree for the current feature already exists:
   ```bash
   git worktree list
   ```
2. If the current feature (per `.specify/feature.json` / the active
   `specs/<NNN-slug>/` directory) has no worktree yet, create one before doing
   any implementation work:
   ```bash
   git worktree add ../caidence-<NNN-slug> -b <NNN-slug>
   ```
3. Do all subsequent file edits and `/speckit-implement` work inside that
   worktree path, not in the original `caidence/` checkout. `main` is for
   finished, merged work only.

If you are a subagent dispatched to implement a specific task or Unit, assume
this policy applies to you too — check `git worktree list` yourself before
writing code rather than trusting that the parent session already did.
