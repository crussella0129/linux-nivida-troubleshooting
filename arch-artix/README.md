# Arch & Artix — NVIDIA Troubleshooting

Rolling-release, `pacman`-based. Arch ships the **newest** NVIDIA drivers (570+), so Wayland is more viable here than on Debian — but rolling release adds its own failure mode: **partial upgrades that mismatch `nvidia-utils` against the kernel module → black screen.** Artix is Arch without systemd, which changes how you start the display manager and manage suspend.

## The principles that drive everything here

> 1. **Never partial-upgrade.** `pacman -Sy nvidia` (or installing one package against a stale db) leaves `nvidia-utils` out of sync with the kernel module and black-screens you. Always full-upgrade: **`sudo pacman -Syu`**.
> 2. **Pick the driver package that matches your kernel** (`nvidia` for `linux`, `nvidia-lts` for `linux-lts`, `nvidia-dkms` for anything else or multiple kernels).
> 3. **Order still matters, and Wayland on Optimus still bites.** If a session black-screens, log into **X11** first before changing anything.

## Contents

| File | What it's for |
|------|---------------|
| [README.md](README.md) (this file) | **Main Arch guide** — full install + KMS + Optimus walkthrough. |
| [artix.md](artix.md) | Artix differences — OpenRC / runit / s6 / dinit service commands, no `journalctl`, suspend handling. |
| [recovery.md](recovery.md) | Emergency runbook — black screen / no login right now. |
| [quick-reference.md](quick-reference.md) | One-page cheatsheet. |

---

## Step 0: Know your kernel and pick the right driver package

This is the Arch-specific decision that everything else depends on.

```bash
pacman -Q linux linux-lts linux-zen linux-hardened 2>/dev/null   # which kernel(s) installed?
uname -r
```

| Installed kernel | NVIDIA package to use |
|------------------|------------------------|
| `linux` (default) | `nvidia` |
| `linux-lts` | `nvidia-lts` |
| `linux-zen`, `linux-hardened`, custom, **or more than one** | `nvidia-dkms` |

- **`nvidia`** ships a prebuilt module matched to the current `linux` package — fastest, no build step, but you must keep them upgraded together.
- **`nvidia-dkms`** rebuilds the module for any kernel via DKMS — the safe choice if you run zen/lts/multiple kernels. Needs the matching `*-headers` package.
- **Open kernel modules:** `nvidia-open` (matched to `linux`) or `nvidia-open-dkms`. **Recommended by NVIDIA for Turing (RTX 20-series) and newer.** For an RTX 40-series Alienware, `nvidia-open-dkms` is a good default; older Maxwell/Pascal GPUs must use the proprietary `nvidia`/`nvidia-dkms`.

> Throughout this guide, substitute your chosen package name. Examples use `nvidia-dkms` because it's the most robust for a laptop you'll keep upgrading.

---

## Step 1: Enable multilib (for 32-bit / Steam / Wine)

Edit `/etc/pacman.conf` and uncomment:

```ini
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Then sync:

```bash
sudo pacman -Syu
```

(Skip multilib only if you will never run 32-bit GL apps. Most desktops want it.)

---

## Step 2: Install headers FIRST (only needed for the `*-dkms` packages)

Same lesson as Debian: DKMS needs headers present to build.

```bash
sudo pacman -S linux-headers          # match each installed kernel:
# sudo pacman -S linux-lts-headers
# sudo pacman -S linux-zen-headers
```

If you chose the non-DKMS `nvidia` or `nvidia-open` package (prebuilt module), you can skip headers — but installing them doesn't hurt.

---

## Step 3: Install the NVIDIA stack

```bash
sudo pacman -S nvidia-dkms nvidia-utils nvidia-settings \
               lib32-nvidia-utils \
               opencl-nvidia cuda
