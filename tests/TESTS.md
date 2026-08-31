# SSH skill — project-local test checklist

> **Scope:** these tests live ONLY in this repo. They validate that the skill
> in `.opencode/skills/ssh/SKILL.md` is functional. They are **not** shipped to
> the global skill directory. Run them from the repo root before trusting a
> global install.
>
> Each test targets one rule from the skill. Mark `PASS`/`FAIL` with the
> command output that proves it.

## Setup

1. Pick the target host. Replace `<D>` below with a host alias from
   `~/.ssh/config` (candidates: `MICSHPC`, `HKUSTGZServer`).
   ```
   D=<D>        # e.g. D=HKUSTGZServer
   PROJ=ssh-skill-test
   ```

2. Create the scratch project on `$D`:
   ```
   ssh $D "mkdir -p ~/projects/$PROJ/tmp && cd ~/projects/$PROJ && pwd"
   ```

---

## T0 — Host resolution + connectivity  *(skill §0, §1)*
**Goal:** the chosen alias connects and reports platform + cores.
```
ssh $D 'hostname; uname -a; nproc'
```
**Pass:** command returns a hostname, a Linux `uname` string, and an integer core count. No password prompt, no timeout.

---

## T1 — Non-root account  *(skill §1, §7)*
**Goal:** the remote user is not root.
```
ssh $D 'echo "uid=$(id -u) user=$(whoami)"; [ "$(id -u)" = "0" ] && echo ROOT || echo NONROOT'
```
**Pass:** prints `NONROOT`. (If `ROOT`, stop — the skill forbids root.)

---

## T2 — No `/tmp`, use project-local `./tmp`  *(skill §3.1)*
**Goal:** scratch files land under the project, never under `/tmp`.
```
ssh $D 'cd ~/projects/'"$PROJ"' && EXP=tmp/$(date +%Y%m%d-%H%M)-no-tmp && mkdir -p "$EXP" && echo hi > "$EXP/scratch.txt" && echo "WROTE=$(pwd)/$EXP/scratch.txt" && readlink -f "$EXP/scratch.txt"'
```
**Pass:** the resolved path starts with `$HOME/projects/$PROJ/tmp/...` and does **not** start with `/tmp`.

---

## T3 — Dependency layout: `~/toolset` + `~/.local`  *(skill §3.3)*
**Goal:** non-root install layout works; a `~/.local/bin` binary is runnable.
```
ssh $D 'set -e
mkdir -p ~/toolset ~/.local/bin
# fake "source" install
printf "#!/bin/sh\necho toolset-ok\n" > ~/toolset/fakebin.sh
chmod +x ~/toolset/fakebin.sh
# expose via ~/.local/bin symlink
ln -sf ~/toolset/fakebin.sh ~/.local/bin/fakebin
export PATH="$HOME/.local/bin:$PATH"
command -v fakebin && fakebin'
```
**Pass:** prints a path ending in `/.local/bin/fakebin` and then `toolset-ok`.

---

## T4 — Environment probe parses  *(skill §1)*
**Goal:** the exact probe command from §1 runs and emits every expected key.
```
ssh $D 'set +e
echo "HOST=$(hostname)"
echo "CORES=$(nproc 2>/dev/null || getconf _NPROCESSORS_ONLN)"
echo "PY=$(python3 --version 2>&1 || echo missing)"
echo "CONDA=$(conda --version 2>&1 || echo missing)"
echo "UID=$(id -u)"
for d in "$HOME/toolset" "$HOME/.local"; do [ -d "$d" ] && echo "HAS:$d" || echo "MISSING:$d"; done'
```
**Pass:** output contains lines starting with `HOST=`, `CORES=`, `PY=`, `CONDA=`, `UID=`, and at least one `HAS:` or `MISSING:` line. Record whether python/conda are present for T5.

---

## T5 — Project-local Python venv  *(skill §3.9)*
**Goal:** create a venv *inside the project*, install a tiny package, import it.
```
ssh $D 'set -e
cd ~/projects/'"$PROJ"'
[ -d .venv ] || python3 -m venv .venv
. .venv/bin/activate
pip install -q six
python -c "import six; print(\"six-\", six.__version__)"'
```
**Pass:** prints `six- <version>`. (If T4 showed python is missing, install python under `~/toolset`/`~/.local` first per skill §3.3 and re-run; do not use sudo.)

---

