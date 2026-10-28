# AGENTS.md

This file provides guidance to the AI agent when working with code in this repository.

## What this is

A single-file bash CLI (`ai`) plus `README.md`. There is no package manager, build step, CI, or
test suite — do not search for one. New behaviour belongs in `ai` or the README, not in a new file.

## Verify changes with

```bash
bash -n ai                                    # lint
./ai -l; ./ai -a; ./ai --missing; ./ai doctor # the observable surface
```

Everything runs against the real machine: detection walks `PATH` plus vendor install dirs, so
results are environment-dependent. Prefer `AI_EXTRA_DIRS=/scratch/bin` with fake executables over
installing anything real, and delete the scratch dir afterwards.

To exercise the picker, stub the `exec` line and drive it through a real pty (see the README's
"Testing without launching a real TUI"). `printf '%b'` for escapes, `\r` for Enter, always wrap in
`timeout` because raw mode disables EOF, and compare `stty -g` before/after to prove the tty is
restored.

## Rules the code depends on

- **Keep startup fork-free.** No `$(...)`, `grep`, or `command -v` inside the discovery or registry
  loops; helpers set globals (`S_PATH`, `FOUND_PATH`) instead of printing. A process per lookup
  took startup from 0.15s to 1.6s on WSL. The `AGENT_TOKENS` extglob pre-filter must stay in front
  of `AGENT_RE`, and the token list belongs in `AGENT_TOKENS`/config/env rather than a new regex.
- **`set -u` is on, `set -e` must stay off.** `[ ... ] && continue` and `(( ... ))` are used as
  non-fatal tests throughout. Possibly-empty arrays need `${a[@]+"${a[@]}"}`.
- **Launch args are split on spaces only, never globbed** — `--mode *` must reach the tool as a
  literal `*`. Registry/config entries are keyed by the binary's base name.
- **`stat` must use `-L`** in the permission check: a symlink's own mode is always 777, which would
  otherwise reject every npm-global tool.
- **bash 4+ only** (`declare -A`, `${s^}`); no GNU-only coreutils such as `readlink -f`.

## Repository etiquette

- This repo is **public**. Never write local machine details — home directory, `/mnt` layout,
  account names — into the README, comments, or commit messages; use `~`-relative paths.
- The working copy sits on a Windows (drvfs) mount: keep LF endings — CRLF breaks the shebang —
  and preserve the exec bit (`core.fileMode` is on).
- Commits use the repo-local GitHub noreply identity already in `.git/config`; do not touch global
  git config.
- Push after committing, but ask first for anything that rewrites history or force-pushes.