```

What each does:

| Package | Purpose |
|---------|---------|
| `nvidia-dkms` (or `nvidia`/`nvidia-open-dkms`) | The kernel module |
| `nvidia-utils` | Userspace driver libs, `nvidia-smi`, the X driver, GL/Vulkan ICDs |
| `lib32-nvidia-utils` | 32-bit userspace libs (Steam, Wine) — needs multilib |
| `nvidia-settings` | GUI control panel |
| `opencl-nvidia` | OpenCL runtime |
| `cuda` | Full CUDA toolkit + `nvcc` (large; skip if you only use pip-installed PyTorch) |

> **Critical Arch rule:** `nvidia-utils` and the kernel-module package **must be the same version**. `pacman -Syu` guarantees this. A `pacman -Sy nvidia-utils` (sync db but don't upgrade everything) does **not**, and is a classic black-screen cause. See [Lessons](#lessons-learned).

---

## Step 4: Early KMS — load the modules in the initramfs

This is the Arch equivalent of the work Debian's installer does silently. It's required for a clean boot and **mandatory for Wayland**.

Edit `/etc/mkinitcpio.conf`:

```bash
sudo nano /etc/mkinitcpio.conf
```

1. Add the NVIDIA modules to the `MODULES` array so they load early:
   ```
   MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
   ```
2. **Remove `kms` from the `HOOKS` array** if present — the `kms` hook pulls nouveau into the initramfs, which fights the proprietary driver:
   ```
   HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block filesystems fsck)
   #                                                  ^^^ remove this 'kms'
   ```

Rebuild the initramfs:

```bash
sudo mkinitcpio -P
```

---

## Step 5: The pacman hook (only for the prebuilt `nvidia` / `nvidia-open` packages)

The prebuilt (non-DKMS) module can fall out of sync when the kernel updates but the initramfs isn't regenerated, producing a black screen on the next boot. A pacman hook fixes this automatically. **DKMS packages don't need this** — DKMS handles rebuilds itself.

```bash
sudo nano /etc/pacman.d/hooks/nvidia.hook
```

```ini
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia
Target=linux
# add a Target line for each kernel/driver you use, e.g. nvidia-open, linux-lts