## T6 — Parallelism capped at 15 cores  *(skill §3.6)*
**Goal:** a job runs across multiple workers, never requesting more than 15.
```
ssh $D 'cd ~/projects/'"$PROJ"'
CORES=$(nproc 2>/dev/null || getconf _NPROCESSORS_ONLN)
N=$(( CORES < 15 ? CORES : 15 ))
echo "using N=$N workers (cores=$CORES)"
seq 1 $((N*4)) | xargs -P "$N" -I{} sh -c "echo worker-{} on $(hostname)" | sort | uniq -c | head'
```
**Pass:** prints `using N=…` with `N ≤ 15`, and workers actually execute in parallel (many distinct `worker-*` lines). No `xargs` error.

---

## T7 — Experiment log + `op.md` round-trip  *(skill §3.5, §5, §6)*
**Goal:** run a trivial experiment that writes `log.txt` + `op.md`, then rsync them back to `$S`.
```
# on $D
ssh $D 'cd ~/projects/'"$PROJ"' && EXP=tmp/$(date +%Y%m%d-%H%M)-roundtrip && mkdir -p "$EXP"
( echo "running"; seq 1 3 | sed "s/^/line /"; echo done ) 2>&1 | tee "$EXP/log.txt"
cat > "$EXP/op.md" <<EOF
## $(date "+%Y-%m-%d %H:%M") roundtrip
- Host: '"$D"'
- Goal: validate log+op.md pullback
- Commands: echo/seq
- Result: pass
- Artifacts: $EXP/log.txt
EOF
echo "EXP=$EXP" > /tmp/ssh-skill-exp-path 2>/dev/null || true
cd ~/projects/'"$PROJ"' && ls tmp/'
# pull back to $S
rsync -avz "$D:~/projects/$PROJ/tmp/" ./tmp/
```
**Pass:** locally, `./tmp/*-roundtrip/log.txt` contains `line 1`/`line 2`/`line 3`/`done`, and the matching `op.md` contains `- Host:` and `Artifacts:`.

---

## T8 — Always rebuild on `$D`  *(skill §3.6)*
**Goal:** sync source from `$S`, build on `$D`, run, capture output. (C if a compiler exists; Python fallback otherwise.)
```
mkdir -p src && printf '#include <stdio.h>\nint main(){printf("built-on-D\\n");return 0;}\n' > src/hello.c
rsync -avz --exclude='.git' --exclude='tmp' --exclude='.venv' ./ "$D:~/projects/$PROJ/"
ssh $D 'cd ~/projects/'"$PROJ"' && (cc src/hello.c -o tmp/hello 2>/dev/null && ./tmp/hello) || (echo "no-cc; python fallback"; echo "print(\"built-on-D\")" > tmp/hello.py && python3 tmp/hello.py)'
```
**Pass:** prints `built-on-D`. The binary/script was built on `$D`, not copied from `$S`.

---

## T9 — Self-resolution stress (no sudo)  *(skill §3.10)*
**Goal:** when a tool is missing, the skill installs it under `~/toolset`/`~/.local` **without sudo**.
```
# ask for a deliberately missing tool, then provide a non-root install
ssh $D 'set -e
export PATH="$HOME/.local/bin:$PATH"
if command -v rts-yq-real >/dev/null 2>&1; then echo already; exit; fi
# simulate "tool missing -> install non-root" with a tiny shell script placed in ~/.local/bin
printf "#!/bin/sh\necho resolved-from-local\n" > ~/.local/bin/rts-yq-real
chmod +x ~/.local/bin/rts-yq-real
# verify no sudo was used and the tool now resolves
command -v rts-yq-real && rts-yq-real'
```
**Pass:** prints a path under `~/.local/bin/rts-yq-real` then `resolved-from-local`. Throughout, no `sudo`, no `apt`/`yum` requiring root.

---

## T10 — Source `$HOME/.my_vars` before experiments  *(skill §1a, §3.4)*
**Goal:** `set -a; source ~/.my_vars; set +a` exports the file's variables into the experiment shell (visible to child processes).

This test is non-destructive: it backs up any existing `~/.my_vars`, writes a throwaway test file, verifies, then restores the original.
```
ssh $D 'set -e
F="$HOME/.my_vars"
# back up existing file (if any)
if [ -f "$F" ]; then cp "$F" "$F.bak.$(date +%s)"; HAD=1; else HAD=0; fi
# write a throwaway vars file
printf "%s\n" "MYBENCH=$HOME/benchmarks/foo" "MYTOOL=frobnicate" > "$F"
# source + verify export into a CHILD process (python), proving they left the shell
set -a; source "$F"; set +a
echo "shell-sees: MYBENCH=$MYBENCH MYTOOL=$MYTOOL"
python3 -c "import os;print(\"child-sees:\", os.environ.get(\"MYBENCH\"), os.environ.get(\"MYTOOL\"))"
# restore
if [ "$HAD" = "1" ]; then mv "$F.bak."* "$F"; else rm -f "$F"; fi
echo "restored (HAD=$HAD)"'
```
**Pass:** `shell-sees:` and `child-sees:` both print `MYBENCH=.../benchmarks/foo` and `MYTOOL=frobnicate`, and the final line says `restored (HAD=0|1)` (original file state preserved).

