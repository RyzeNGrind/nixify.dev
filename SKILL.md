---
name: nixify.dev
description: >
  Developer-facing cluster-management skill for the DAS nixify monorepo. Covers the
  validate-host/release-host/rca-diff/lkg command semantics, colmena remote-build
  workflow, branch/commit SOP, remote-builder fabric, cells architecture, and
  validation marker protocol. Use when working in RyzeNGrind/nixify or onboarding
  a new agent to the cluster deployment workflow.
status: live
created: 2026-07-09
sources:
  - nixify cells/repo/devshells/default.nix (command definitions, quoted verbatim)
  - DAS/Reports/CLUSTER-DRIFT-AUDIT-2026-06-28.md
  - DAS/Governance/MICROVM-AGENT-PERSISTENCE-EVAL-2026-07-09.md
  - ~/Workspaces/CLAUDE.md (branch SOP + DoD three-gate)
---

# nixify.dev

DAS nixify cluster developer reference. All heavy Nix work (eval, build, activation) runs on
remote builders — NEVER locally on pc/p3520 (NixOS-WSL hosts, no meaningful build capacity).

---

## 1. Remote-Builder Fabric

| Host | Tailscale IP | Arch | Role |
|---|---|---|---|
| ws | 100.117.240.43 | x86_64-linux | PRIMARY builder — KVM present (42 G/98 G used 2026-07-09), CUDA, nixosTest |
| bld | 100.89.4.19 | x86_64-linux | Secondary x86_64 ROCm builder (tailnet up; provisioning state unconfirmed — probe `ssh bld 'id nix-builder'` before wiring buildMachines) |
| alpha | 100.125.115.81 | aarch64-linux | aarch64 builder; NO nested KVM (A1.Flex OCI); use for cross-compile/nspawn only |
| pc (this host) | WSL2 NixOS | x86_64 | Dev terminal only — `max-jobs = 0` enforced; dispatch all builds to ws |
| p3520 | offline as of 2026-07-09 | x86_64 | Dev terminal only when online |

**Builder wiring**: `NIX_CONFIG="builders = ssh-ng://ws x86_64-linux"` or the devshell's `nix-fast-build`
target (which inherits ws builder via distributed builds profile in `cells/nixos/nixosProfiles/default.nix`).

source: DAS/Reports/CLUSTER-DRIFT-AUDIT-2026-06-28.md; DAS/Governance/MICROVM-AGENT-PERSISTENCE-EVAL-2026-07-09.md §4a

---

## 2. devshell Commands

Load the devshell with `nix develop` (or `direnv allow` — operator only for initial allow).
The following commands are defined in `cells/repo/devshells/default.nix`:

### validate-host \<hostname\>

**Full remote validation pipeline. Stamps HEAD SHA for push gate.**

Steps (quoted from devshell source):
1. Clean working-tree check — uncommitted changes abort.
2. Ship HEAD to `ssh://ws/root/nixify.git` via named remote `ws-build`.
3. Remote eval + build of `nixosConfigurations.<host>` toplevel ON ws:
   `nix build 'git+file:///root/nixify.git?ref=refs/validate/<host>&rev=<sha>#nixosConfigurations.<host>.config.system.build.toplevel'`
4. Copy closure to local store via `nix copy --from ssh-ng://ws`.
5. Closure diff: `nvd diff <baseline> <out>` + `nix store diff-closures` + `nix-diff` RCA → saved to `~/.cache/nixify/rca-<host>-<sha7>.diff`.
6. Regression suite: enumerate all non-alpha flake checks on ws, build each sequentially (GC between heavy checks when ws store < 40 GB free). ABORTS if any non-alpha check fails — no marker, no release.
7. Activation: `switch-to-configuration dry-activate` then `test` (local if `HOST=$(hostname)`, else remote over SSH). ADR-008: if dry-activate mentions `dbus-broker` or user-units → use `boot` staging instead of live test.
8. Stamp marker: `.git/nixify/validated/<sha>` with line `<host> <toplevel> <ts> (validated|boot-staged-ADR008)[ colmena-test-pass] regression-pass`.
9. GC-root last-known-good: `~/.cache/nixify/lkg-<host>` → symlink to validated closure.

