# CLAUDE.md

Instructions for coding agents working in this repository. `AGENTS.md` points here.

## What this is

A privacy-first, open-source Secret Santa web app on Cloudflare Workers. See `README.md`.
Design documents live in `docs/superpowers/specs/`.

## Repository facts

- **Canonical host: GitHub** (`rubicon/rubicon-secret-santa`). This is a public open-source
  repository, so GitHub is the active host, not a mirror.
- License: AGPL-3.0-only. New source files carry an `SPDX-License-Identifier: AGPL-3.0-only` header.
- Work flow: issue first, then a `dev/<issue>-<slug>` branch in a worktree under `worktrees/`,
  then a pull request with `Closes #N`. No direct pushes to `main`. Squash merge only.
- Commits: Conventional Commits, signed.

## Public-repository rules

- Never commit a real email address, Cloudflare account, zone, application or identity-provider ID,
  or a real hostname belonging to a deployment. Use `example.com` placeholders in code, tests,
  fixtures and docs. Real values belong in local config and Worker secrets only.
- No AI-authorship trailers or "Generated with" lines in commits, pull requests or files.
