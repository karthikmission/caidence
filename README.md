# Caidence

A public registration-and-approval web app for workshop participants: people
register with their email and GitHub ID, an administrator approves them, and
on approval the system provisions their workshop access and hands them
credentials — once, over an authenticated session, never in plaintext
anywhere else.

This repo is also the workshop's own starter project: a stripped-down branch
of the same codebase, with the privileged provisioning step replaced by a
safe stub, is what participants fork and extend during the exercise.

Built using the same AI-DLC + Balrog methodology it exists to teach — see
`specs/` for the requirements, plan, and tasks this project is built from.

No infrastructure identifiers (account IDs, resource ARNs, internal URLs,
credentials) belong in this repo. It's public.