[Action]
Description=Update NVIDIA module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux*) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```

---

## Step 6: Enable DRM KMS modesetting (required for Wayland; recommended generally)

Add the kernel parameter `nvidia_drm.modeset=1`. Modern drivers (545+) default this on, but set it explicitly.

**GRUB:** edit `/etc/default/grub`, append to `GRUB_CMDLINE_LINUX_DEFAULT`:
```
nvidia_drm.modeset=1
```
then `sudo grub-mkconfig -o /boot/grub/grub.cfg`.

**systemd-boot:** add `nvidia_drm.modeset=1` to the `options` line in `/boot/loader/entries/*.conf`.

> Note: `grub-mkconfig` regenerates the GRUB config and is **unrelated** to `/etc/pacman.conf` edits — different subsystems, same as the Debian guide's note about `update-grub` vs `sources.list`.

---

## Step 7: Optimus / hybrid graphics

Three approaches, simplest first:

### A. PRIME render offload (no extra daemon — recommended)

Intel drives the display; render specific apps on NVIDIA on demand. Install the wrapper:

```bash
sudo pacman -S nvidia-prime    # provides the `prime-run` wrapper (yes, this package exists on Arch)
prime-run <command>
# equivalent to:
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <command>
```

### B. `optimus-manager` (AUR) — full GPU switching with a daemon

```bash
# via an AUR helper, e.g. paru/yay
paru -S optimus-manager
sudo systemctl enable --now optimus-manager   # (Artix: see artix.md)
optimus-manager --switch hybrid               # hybrid | integrated | nvidia
```

Conflicts with manually-written Xorg GPU config — let it manage Xorg. Reboot/relogin after switching.

### C. `envycontrol` (AUR) — simpler one-shot switcher

```bash
paru -S envycontrol
sudo envycontrol -s hybrid     # integrated | hybrid | nvidia
```

For a laptop you mostly want **hybrid / on-demand**. CUDA compute works in all modes without env vars.

---

## Step 8: Display manager & the session trap

Arch ships current drivers, so Wayland *can* work — but the same Optimus failure mode from the Debian guide applies. **On first setup, log into X11.**

- **SDDM (KDE):** at the login screen, pick **"Plasma (X11)"** from the session selector before logging in. To bias the default:
  ```bash
  sudo mkdir -p /etc/sddm.conf.d
  printf '[General]\nDisplayServer=x11\n' | sudo tee /etc/sddm.conf.d/10-x11.conf
  ```
- **GDM (GNOME):** historically GDM masks its own Wayland session when the NVIDIA driver is present (via a udev rule). If GDM itself won't start on Wayland, that udev rule may need to be present:
  ```bash
  ls /usr/lib/udev/rules.d/61-gdm.rules    # ships with gdm; don't delete it
  ```
  At login, pick **"GNOME on Xorg."** To force X11: edit `/etc/gdm/custom.conf` → `[daemon]` → `WaylandEnable=false`.

Enable the DM service (systemd shown; **Artix uses different commands — see [artix.md](artix.md)**):

```bash
sudo systemctl enable --now sddm    # or gdm
```

---

## Step 9: Reboot & verify

```bash
sudo reboot
```

```bash
nvidia-smi                                   # GPU table
echo $XDG_SESSION_TYPE                         # x11 (until you intentionally move to Wayland)
cat /sys/module/nvidia_drm/parameters/modeset  # Y if KMS enabled
glxinfo | grep "OpenGL renderer"             # Intel (correct in hybrid)
prime-run glxinfo | grep "OpenGL renderer"   # NVIDIA
nvcc --version                               # if you installed cuda
python -c "import torch; print(torch.cuda.is_available())"   # True
```

`mkinitcpio` warnings about possibly-missing firmware are usually harmless; an actual failure is an error, not a warning.

---

## Wayland on Arch

Because Arch ships 570+ (well above the 555 floor that blocks Debian), Wayland is genuinely usable on Arch — *if* you have `nvidia_drm.modeset=1` (Step 6) and a current driver. To move:

1. Confirm `cat /sys/module/nvidia_drm/parameters/modeset` prints `Y`.
2. Remove any `DisplayServer=x11` / `WaylandEnable=false` overrides.
3. At the login screen pick the **Wayland** session.

Optimus laptops can still see flicker / XWayland sync issues. If anything misbehaves, drop back to X11 — it remains the most stable path. (Same conclusion as the Debian guide, just with a newer driver floor.)

---

## Secure Boot

Arch installs usually run with Secure Boot off. If you enable it (e.g. with `sbctl`), the DKMS-built module is unsigned and won't load (`Required key not available`). Options:

- Disable Secure Boot in BIOS (simplest), **or**
- Sign the kernel + modules yourself: set up `sbctl`, enroll keys, and sign on each rebuild (a DKMS sign hook). This is more involved than Debian's `update-secureboot-policy`; see the Arch wiki "Unified Extensible Firmware Interface/Secure Boot" if you need it.

---

## Lessons Learned

1. **Partial upgrades are the #1 Arch black-screen cause.** `nvidia-utils` must match the running kernel module exactly. Never `pacman -Sy <pkg>` a single package — always `pacman -Syu` the whole system. If you see "API mismatch" in Xorg logs or `nvidia-smi` reports a version mismatch, you partial-upgraded.
2. **Match the driver package to the kernel.** `nvidia` ↔ `linux`, `nvidia-lts` ↔ `linux-lts`, `nvidia-dkms` for anything custom or multiple kernels. Mismatch = module won't load after the next kernel update.
3. **The pacman hook is only for the prebuilt packages.** `*-dkms` rebuilds itself; don't double up.
4. **`nvidia_drm.modeset=1` is mandatory for Wayland** and harmless for X11. Verify via `/sys/module/nvidia_drm/parameters/modeset`.
5. **Don't blacklist nouveau by hand and also load nvidia early** — putting the nvidia modules in `MODULES` and dropping the `kms` hook is the clean way; manual nouveau blacklists are only for recovery.
6. **`grub-mkconfig` ≠ `pacman.conf`.** Bootloader config and package config are separate subsystems (mirrors the Debian `update-grub` note).
7. **Optimus laptops: prefer PRIME offload (`prime-run`)** over a switching daemon unless you specifically need full nvidia-only mode. Less to break.
8. **When a session black-screens, switch to X11 first.** Cheapest fix, every time — the rolling-release newness doesn't change that on Optimus.
