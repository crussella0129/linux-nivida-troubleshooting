# RHEL / Rocky / AlmaLinux (and CentOS Stream)

RHEL and its rebuilds (Rocky, AlmaLinux, CentOS Stream) are `dnf`-based like Fedora, but **they do not use RPM Fusion.** You have two supported routes, and they pull from different places than the [Fedora guide](README.md). Everything about Secure Boot, the session trap, and "wait for the module to build" still applies — only the *source* of the driver changes.

> **Pick one route and stick with it. Don't mix ELRepo and the CUDA repo** — you'll get duplicate/conflicting modules.

## Route A — NVIDIA's official CUDA repo (recommended for workstations & ML)

This is NVIDIA's own packaging; it tracks current drivers and is the right choice if you want CUDA anyway.

```bash
# 1. Add EPEL (dependencies) — example for EL9 (RHEL/Rocky/Alma 9):
sudo dnf install epel-release          # on RHEL: enable the codeready-builder repo too
# RHEL only:
# sudo subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms

# 2. Add NVIDIA's CUDA repo (rhel9 / rhel8 as appropriate):
sudo dnf config-manager --add-repo \
  https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo

# 3. Clean & install the driver (DKMS-style auto-rebuild via the 'dkms' module stream):
sudo dnf clean all
sudo dnf module install nvidia-driver:latest-dkms
# (or a pinned branch, e.g. nvidia-driver:570-dkms)

# 4. Optionally the full CUDA toolkit:
sudo dnf install cuda-toolkit
```

The `-dkms` module stream rebuilds the kernel module on kernel updates (needs `kernel-devel` + `dkms`, pulled in as deps). There's also a `latest` (precompiled) stream, but `-dkms` survives kernel upgrades cleanly.

## Route B — ELRepo (`kmod-nvidia`)

ELRepo packages a kABI-tracking `kmod-nvidia` that survives minor-kernel updates without rebuilding.

```bash
# 1. Import key & install ELRepo (EL9 shown):
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo dnf install https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm

# 2. Detect and install the right driver:
sudo dnf install kmod-nvidia          # latest; or nvidia-detect to pick a branch
```

ELRepo's `kmod-nvidia` is kABI-tracking, so a Y-stream kernel update usually keeps working without a rebuild — convenient for servers you don't want recompiling modules.

---

## Everything else is the same as Fedora

| Concern | RHEL/Rocky/Alma | Reference |
|---------|------------------|-----------|
| **Secure Boot** | On by RHEL default; enroll the MOK key. CUDA-repo `-dkms` and ELRepo both sign with a local key you enroll via `mokutil`. | [Fedora Step 4](README.md#step-4-secure-boot--enroll-the-akmods-signing-key-fedora-defaults-to-sb-on) |
| **Wait for build** | DKMS/kmod must finish before reboot — `modinfo -F version nvidia`. | [Fedora Step 3](README.md#step-3-wait-for-the-kmod-to-build--do-not-reboot-yet) |
| **KMS / kernel args** | `sudo grubby --update-kernel=ALL --args="nvidia-drm.modeset=1 rd.driver.blacklist=nouveau"` | [Fedora Step 5](README.md#step-5-kms--kernel-arguments) |
| **Session trap** | GNOME on Xorg vs Wayland; `/etc/gdm/custom.conf` `WaylandEnable=false`. RHEL 9 default DE is GNOME/GDM. | [Fedora Step 7](README.md#step-7-the-session-trap-still-applies) |
| **Recovery** | Same runbook; substitute `dracut --force` for initramfs rebuild. | [recovery.md](recovery.md) |

### Secure Boot signing note (CUDA repo / ELRepo)

If `mokutil --sb-state` is enabled and the module won't load:

```bash
# DKMS on EL signs with a generated key; enroll it:
sudo mokutil --import /var/lib/dkms/mok.pub        # path varies by setup
# or for the NVIDIA/akmods-style key:
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
# reboot → MOK Manager → Enroll MOK → password → reboot
```

If you can't be bothered with MOK on a server, disabling Secure Boot in firmware is acceptable for non-hardened deployments — but on anything security-sensitive, enroll the key and keep Secure Boot on.

---

## Headless / server installs

For compute-only nodes (no display), you don't need the X driver or the session dance:

```bash
# CUDA repo, headless:
sudo dnf module install nvidia-driver:latest-dkms
# verify:
nvidia-smi
```

The "log into X11" advice is moot on a headless box — there's no display manager. Just confirm the module built (`modinfo -F version nvidia`), Secure Boot signing is handled, and `nvidia-smi` returns a table.

---

## Gotchas

1. **No RPM Fusion on RHEL family.** Use the CUDA repo (Route A) or ELRepo (Route B), not `akmod-nvidia` from RPM Fusion.
2. **EPEL + CodeReady/CRB are prerequisites** for many dependencies on RHEL — enable them first or installs fail with unresolved deps.
3. **Don't mix Route A and Route B.** Conflicting `kmod`/`dkms` modules cause load failures.
4. **`-dkms` stream survives kernel upgrades; `latest` (precompiled) can lag a kernel bump.** Prefer `-dkms` on machines that get kernel updates.
5. **Secure Boot is on by default on RHEL** — same MOK enrollment requirement as Fedora.
6. **CentOS Stream tracks ahead of RHEL** — match the repo EL version (el9) to your release, not to "RHEL 9.x" point releases.