---

## T11 — Total thread budget ≤ 15 (nested threading) + hard gate  *(skill §3.6, §3.7)*

**Goal:** (a) demonstrate that outer `-P 15` alone oversubscribes when each job spawns its own threads (BLAS defaults to `nproc`); (b) verify the thread-cap prefix + `/usr/bin/time -v` gate keep total usage ≤ 15.

> Step (a) deliberately saturates the box for a few seconds — run when the server is idle, or shrink the matrix size.

```
ssh $D 'set -e
set -a; source "$HOME/.my_vars"; set +a
cd ~/projects/'"$PROJ"'
[ -d .venv ] || python3 -m venv .venv
.venv/bin/pip install -q numpy
mkdir -p tmp/t11

cat > tmp/t11/mm.py <<"PY"
import numpy as np, sys, time
a = np.random.rand(1500, 1500); b = np.random.rand(1500, 1500)
t = time.time()
for _ in range(int(sys.argv[1])): c = a @ b
print("job-done", round(time.time() - t, 2))
PY

cat > tmp/t11/cores.sh <<"SH"
#!/bin/sh
awk -F": " "/User time/{u=\$2}/System time/{s=\$2}/Elapsed/{n=split(\$NF,t,\":\");e=t[n]+(n>1?t[n-1]*60:0)+(n>2?t[n-2]*3600:0)}END{printf \"cores_used=%.2f\n\",(u+s)/e}" "$1"
SH
chmod +x tmp/t11/cores.sh

# (a) NO caps: 15 jobs, each free to spawn nproc BLAS threads
seq 1 15 | /usr/bin/time -v -o tmp/t11/time_nocaps.txt xargs -P 15 -I{} .venv/bin/python tmp/t11/mm.py 4
./tmp/t11/cores.sh tmp/t11/time_nocaps.txt

# (b) capped: same jobs with the rule-6 thread prefix
export OMP_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 MKL_NUM_THREADS=1 NUMEXPR_NUM_THREADS=1 RAYON_NUM_THREADS=1 VECLIB_MAXIMUM_THREADS=1
seq 1 15 | /usr/bin/time -v -o tmp/t11/time_capped.txt xargs -P 15 -I{} .venv/bin/python tmp/t11/mm.py 4
./tmp/t11/cores.sh tmp/t11/time_capped.txt'
```

**Pass:**
- (a) prints `cores_used` **> 15** — reproduces the oversubscription hole (this sub-step intentionally violates the gate).
- (b) prints `cores_used` **≤ 15.5** — the hard gate passes once the caps are exported.

---

## T12 — Control verification: gate rejects violations; env caps are cooperative  *(skill §3.6, §3.7)*

**Goal:** prove the budget control actually *enforces*: (1) the gate exits 0 on a compliant run, (2) a workload that ignores env caps is measured over-budget and the gate **exits 1**, (3) `taskset` contains even that workload (the escalation path).

```
ssh $D 'set -e
set -a; source "$HOME/.my_vars"; set +a
cd ~/projects/'"$PROJ"'
mkdir -p tmp/t12
# gate: parse time.txt, exit non-zero on violation
cat > tmp/t12/gate.sh <<"SH"
#!/bin/sh
L=${2:-15.5}
C=$(awk -F": " "/User time/{u=\$2}/System time/{s=\$2}/Elapsed/{n=split(\$NF,t,\":\");e=t[n]+(n>1?t[n-1]*60:0)+(n>2?t[n-2]*3600:0)}END{printf \"%.2f\",(u+s)/e}" "$1")
echo "cores_used=$C limit=$L"
awk -v c="$C" -v l="$L" "BEGIN{exit !(c+0<=l+0)}"
SH
chmod +x tmp/t12/gate.sh

# (1) compliant: caps + xargs -P 15  -> expect GATE exit 0
export OMP_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 MKL_NUM_THREADS=1 NUMEXPR_NUM_THREADS=1 RAYON_NUM_THREADS=1 VECLIB_MAXIMUM_THREADS=1
seq 1 15 | /usr/bin/time -v -o tmp/t12/time_capped.txt xargs -P 15 -I{} .venv/bin/python tmp/t11/mm.py 4 > /dev/null
./tmp/t12/gate.sh tmp/t12/time_capped.txt && echo "GATE: PASS (exit 0)" || echo "GATE: FAIL (exit 1)"

# (2) adversarial: 60 raw pthreads, ALL caps exported -> expect cores >> 15, GATE exit 1
/usr/bin/time -v -o tmp/t12/time_adv.txt ./tmp/t12/adv 60
if ./tmp/t12/gate.sh tmp/t12/time_adv.txt; then echo "GATE: PASS (exit 0)"; else echo "GATE: FAIL (exit 1) -> run correctly REJECTED"; fi

# (3) escalation: same program hard-pinned -> expect cores <= 15, GATE exit 0
taskset -c 0-14 /usr/bin/time -v -o tmp/t12/time_adv_pin.txt ./tmp/t12/adv 60
./tmp/t12/gate.sh tmp/t12/time_adv_pin.txt && echo "GATE: PASS under taskset (exit 0)"'
```

