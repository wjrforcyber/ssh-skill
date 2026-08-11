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

## T5 — Project-local Python venv  *(skill §3.7)*
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

## T6 — Parallelism capped at 15 cores  *(skill §3.5)*
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

## T7 — Experiment log + `op.md` round-trip  *(skill §3.4, §5, §6)*
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

## T9 — Self-resolution stress (no sudo)  *(skill §3.8)*
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

**Gate:** all ten must be PASS before copying `SKILL.md` to `~/.config/opencode/skills/ssh/`.

## Cleanup (after the gate passes)
```
ssh $D "rm -rf ~/projects/$PROJ"
rm -rf ./tmp src
```
