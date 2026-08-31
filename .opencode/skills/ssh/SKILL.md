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

Use the chosen alias as `<D>` for every `ssh`/`rsync`/`scp` command.

### 0a. Connection retry policy

The VPN to `$D` is sometimes down. Wrap the FIRST connection of a session in this exact retry loop — max **5 attempts**, **10 s** sleep between attempts, fast-fail per attempt — then stop and report:

```
for i in 1 2 3 4 5; do
  ssh -o ConnectTimeout=10 -o ConnectionAttempts=1 <D> 'hostname' && break
  if [ "$i" = 5 ]; then echo "FAILED to connect to <D> after 5 attempts (VPN down? last ssh error above)"; exit 1; fi
  echo "attempt $i failed; retrying in 10s"; sleep 10
done
```

Rules:
- Never sleep more than **10 s** between attempts (no 60 s/600 s waits); never exceed **5** attempts total.
- `ConnectTimeout=10` makes each attempt fail fast; `ConnectionAttempts=1` stops ssh from retrying internally on top of this loop.
- After the 5th failure, STOP: report "failed to connect" with the last ssh error and wait for the user (likely VPN issue). Do not keep retrying and do not silently fall back to another host.
- Only the first connection of a session needs the loop; once verified up, run subsequent commands directly (still with `-o ConnectTimeout=10`).

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
echo "GTIME=$(ls /usr/bin/time 2>/dev/null || echo missing)"
echo "SHELL=$SHELL"'
```

Decide from the output (auto-detect first, install only what's missing):
- `python3` present → use it for the project venv.
- `conda` present and the project already uses conda → match that style.
- `~/toolset` or `~/.local` missing → create them (§3.3).
- `/usr/bin/time` missing → needed for the §3.7 gate; build GNU time into `~/toolset` per §3.3, or use the bash `TIMEFORMAT='%U %S %R'` fallback.
- `UID=0` → STOP, this skill forbids root; ask the user for a non-root account.

### 1a. Source `$HOME/.my_vars` (every session)

`$D` maintains a file `$HOME/.my_vars` of environment variables pointing to
useful tools/benchmarks. **Every** remote shell that runs an experiment or
build must export these first:

```
ssh <D> 'set -a; source "$HOME/.my_vars"; set +a; <your command>'
```

`set -a` makes every assignment in the file an exported env var; `set +a`
turns that off after. After sourcing, list what was loaded so you know which
tools/benchmarks are available:

```
ssh <D> 'set -a; source "$HOME/.my_vars"; set +a; env | sort'
```

If `$HOME/.my_vars` is absent, skip silently (do not fail the run) but mention
it to the user — they may need to create it on `$D`. Never write secrets from
`$S` into this file; it is maintained on `$D` by the user.

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
4. **Source `$HOME/.my_vars` before every experiment/build.** Prefix each remote command with `set -a; source "$HOME/.my_vars"; set +a;` (see §1a). This makes tool/benchmark paths from the file available to the run.
5. **Log every experiment.** Capture stdout+stderr with `2>&1 | tee ./tmp/<exp>/log.txt`.
6. **Total thread budget: ≤ 15 cores for the whole run.** The budget applies to the *sum* across all layers: `outer_jobs × threads_per_job ≤ 15`. Outer patterns (`make -j 15`, `xargs -P 15`, `ninja -j 15`, python `Pool(15)` / `n_jobs=15`) are valid ONLY with per-job threading capped at 1; a single-process run may use `T=15`. Most runtimes default their thread count to `nproc` (e.g. 64) — 15 jobs × 64 threads ≈ 960 threads saturates the whole box. So every experiment prefix extends §1a with thread caps:
   ```
   set -a; source "$HOME/.my_vars"; set +a
   T=1                                    # choose so that outer_jobs × T ≤ 15
   export OMP_NUM_THREADS=$T OPENBLAS_NUM_THREADS=$T MKL_NUM_THREADS=$T \
          NUMEXPR_NUM_THREADS=$T RAYON_NUM_THREADS=$T VECLIB_MAXIMUM_THREADS=$T
   ```
7. **Core-budget check — hard gate.** Wrap every experiment (the top-level command, not each job) in GNU time:
   ```
   /usr/bin/time -v -o ./tmp/<exp>/time.txt <experiment command>
   ```
   Then compute the cores actually used and gate on it:
   ```
   awk -F": " "/User time/{u=\$2}/System time/{s=\$2}/Elapsed/{n=split(\$NF,t,\":\");e=t[n]+(n>1?t[n-1]*60:0)+(n>2?t[n-2]*3600:0)}END{printf \"cores_used=%.2f\n\",(u+s)/e}" ./tmp/<exp>/time.txt
   ```
   Make the gate mechanical (non-zero exit on violation), never a judgment call:
   ```
   C=$(awk -F": " "/User time/{u=\$2}/System time/{s=\$2}/Elapsed/{n=split(\$NF,t,\":\");e=t[n]+(n>1?t[n-1]*60:0)+(n>2?t[n-2]*3600:0)}END{printf \"%.2f\",(u+s)/e}" ./tmp/<exp>/time.txt)
   awk -v c="$C" "BEGIN{exit !(c+0<=15.5)}" || { echo "GATE FAIL: cores_used=$C > 15"; exit 1; }
   ```
   Gate: `cores_used ≤ 15` (tolerance +0.5). If exceeded, the run is **INVALID** — tighten the rule-6 caps, re-run, and only then accept the results. Record the number in `op.md` (§5). If `/usr/bin/time` is missing, see the §1 fallback.

   **Scope of the control:** rule-6 env caps are *cooperative* — OpenMP/BLAS-style runtimes honor them, but code spawning raw threads (pthreads, Go, JVM, custom pools) ignores them; only this gate catches those, post-hoc. If a codebase repeatedly violates the budget, escalate to hard pinning: `taskset -c 0-14 <cmd>` (no root needed) — threads then timeshare 15 CPUs no matter how many are spawned.
8. **Always rebuild on `$D`** (different platform / toolchain). Sync source, then build on `$D`.
9. **Python projects:** create a project-local virtual environment (`.venv`), or a conda env matching the project's convention. Install every package there.
10. **Self-resolve errors.** If a build/experiment fails because a tool is missing, install it under `~/toolset`/`~/.local` per §3.3 and retry. Never report "needs sudo" — there is always a non-root path. Only after a genuine dead-end should you ask the user.
11. **Record operations in `op.md`** (§5) and bring it back to `$S`.

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
- Cores: ~N.N / 15 (time -v: pass | FAIL → tightened caps and re-ran)
- Errors: <none | description + how you fixed it>
- Artifacts: ./tmp/<exp>/{log.txt, results/}
- Next: <follow-up, if any>
```

## 6. Review on `$S` (after each remote run)

1. Pull `op.md` + the relevant `tmp/` logs to `$S`.
2. Check for errors; audit the `- Cores:` line (must be ≤ 15 and not FAIL); verify the result matches what was expected.
3. If errors → fix the script/plan on `$S`, re-push, re-run. Do **not** edit experiment logic "live" on `$D`.
4. Report findings to the user with `path:line` references.

## 7. Hard constraints (never violate)

- No LLM on `$D` — only executable scripts/plans cross the wire.
- No `sudo`, no `/tmp`, no heavy compute on `$S`.
- No hardcoded hostnames/credentials — resolve from `~/.ssh/config` or ask.
- Every remote operation is logged and reviewed on `$S`.