(Reuses `tmp/t11/mm.py` and `tmp/t12/adv` from T11's build: `cc -O2 -pthread adv.c -o adv` on a 60-thread busy-loop program.)

**Pass:** exactly this pattern — compliant run `cores_used ≤ 15.5` + exit 0; adversarial run `cores_used > 15` (env caps ignored, as designed) + exit 1; taskset run `cores_used ≤ 15.5` + exit 0.

---

## T13 — Connection retry policy (5 attempts, 10 s sleep, then abort)  *(skill §0a)*

**Goal:** verify the retry loop makes exactly 5 fast-failing attempts with 10 s sleeps, then aborts with the FAILED message (no long sleeps) — and that a reachable host passes on attempt 1.

Runs on `$S`; no remote changes. Part (a) targets a local port with nothing listening (instant "connection refused"), so elapsed time isolates the four 10 s sleeps.

```
D=<D>     # your host alias
attempts=0
retry() { # $1 = ssh args
  for i in 1 2 3 4 5; do
    attempts=$((attempts+1))
    ssh -o ConnectTimeout=10 -o ConnectionAttempts=1 "$@" 'hostname' 2>/dev/null && return 0
    if [ "$i" = 5 ]; then echo "FAILED to connect after 5 attempts"; return 1; fi
    sleep 10
  done
}

# (a) unreachable target
attempts=0; S=$(date +%s); retry -p 22222 127.0.0.1; RC=$?; E=$(( $(date +%s) - S ))
echo "(a) attempts=$attempts rc=$RC elapsed=${E}s"

# (b) reachable target
attempts=0; S=$(date +%s); retry "$D"; RC=$?; E=$(( $(date +%s) - S ))
echo "(b) attempts=$attempts rc=$RC elapsed=${E}s"
```

**Pass:**
- (a) `attempts=5`, `rc=1`, `FAILED to connect after 5 attempts` printed, `elapsed` ≈ 40 s and **< 90 s** (proves 10 s sleeps, nothing like 600 s).
- (b) `attempts=1`, `rc=0`, fast.

---

## Result summary

| Test | Rule | Result |
|------|------|--------|
| T0 | host resolution + connectivity | ☐ PASS / ☐ FAIL |
| T1 | non-root account | ☐ PASS / ☐ FAIL |
| T2 | no `/tmp`, project-local `./tmp` | ☐ PASS / ☐ FAIL |
| T3 | `~/toolset` + `~/.local` layout | ☐ PASS / ☐ FAIL |
| T4 | env probe parses | ☐ PASS / ☐ FAIL |
| T5 | project-local venv | ☐ PASS / ☐ FAIL |
| T6 | parallelism ≤ 15 | ☐ PASS / ☐ FAIL |
| T7 | log + op.md round-trip | ☐ PASS / ☐ FAIL |
| T8 | rebuild on `$D` | ☐ PASS / ☐ FAIL |
| T9 | self-resolution, no sudo | ☐ PASS / ☐ FAIL |
| T10 | source `$HOME/.my_vars` before experiments | ☐ PASS / ☐ FAIL |
| T11 | total thread budget ≤ 15 + hard gate | ☐ PASS / ☐ FAIL |
| T12 | gate rejects violations; env caps cooperative | ☐ PASS / ☐ FAIL |
| T13 | connection retry 5 × 10 s then abort | ☐ PASS / ☐ FAIL |

**Gate:** all fourteen must be PASS before copying `SKILL.md` to `~/.config/opencode/skills/ssh/`.

## Cleanup (after the gate passes)
```
ssh $D "rm -rf ~/projects/$PROJ"
rm -rf ./tmp src
```
