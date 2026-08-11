---
name: ssh
description: Use when running experiments or builds on a remote compute server over SSH, syncing a project to a remote machine, or pulling back experiment logs and an op.md for review. Enforces the local/remote split — thinking on local, heavy compute on the remote server via ssh; no sudo; deps under ~/toolset or ~/.local; no /tmp; project-local venv; up to 15 cores. Trigger keywords: ssh, remote server, cluster, HPC, GPU box, deploy to server, run experiment remotely, sync project, op.md, experiment logs.
---

# SSH remote-compute skill

Separates thinking (local, `$S`) from heavy computation (remote, `$D`). Only
executable scripts/plans and logs cross the wire — **never call an LLM on `$D`**.

Notation: `$S` = local machine, `$D` = remote server, `<D>` = the chosen ssh host alias.

## When to apply

Use when the task involves running experiments, builds, or heavy computation on
a remote server reached over ssh. If work is light enough to run locally
(smoke test, tiny check), do **not** use this skill — just run it on `$S`.

## 0. Resolve the remote host

Never hardcode hostnames, IPs, or credentials. They live in `~/.ssh/config`.

1. Read `~/.ssh/config` and collect every `Host` entry that looks like a compute target.
2. **Exactly one** candidate → use it.
3. **More than one** → ask the user which host to use this session; cache the choice.
4. **None** → ask the user for `user@host[:port]`.

Use the chosen alias as `<D>` for every `ssh`/`rsync`/`scp` command. If a
connection fails, surface the exact ssh error and re-resolve.

## 1. Probe the remote environment (once per session)

Before real work, run a single one-shot probe and adapt all later steps to it:

```
ssh <D> 'set +e
echo "HOST=$(hostname)"
echo "UNAME=$(uname -a)"
echo "CORES=$(nproc 2>/dev/null || getconf _NPROCESSORS_ONLN)"
echo "HOME=$HOME"
echo "PY=$(python3 --version 2>&1 || echo missing)"
echo "CONDA=$(conda --version 2>&1 || echo missing)"
echo "PIP=$(pip3 --version 2>&1 || echo missing)"
for d in "$HOME/toolset" "$HOME/.local"; do [ -d "$d" ] && echo "HAS:$d" || echo "MISSING:$d"; done
echo "UID=$(id -u)"
echo "SHELL=$SHELL"'
```

Decide from the output (auto-detect first, install only what's missing):
- `python3` present → use it for the project venv.
- `conda` present and the project already uses conda → match that style.
- `~/toolset` or `~/.local` missing → create them (§3.3).
- `UID=0` → STOP, this skill forbids root; ask the user for a non-root account.

## 2. Local rules (`$S`)

1. **No heavy computation locally** — smoke tests and trivial checks only.
2. **Always pull results back** to `$S` for analysis after each remote experiment.
3. **Decide sync scope per task** — whole project vs. just the needed files. Prefer minimal sync when the project is large.

## 3. Remote rules (`$D`)

1. **No `/tmp`.** Use a project-local temp dir: `./tmp/`. Every experiment lives in its own folder `./tmp/YYYYMMDD-HHMM-<slug>/`.
2. **No root, no sudo.** Anything that seems to need elevated privilege must instead be installed as a non-root user.
3. **Dependency layout:**
   - Sources / builds → `~/toolset/` (e.g. `~/toolset/foo-1.2/`).
   - Binaries / symlinks → `~/.local/` (e.g. `~/.local/bin/foo`).
   - Put `~/.local/bin` on PATH: `export PATH="$HOME/.local/bin:$PATH"` (append to `~/.bashrc`/`~/.zshrc` if it should persist for the session).
   - Python packages go into the project venv, not system site-packages.
4. **Log every experiment.** Capture stdout+stderr with `2>&1 | tee ./tmp/<exp>/log.txt`.
5. **Parallelism up to 15 cores.** Cap at `min(15, nproc)`. Patterns: `make -j 15`, `xargs -P 15`, `ninja -j 15`, python `multiprocessing.Pool(15)` / joblib `n_jobs=15`.
6. **Always rebuild on `$D$`.** Never trust binaries synced from `$S` (different platform / toolchain). Sync source, then build on `$D$`.
7. **Python projects:** create a project-local virtual environment (`.venv`), or a conda env matching the project's convention. Install every package there.
8. **Self-resolve errors.** If a build/experiment fails because a tool is missing, install it under `~/toolset`/`~/.local` per §3.3 and retry. Never report "needs sudo" — there is always a non-root path. Only after a genuine dead-end should you ask the user.
9. **Record operations in `op.md`** (§5) and bring it back to `$S`.

## 4. Sync patterns (`$S` ↔ `$D`)

Let `$PROJ` = project name, remote base `~/projects/$PROJ`.

Up (source only; exclude build artifacts and secrets):
```
rsync -avz --delete \
  --exclude='.git/' --exclude='node_modules/' --exclude='__pycache__/' \
  --exclude='.venv/' --exclude='env/' --exclude='tmp/' \
  --exclude='*.o' --exclude='*.pyc' --exclude='build/' --exclude='dist/' \
  --exclude='.env' \
  ./ "<D>:~/projects/$PROJ/"
```

Down (results + op.md + logs only):
```
rsync -avz "<D>:~/projects/$PROJ/tmp/" ./tmp/
rsync -avz "<D>:~/projects/$PROJ/op.md" ./op.md
```

Never sync secrets (`.env` with keys, credentials, private tokens).

## 5. `op.md` convention

`op.md` is an append-only log of what was done on `$D`. One section per
session/experiment:

```
## YYYY-MM-DD HH:MM <slug>
- Host: <D>
- Goal: <one line>
- Commands:
    <verbatim or faithful summary>
- Result: <pass/fail + key numbers>
- Errors: <none | description + how you fixed it>
- Artifacts: ./tmp/<exp>/{log.txt, results/}
- Next: <follow-up, if any>
```

## 6. Review on `$S` (after each remote run)

1. Pull `op.md` + the relevant `tmp/` logs to `$S`.
2. Check for errors; verify the result matches what was expected.
3. If errors → fix the script/plan on `$S`, re-push, re-run. Do **not** edit experiment logic "live" on `$D`.
4. Report findings to the user with `path:line` references.

## 7. Hard constraints (never violate)

- No LLM on `$D` — only executable scripts/plans cross the wire.
- No `sudo`, no `/tmp`, no heavy compute on `$S`.
- No hardcoded hostnames/credentials — resolve from `~/.ssh/config` or ask.
- Every remote operation is logged and reviewed on `$S`.