Overrides:
- `NIXIFY_NO_ACTIVATE=1` — skip test activation (marker verdict = `build-validated-only`; does NOT satisfy release gate).
- `NIXIFY_COLMENA_TEST=1` — additionally run `colmena apply test --on <host>` on ws for deploy-path proof (adds `colmena-test-pass` token).
- `NIXIFY_BUILD_HOST=<host>` — override builder (default `ws`).
- `NIXIFY_BUILD_REPO=<path>` — override bare repo path on builder (default `/root/nixify.git`).

source: nixify/cells/repo/devshells/default.nix `validate-host` command

### release-host \<hostname\>

**Tag validated HEAD + fast-forward release/\<host\> branch.**

Requires: validated marker at `.git/nixify/validated/<HEAD-sha>` for the specified host.

Actions:
- Reads closure path from marker, computes `narHash` via `nix path-info --json`.
- Creates annotated tag `release/<host>/<UTC-YYYYMMDD-HHMMSS>-<sha7>` with closure metadata.
- Fast-forwards branch `release/<host>` to HEAD (linear only — non-FF is refused).
- GC-roots release closure at `~/.cache/nixify/release-<host>-<sha7>`.
- Prints: `git push origin <tag> release/<host>` command to publish (push gate verifies marker + tag).

source: nixify/cells/repo/devshells/default.nix `release-host` command

### rca-diff \[baseline\] \[target\]

**Derivation-level root-cause diff. Defaults: `/run/current-system` vs `~/.cache/nixify/lkg-<hostname>`.**

Runs: `nvd diff <A> <B>` + `nix-diff <deriver-A> <deriver-B>` (requires `keep-derivations=true` at build time).

source: nixify/cells/repo/devshells/default.nix `rca-diff` command

### lkg

**Show last-known-good closures and validation markers.**

Prints:
- `ls -l ~/.cache/nixify/` — GC-rooted LKG + release closure symlinks.
- All marker files under `.git/nixify/validated/` with contents.

source: nixify/cells/repo/devshells/default.nix `lkg` command

### Other useful commands

| Command | Category | Purpose |
|---|---|---|
| `build-host <host>` | build | `colmena build --on <host>` with nom output |
| `deploy-host <host>` | deploy | `colmena apply --on <host>` with nom output |
| `diff-switch <host>` | deploy | Local `nixos-rebuild build` + nvd/nix-diff diff (standalone; heavy eval — run on ws) |
| `nfb` | build | `nix-fast-build --flake .#checks --skip-cached` |
| `check-eval` | build | `nix flake check --no-build` |
| `fmt` | dev | `alejandra .` (operator-preferred Nix formatter) |
| `lint` | dev | `deadnix --fail . && statix check .` |
| `harvest <ref> <path>` | migrate | Show pattern from source ref + diff vs working tree |
| `agent-status` | agents | Print WORKTREE.toml + last session notes |

---

## 3. Branch + Commit SOP

Branch namespace: `<owner>/<type>/<topic>`
- Owners: `claude`, `gemini`, `copilot`, `codex`, `RyzeNGrind`
- Types: `hotfix`, `feat`, `fix`, `refactor`, `docs`, `chore`, `ci`, `test`
- `master` / `RyzeNGrind/*` are OPERATOR-PROTECTED — agents never commit/push there.
  Override: `NIXIFY_OPERATOR=1` (requires explicit operator grant, quote date + exact wording + scope in commit body).
- `release/<host>` branches: linear FF pointers only, never direct commits (branch-guard enforced).

Commit style: conventional-commit `type(scope): summary`. Scopes in use: `home`, `nix`, `gates`, `cells`, `beta`, `colmena`, `agents`, `claude`. Fixes AMEND the introducing commit — never patch-on-top.

Escape hatches:
- `NIXIFY_BRANCH_OK=1` — bypass type-segment check for ephemeral worktrees.
- `NIXIFY_OPERATOR=1` — bypass operator-protection (must quote grant).

---

## 4. DoD Three-Gate (Definition of Done — 2026-06-25)

A task is DONE only when ALL THREE hold, in order:

1. **Proof** — acceptance test passes AND durable proof is stamped:
   - Nix repo: `.git/nixify/validated/<sha>` via `validate-host` (marker must contain `regression-pass`).
   - Web repo: CI run URL in commit/PR body.
2. **Merged** — work is merged to declared target branch (FF-only).
3. **Tagged** — `release/<host>/<UTC>-<sha7>` annotated tag records closure + evidence.

