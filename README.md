# ssh-skill

An [opencode](https://opencode.ai) skill that separates **thinking** (local machine, `$S`) from **heavy computation** (remote server, `$D`). Only executable scripts and logs cross the wire — no LLM runs on the remote box. Designed for daily use against a group/HPC compute server over SSH.

> [!WARNING]
> It is generally not recommended for LLM to access remote server for security reason. This project is customized for myself, use at your own risk.

## What the skill does

When a task involves running experiments or builds on a remote server, the skill auto-triggers and enforces these rules:

**Local (`$S`)**
- No heavy computation — smoke tests and trivial checks only.
- Every remote experiment's results are pulled back to `$S` for analysis.
- Sync scope (whole project vs. needed files) is decided per task.

**Remote (`$D`)**
- No `/tmp` — scratch work goes in a project-local `./tmp/YYYYMMDD-HHMM-<slug>/`.
- No `sudo`/root — dependencies install as a normal user:
  - sources/builds → `~/toolset/`
  - binaries/symlinks → `~/.local/` (`~/.local/bin` on `PATH`)
- `$HOME/.my_vars` is sourced (`set -a; source ~/.my_vars; set +a`) before every experiment/build so tool/benchmark paths are available.
- Every experiment is logged (`log.txt`).
- Total thread budget ≤ **15 cores** for the whole run (`outer_jobs × threads_per_job ≤ 15`), enforced via `OMP/OPENBLAS/MKL/…_NUM_THREADS` caps; every run is wrapped in `/usr/bin/time -v` and hard-gated — >15 cores actually used means the run is invalid and must be re-run with tighter caps.
- Always rebuild on `$D` (different platform/toolchain than `$S`) — never trust synced binaries.
- Python projects use a project-local venv (`.venv`) or conda env matching the project.
- Self-resolve missing-tool errors via non-root install; never request sudo.
- Operations are recorded in `op.md`, which is pulled back to `$S`.

**Host resolution** — reads `~/.ssh/config`; if one `Host` entry exists it's used, if several the agent asks once per session, if none it asks for `user@host[:port]`. No hostnames or credentials are stored in the skill.

**Connection retries** — the first connection of a session is wrapped in a 5-attempt loop with 10 s sleeps (fast-fail per attempt); after the 5th failure it aborts with "failed to connect" and waits for the user (VPN-down case). Never longer sleeps.

**Review (`$S`)** — after each run, fetch `op.md` + logs, check for errors, verify against expectations, fix the script on `$S` and re-run if needed.

## Repository layout

```
ssh-skill/
├── README.md                    # this file
├── .opencode/
│   └── skills/ssh/SKILL.md      # the shippable skill (source of truth)
└── tests/
    └── TESTS.md                 # project-local validation checklist (not shipped globally)
```

The `tests/` folder is a validation sandbox only — it stays in this repo and is intentionally **not** copied to the global skill directory.

## Prerequisites

1. **opencode** installed.
2. **SSH key access** to the target server, with an entry in `~/.ssh/config`, e.g.:
   ```
   Host MyServer
       HostName 10.x.x.x
       User youruser
   ```
3. **`rsync`** on the local machine (preinstalled on macOS and most Linux distros).

## Deploy on a new local machine

```bash
# 1. clone
git clone <this-repo-url> ~/ssh-skill
cd ~/ssh-skill

# 2. install the skill globally (available in every project)
mkdir -p ~/.config/opencode/skills/ssh
cp .opencode/skills/ssh/SKILL.md ~/.config/opencode/skills/ssh/SKILL.md

# 3. confirm your ssh target is reachable
ssh MyServer 'hostname; uname -a; nproc'

# 4. (optional) validate the skill end-to-end — see tests/TESTS.md

# 5. restart opencode so the skill loads
```

After restart, the skill auto-triggers whenever a task mentions running experiments/builds on a remote server, an HPC/cluster/GPU box, syncing a project over ssh, or pulling back experiment logs and `op.md`.

## Validating the install

Open [`tests/TESTS.md`](./tests/TESTS.md) and run the T0–T13 checklist against a chosen host. Each test targets one skill rule (connectivity, non-root, no-`/tmp`, `~/toolset`+`~/.local`, env probe, venv, parallelism ≤15, log+`op.md` round-trip, rebuild-on-`$D`, self-resolution, `~/.my_vars` sourcing, thread budget + hard gate, gate rejection control, connection retry policy). All fourteen must pass before trusting the install. The checklist creates a throwaway `~/projects/ssh-skill-test/` on the target and cleans it up at the end.

## Customizing

- **Default host preference** — the skill asks whenever more than one `Host` exists. To bias it toward a specific box, keep a single primary `Host` in `~/.ssh/config` (or trim candidates).
- **Remote project base** — defaults to `~/projects/<name>`. Edit the sync patterns in `SKILL.md` §4 to change this.
- **Excludes** — extend the `rsync --exclude` list in `SKILL.md` §4 to match your project's artifacts.

## Files of interest

- `.opencode/skills/ssh/SKILL.md:1` — frontmatter (`name`, trigger `description`) and the full rule body.
- `tests/TESTS.md:1` — the validation checklist.
