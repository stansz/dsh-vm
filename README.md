# dsh-vm

Run [cc_to_dsh](https://github.com/stansz/cc_to_dsh) handoffs inside a **microVM**. Claude Code
plans, dsh executes with **full access** (no sandbox, no approvals) inside a throwaway VM, and
Claude Code verifies the result before anything touches your repo.

Why: dsh's own sandbox has to block a lot (browsers, package installs, random tools) to stay safe.
Here the VM is the security boundary instead, so dsh can do anything inside it, and the worst case
is a wrecked VM that gets deleted anyway.

```
Claude Code (host) ── writes .handoff/plan.md
      │
      └─ dsh-vm run ─▶ git clone --no-hardlinks → ~/.cache/dsh-vm/runs/<repo>-<time>/
                        podman run --runtime krun   (own kernel, 2 GB, 2 CPUs, --rm)
                          ├─ the clone mounted at /work (virtiofs)
                          ├─ DEEPSEEK_API_KEY + network
                          └─ dsh --profile headless --permission-mode danger-full-access
      │
Claude Code (host) ── reviews the diff in the run dir, runs Acceptance there
      │
      └─ dsh-vm apply ─▶ changes land in your repo as uncommitted changes → you commit
```

## What the VM can and cannot reach

| Inside the VM | Not inside the VM |
|---|---|
| A clone of the repo (with `.handoff/`) | Your real checkout and its `.git` |
| `DEEPSEEK_API_KEY` | SSH keys, `gh` token, any git remote: **it cannot push** |
| The network (DeepSeek API, npm, web) | Your home directory and the rest of the host |
| The npm download cache (a podman volume) | The host kernel (libkrun gives the VM its own) |

The clone uses `--no-hardlinks`, so the VM can't modify your repo's object files through shared
inodes. The key is the one secret inside; use a spend-capped key.

dsh runs as **root** inside the VM, so it can `dnf install` whatever a task needs. Under rootless
Podman that root is your own unprivileged user on the host, and files it writes come back owned
by you. (sudo would not work anyway: under krun the image reaches the VM over virtiofs, where
setuid binaries don't get root.) Chromium runs with `--no-sandbox`, since the VM is the sandbox.

## What's in the image

Fedora 45 with: Node 24 (npm, plus pnpm and yarn via corepack), dsh, git, headless Chromium,
gcc/g++/make for native modules, python3, ripgrep, fd, jq, curl, unzip/tar/xz, patch, less,
file, tree. Anything else: dsh can `dnf install` it during the run.

## Why libkrun, not Firecracker

Both are minimal Rust VMMs with similar overhead (~100 ms boot, a few MB). libkrun wins here
because it shares a host folder over **virtiofs** (Firecracker has block devices only), runs
**rootless** with no TAP/NAT setup (its TSI networking proxies guest sockets through the host),
and takes an ordinary container image via `podman --runtime krun`.

## Requirements

- Linux with KVM (`/dev/kvm`), rootless Podman, and `crun-krun` (Fedora: `sudo dnf install crun-krun`)
- `DEEPSEEK_API_KEY` in the environment Claude Code runs in

## Install

```bash
git clone https://github.com/stansz/dsh-vm ~/code/dsh-vm
cd ~/code/dsh-vm
bin/dsh-vm build                                   # ~2.5 GB image, a few minutes
ln -sf "$PWD/bin/dsh-vm" ~/.local/bin/dsh-vm
mkdir -p ~/.claude/skills && cp -r skills/handoff ~/.claude/skills/
```

This installs the VM handoff as **the** `handoff` skill. If you already have the
[cc_to_dsh](https://github.com/stansz/cc_to_dsh) `handoff` skill and want to keep it as a
no-VM fallback, copy it to `handoff-local` first and rename it:

```bash
cp -r ~/.claude/skills/handoff ~/.claude/skills/handoff-local
sed -i 's/^name: handoff$/name: handoff-local/' ~/.claude/skills/handoff-local/SKILL.md
```

and change its `description:` so it only triggers on "handoff-local" / "without a VM".

The image bakes in the dsh profile from `image/` (bundles: base, headless, agent teams;
model `deepseek-flash`, reasoning low). Edit `image/cordis.patch.yml`, or bump `DSH_VERSION`
with `bin/dsh-vm build --build-arg DSH_VERSION=<version>`, then rebuild.

## Use

In Claude Code: "hand this off" (or `/handoff`). By hand:

```bash
cd your-repo                         # clean tree, .handoff/plan.md written
dsh-vm run "Implement .handoff/plan.md exactly as written."   # prints the run dir first
dsh-vm list
dsh-vm apply ~/.cache/dsh-vm/runs/<run>   # replay changes into the repo (uncommitted)
dsh-vm clean ~/.cache/dsh-vm/runs/<run>   # or: dsh-vm clean --all
dsh-vm build                              # rebuild the image (new dsh, edited profile)
```

| Variable | Default | |
|---|---|---|
| `DSH_VM_MEMORY` | `2g` | VM memory cap (unused memory goes back to the host) |
| `DSH_VM_CPUS` | `2` | CPU-time quota (the guest still sees every core) |
| `DSH_VM_RUNTIME` | `krun` | `crun` runs it as a plain container, for debugging |
| `DSH_VM_IMAGE` | `localhost/dsh-vm:latest` | |
| `DSH_VM_RUNS` | `~/.cache/dsh-vm/runs` | Where run dirs go; `apply`/`clean` refuse anything else |
| `DSH_VM_NPM_CACHE` | `dsh-vm-npm` | Podman volume for npm's download cache (`podman volume rm dsh-vm-npm` to reset) |

## Resources

The VM itself costs ~30 MB. What runs inside costs the same as on the host: dsh ~200 MB, a Vite
build or test run ~500 MB, headless Chromium ~0.5–1 GB. On a 7–8 GB machine run one
browser-heavy handoff at a time.