Until all three hold: status is `staged-for-review` or `blocked` — never `done/✅`.
`GREEN on ws` is a CLAIM, not DONE. Cite the marker SHA or CI URL.

---

## 5. Validation Marker Protocol

**Path**: `.git/nixify/validated/<commit-sha>`
**Format** (one line per host per SHA):
```
<host> <toplevel-store-path> <ts-UTC> (validated|boot-staged-ADR008)[ colmena-test-pass] regression-pass
```

Example:
```
ws /nix/store/abc123…-nixos-system-ws-25.05 2026-07-09T08:15:00Z validated regression-pass
```

- `validated` = `switch-to-configuration test` ran successfully.
- `boot-staged-ADR008` = WSL2 dbus/user-unit restart detected; used `boot` instead of live `test`; WSL instance restart completes activation.
- `regression-pass` = all non-alpha flake checks built green on ws during `validate-host`.
- `build-validated-only` = activation skipped (`NIXIFY_NO_ACTIVATE=1`); does NOT satisfy release push gate.

The pre-push hook refuses `release/<host>` branch/tag pushes whose HEAD SHA lacks the `regression-pass` token.

---

## 6. cells Architecture

Repo uses [divnix/std](https://github.com/divnix/std) `cellsFrom = "cells/"` convention.

| Cell | Path | Contents |
|---|---|---|
| `hosts` | `cells/hosts/` | `nixosConfigurations/*` — per-host NixOS closures |
| `modules` | `cells/modules/` | Reusable NixOS modules (cluster-hosts, ntfy-sh, cloudflared-tunnel-alpha, …) |
| `agents` | `cells/agents/` | Agent role NIX profiles, packages, skills, harness, tests |
| `agents/das/` | `cells/agents/das/` | DAS-specific agent roles (das-cto, das-ceo, …) |
| `agents/harness/` | `cells/agents/harness/` | OpenHive DAG integration: planner/implementer/tester/integrator |
| `agents/nixosProfiles/` | `cells/agents/nixosProfiles/` | Per-role NixOS profile (orchestrator, etc.) |
| `agents/skills/` | `cells/agents/skills/` | First-party SKILL.md deliverables (this file's sibling) |
| `repo` | `cells/repo/` | devshells, CI tooling |

Agent roster: defined declaratively in `cells/agents/` (7 generic std/hive roles + 4 harness roles + 4 DAS-specialized roles). NEVER invent new persona names — roles come only from the cell definitions.

source: nixify repo cells/ tree

---

## 7. Colmena Remote-Build Workflow

```bash
# Build only (no deploy):
build-host ws           # equivalent: colmena build --on ws

# Deploy (activation):
deploy-host ws          # equivalent: colmena apply --on ws

# Colmena deploy-path proof during validation (on ws, not local):
NIXIFY_COLMENA_TEST=1 validate-host ws
# — runs colmena apply test --on ws from ws-side checkout, adds colmena-test-pass token

# NEVER run colmena locally on pc/p3520 — evals the hive locally = heavy nix on constrained host
```

Alpha aarch64 build note: ws cannot build alpha (aarch64) natively. Use `--builders` override:
```bash
nix build .#nixosConfigurations.alpha.config.system.build.toplevel \
  --builders "ssh-ng://alpha aarch64-linux" --max-jobs 0
```

---

## 8. Pre-Commit Hooks Summary

Installed by `nix develop` via git-hooks.nix (ADR-006 — inline Nix, no hand-written hook files):

| Hook | Stage | Action |
|---|---|---|
| `alejandra` | pre-commit | Format all .nix files |
| `deadnix` | pre-commit | Flag dead Nix code |
| `statix-changed` | pre-commit | Statix anti-pattern lint on changed files only |
| `nix-instantiate` | pre-commit | Parse-validate every changed .nix file |
| `flake-checker` | pre-commit | Validate flake.lock on changes |
| `branch-guard` | pre-commit + pre-push | Block master/RyzeNGrind/* and enforce `<owner>/<type>/<topic>` |
| `remote-validated-push` | pre-push (stdin wrapper) | Block release/* pushes without `regression-pass` marker |

NEVER skip with `--no-verify`. NIXIFY_OPERATOR=1 is the only sanctioned bypass (with quoted grant).

---

## 9. nixos-infect + colmena adoption (OCI hosts)

Reference: https://github.com/elitak/nixos-infect — converts a running Ubuntu/Debian/CentOS OCI
instance to NixOS in-place. This is DESTRUCTIVE to the prior OS by design.

Reference config used for theta (2026-07-09): `references/theta-configuration.nix`

### Preflight checklist (run as `ssh ubuntu@<host>`)

| Check | Command | Expected |
|---|---|---|
| Architecture | `uname -m` | `x86_64` or `aarch64` |
| RAM | `free -h` | ≥1 GB (add swap if exactly 1 GB — see below) |
| Disk | `df -h /` | ≥10 GB free |
| EFI vs BIOS | `[ -d /sys/firmware/efi ] && echo UEFI || echo BIOS` | sets grub config |
| NOPASSWD sudo | `sudo -n true && echo OK` | must be `OK` — if needs password, STOP |
| Root authorized_keys | `sudo cat /root/.ssh/authorized_keys` | OCI adds forced-command; infect reads ubuntu user keys instead |
| Network interface | `ip -4 addr` | note interface name (ens3/eth0); needed if DHCP not auto |
| /etc/nixos exists? | `ls /etc/nixos/` | if present, infect SKIPS makeConf and uses existing files |

### Root-cause of prior theta brick

The previous infect ran with the stock auto-generated `configuration.nix`. The generated file
only copies SSH keys from the current user's `authorized_keys` into
`users.users.root.openssh.authorizedKeys.keys`, but OCI's root `authorized_keys` contains
`forced-command` restrictions that block login ("Please login as the user ubuntu"). After reboot
to NixOS, `root` had these restricted keys. Ubuntu user does not exist in NixOS. Result: SSH
lockout with no recovery path short of console/reinstall.

**Fix**: always provide a custom `oracle-override.nix` that explicitly defines all required users
(root, ryzengrind, nixbuilder) with clean keys and no forced-command.

### Custom config requirements for OCI hosts

All of the following MUST be present in the custom overlay file (e.g. `oracle-override.nix`):

1. `users.users.root.openssh.authorizedKeys.keys` — operator pubkeys WITHOUT forced-command prefix
2. `users.users.ryzengrind` — isNormalUser=true, wheel, same operator keys
3. `users.users.nixbuilder` — isNormalUser=true, group=nixbuilder, remote-builder key
4. `users.groups.nixbuilder = {}; users.groups.ryzengrind = {};` — groups before users
5. `security.sudo.wheelNeedsPassword = lib.mkForce false;`
6. `services.openssh.settings.PermitRootLogin = "prohibit-password"` (via oracle-cloud.nix module)
7. `services.openssh.settings.PasswordAuthentication = false` (via oracle-cloud.nix module)
8. `networking.firewall.allowedTCPPorts = [ 22 ]` (via oracle-cloud.nix module)
9. `services.tailscale.enable = true` — package only; enrollment needs operator authkey
10. `nix.settings.experimental-features = [ "nix-command" "flakes" ]`
11. `system.stateVersion = lib.mkForce "<channel-version>"` — must match `NIX_CHANNEL`

### Swap requirement on 1 GB shapes

1 GB RAM is insufficient for the NixOS closure build. Add swap before running infect:

```bash
# If no swap exists:
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
free -h  # verify Swap: 2.0G
```

nixos-infect itself creates a 1 GB swap at `/tmp/nixos-infect.*.swp` — but only after Nix is
installed. The channel unpack step (nix-env on nixexprs.tar.xz) can OOM before that swap
appears. Pre-adding 2 GB eliminates this race.

### NIXOS_IMPORT mechanism

nixos-infect generates `/etc/nixos/configuration.nix` and imports `$NIXOS_IMPORT` inside it:

```bash
# nixos-infect generated configuration.nix structure:
{ ... }: {
  imports = [
    ./hardware-configuration.nix
    $NIXOS_IMPORT   # ← your file goes here
  ];
  # ... auto-detected: hostName, sshKeys, zramSwap, stateVersion ...
}
```

**If `/etc/nixos/configuration.nix` already exists** (e.g. from a prior infect attempt),
infect's `makeConf()` returns immediately without regenerating — your existing files are used
as-is. In this case, edit the files directly on the host before re-running infect.

**Recommended pattern**: stage all custom files on the host, then run infect:

```bash
# Stage custom files
sudo tee /etc/nixos/oracle-override.nix > /dev/null < oracle-override.nix
sudo tee /etc/nixos/oracle-cloud.nix > /dev/null < oracle-cloud.nix
# (or fetch from beta: ssh pbeta 'sudo cat /etc/nixos/oracle-cloud.nix' | ssh ubuntu@theta 'sudo tee /etc/nixos/oracle-cloud.nix')

# Run infect — ALWAYS inside nohup/tmux to survive SSH disconnect
sudo -i nohup bash -c \
  'NIX_CHANNEL=nixos-25.05 bash <(curl --fail -L https://raw.githubusercontent.com/elitak/nixos-infect/master/nixos-infect) > /tmp/infect.log 2>&1' &
```

If using `NIXOS_IMPORT` instead (no pre-existing config):
```bash
NIXOS_IMPORT=/etc/nixos/oracle-override.nix \
NIX_CHANNEL=nixos-25.05 \
  bash <(curl --fail -L https://raw.githubusercontent.com/elitak/nixos-infect/master/nixos-infect)
```

### Infect safety: build-before-overwrite guarantee

nixos-infect runs `nix-env -A system` to build the NixOS closure BEFORE overwriting the
bootloader or filesystem. If the build fails, it aborts and the host remains on Ubuntu —
safe to retry. A failed infect is always recoverable; a successful infect that reboots into
a broken config is not.

### Post-infect host-key rotation

nixos-infect generates new SSH host keys. Any SSH client that cached the old Ubuntu host key
will see a host-key mismatch on reconnect:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
REMOTE HOST IDENTIFICATION HAS CHANGED!
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

**To reconnect safely** (use a scratch known_hosts to avoid polluting `~/.ssh/known_hosts`):

```bash
ssh -o UserKnownHostsFile=/tmp/theta-kh \
    -o StrictHostKeyChecking=accept-new \
    root@40.233.87.229 'hostname && nixos-version'
```

After verifying the host fingerprint, update `~/.ssh/known_hosts` by removing the old line
for `40.233.87.229` and adding the new one.

### Polling post-reboot (up to 10 min)

```bash
# Poll every 30s until SSH responds (host reboots automatically at end of infect):
until ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new \
          -o UserKnownHostsFile=/tmp/theta-kh \
          root@40.233.87.229 'hostname && nixos-version && id ryzengrind && id nixbuilder' 2>/dev/null; do
  echo "$(date): waiting for theta..."
  sleep 30
done
```

Expected output after successful infect:
```
theta
25.05.XXXXXX (NixOS)
uid=1000(ryzengrind) gid=100(users) groups=100(users),1(wheel)
uid=1001(nixbuilder) gid=1001(nixbuilder) groups=1001(nixbuilder),1(wheel)
```

### Adopting an infected host into colmena

After successful infect, adopt theta into `cells/hosts/` following the beta pattern:

1. **Add host cell** — `cells/hosts/nixosConfigurations/theta.nix` importing the same module
   chain (`oracle-cloud.nix`, `oracle-override.nix`, hardware-configuration.nix).
2. **Colmena target** — in `flake.nix` or `colmena.nix`:
   ```nix
   theta = {
     deployment = {
       targetHost = "40.233.87.229";  # or tailscale IP once enrolled
       buildOnTarget = false;          # always false — build on ws
       tags = [ "oci-x86_64" ];
     };
     imports = [ ./cells/hosts/nixosConfigurations/theta.nix ];
   };
   ```
3. **Build-on-ws only** — `buildOnTarget = false` is mandatory; theta has 1 GB RAM,
   insufficient for NixOS eval+build.
4. **Validate** — `NIXIFY_BUILD_HOST=ws validate-host theta` (builds on ws, activates on theta).
5. **Release** — `release-host theta` after validate-host stamps `regression-pass` marker.
6. **Tailscale enrollment** — operator runs `sudo tailscale up --authkey=<tskey>` on theta
   console after adoption; update `deployment.targetHost` to the tailscale IP for air-gapped
   cluster access.

### Infect monitoring pattern

The infect build takes 5–15 min on a 1 GB/2 vCPU shape. Monitor without blocking:

```bash
# From local machine — non-blocking tail:
ssh ubuntu@ptheta 'sudo tail -f /tmp/infect.log' &

# Key log milestones:
# "unpacking 1 channels..."   → nix-channel update phase (slow, ~2–5 min)
# "building the system..."    → nix-env -A system (main closure build, ~5–10 min)
# "finalizing the installation" → bootloader install + reboot pending
```
