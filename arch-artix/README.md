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

**First decide open vs proprietary modules, then match the package to your kernel.**

Since the NVIDIA 560 driver series, the **open kernel modules are the default and the recommended flavor for Turing (RTX 20-series) and newer** — and Arch's own `nvidia` package now ships the *open* modules, not the proprietary ones. So on a modern card (RTX 20/30/40-series), the open variant is what you want; the proprietary modules are now only for **legacy Maxwell / Pascal / Volta** GPUs that the open modules don't support.

| Installed kernel | Modern GPU (Turing+, RTX 20-series and newer) — **open** | Legacy GPU (Maxwell/Pascal/Volta) — proprietary |
|------------------|----------------------------------------------------------|--------------------------------------------------|
| `linux` (default) | `nvidia-open` | `nvidia` |
| `linux-lts` | `nvidia-open` *(DKMS)* or `nvidia-lts` build | `nvidia-lts` |
| `linux-zen`, `linux-hardened`, custom, **or more than one** | `nvidia-open-dkms` | `nvidia-dkms` |

- **`nvidia-open` / `nvidia-open-dkms`** — the **default for Turing+**; same CUDA/Vulkan/OpenGL/X11 support as proprietary, built from the same source. `nvidia-open` is prebuilt for `linux`; `nvidia-open-dkms` rebuilds for any kernel (needs `*-headers`).
- **`nvidia` / `nvidia-dkms` (proprietary)** — required for **Maxwell/Pascal/Volta**; the open modules don't support them. Note Arch's `nvidia` package itself moved to open modules, so "proprietary" now specifically means the legacy-capable closed build.
- **Match the kernel:** prebuilt packages (`nvidia`, `nvidia-open`, `nvidia-lts`) track a specific kernel; the `*-dkms` variants rebuild for any kernel. Run zen/hardened/custom or multiple kernels → use a `-dkms` package.

> Throughout this guide, substitute your chosen package name. Examples use `nvidia-open-dkms` (the modern default) — on a legacy GPU read those as `nvidia-dkms`.

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

### Multi-kernel pattern (prebuilt modules instead of DKMS)

If you run **both** `linux` and `linux-lts` (a common belt-and-suspenders setup so you always have a known-good fallback kernel), install the matching prebuilt module for *each* instead of using DKMS — this is what the [LogOS](#confirmed-in-logos) installer does:

```bash
sudo pacman -S nvidia nvidia-lts nvidia-utils nvidia-settings
#              ^module for `linux`   ^module for `linux-lts`
```

`nvidia-utils` is shared; each kernel gets its own prebuilt module package. This avoids DKMS build time entirely but **requires the pacman hook in Step 5** to keep the initramfs in sync on upgrades. Choose this *or* `nvidia-dkms` — not both.

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

> **Two valid approaches — know which you're using.** This guide early-loads the NVIDIA modules via `MODULES=(...)` and drops the `kms` hook, which is the Arch Wiki recommendation and is **the more robust path for Wayland / wlroots compositors**.
>
> The [LogOS](#confirmed-in-logos) installer takes the other valid approach: it **keeps the `kms` hook**, leaves `MODULES` minimal (just the root-fs module, e.g. `btrfs`), and relies on `nvidia_drm.modeset=1` (Step 6) plus the early-KMS hook to bring up the GPU. That works fine for X11 and KDE/SDDM. If you go that route, *don't also* add the nvidia modules to `MODULES` — pick one. For a Wayland-first laptop, prefer early-loading the modules.

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

### wlroots compositors (Hyprland / Sway) — required NVIDIA env vars

Plain KDE/GNOME on Wayland mostly works once `modeset=1` is set. **wlroots-based compositors (Hyprland, Sway) need extra NVIDIA environment variables**, or you get an invisible/garbled cursor, a black screen, or a compositor that won't start. The [LogOS](#confirmed-in-logos) installer writes these for Hyprland — set them globally via `/etc/environment.d/`:

```bash
sudo mkdir -p /etc/environment.d
sudo tee /etc/environment.d/90-nvidia-wayland.conf >/dev/null <<'EOF'
LIBVA_DRIVER_NAME=nvidia
__GLX_VENDOR_LIBRARY_NAME=nvidia
WLR_NO_HARDWARE_CURSORS=1
EOF
```

| Variable | Why |
|----------|-----|
| `WLR_NO_HARDWARE_CURSORS=1` | **The big one** — fixes the invisible / flickering hardware cursor on NVIDIA + wlroots. |
| `__GLX_VENDOR_LIBRARY_NAME=nvidia` | Routes GLX through the NVIDIA implementation. |
| `LIBVA_DRIVER_NAME=nvidia` | Use NVIDIA's VA-API driver for hardware video decode. |

Commonly also needed on older setups (add if the compositor still won't start): `GBM_BACKEND=nvidia-drm`. On driver 545+ with `modeset=1` it's usually unnecessary, and setting it can break some Electron/Chromium apps — add only if required.

> `/etc/environment.d/` is read by systemd user sessions. On **Artix** (no systemd) put these in `/etc/environment` (or your compositor's launch env / `~/.config/uwsm/env`) instead — see [artix.md](artix.md).

---

## Secure Boot

Arch installs usually run with Secure Boot off. If you enable it, the DKMS-built NVIDIA module is unsigned and won't load (`Required key not available`). NVIDIA + DKMS + Secure Boot is a genuinely **fragile combo** — every kernel/driver update must re-sign or you black-screen. Three options, easiest first.

### Option A — Disable Secure Boot (recommended for NVIDIA)

BIOS → Security → Secure Boot → Disabled. The [LogOS](#confirmed-in-logos) guidance is explicit about this: for NVIDIA GPUs, disabling Secure Boot is the pragmatic choice; reserve full Secure Boot for AMD machines. No signing, no per-update breakage.

### Option B — Sign DKMS modules with a MOK key

This makes DKMS auto-sign the module on every rebuild, then you enroll the key once. (Lifted from the LogOS build guide, §11.3.1.)

```bash
sudo pacman -S nvidia-dkms nvidia-utils nvidia-settings

# 1. Generate a signing key (valid 100 years)
sudo openssl req -new -x509 -newkey rsa:2048 \
  -keyout /etc/dkms/mok.key -out /etc/dkms/mok.crt \
  -nodes -days 36500 -subj "/CN=DKMS Signing Key/"

# 2. Tell DKMS to use it
sudo tee /etc/dkms/framework.conf.d/mok-signing.conf >/dev/null <<'EOF'
mok_signing_key="/etc/dkms/mok.key"
mok_certificate="/etc/dkms/mok.crt"
sign_tool="/etc/dkms/sign_helper.sh"
EOF

# 3. Signing helper
sudo tee /etc/dkms/sign_helper.sh >/dev/null <<'EOF'
#!/bin/bash
/usr/bin/kmodsign sha512 /etc/dkms/mok.key /etc/dkms/mok.crt "$2"
EOF
sudo chmod +x /etc/dkms/sign_helper.sh

# 4. Rebuild so the module gets signed, then enroll the key
sudo dkms autoinstall
sudo mokutil --import /etc/dkms/mok.crt   # set a one-time password
sudo reboot                                # → MOK Manager → Enroll MOK → password
```

### Option C — Full Secure Boot with `sbctl` (signs the whole boot chain)

If you also want signed kernels/bootloader (not just the NVIDIA module):

```bash
sudo sbctl create-keys
sudo sbctl enroll-keys --microsoft         # keep Microsoft keys for firmware compat
sudo sbctl sign -s /boot/vmlinuz-linux
sudo sbctl sign -s /boot/vmlinuz-linux-lts
# ...sign the bootloader EFI too
# auto-sign on update via a pacman hook:
sudo tee /etc/pacman.d/hooks/99-sbctl.hook >/dev/null <<'EOF'
[Trigger]
Type = Path
Operation = Install
Operation = Upgrade
Target = boot/vmlinuz-*
Target = usr/lib/modules/*/vmlinuz

[Action]
Description = Signing kernels for Secure Boot...
When = PostTransaction
Exec = /usr/bin/sbctl sign-all
Depends = sbctl
EOF
```

Note `sbctl` signs kernels/EFI binaries; the **NVIDIA DKMS module still needs the MOK signing from Option B** — they're complementary. This is why LogOS recommends AMD if you want full Secure Boot, and disabling Secure Boot if you're on NVIDIA.

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
9. **wlroots (Hyprland/Sway) needs `WLR_NO_HARDWARE_CURSORS=1`** and the NVIDIA GLX/VA-API env vars, or the cursor vanishes / the compositor won't start. Plain KDE/GNOME Wayland doesn't need them.

---

## Confirmed in LogOS

Several pieces of this guide are battle-tested in my **LogOS** Arch installer (an automated, security-hardened Arch build), not just drawn from the wiki. What LogOS does in practice:

- **Driver install** (`lib/desktop.sh`): detects the GPU via `lspci` and installs `nvidia nvidia-utils nvidia-settings nvidia-lts` — the dual prebuilt-module pattern (both `linux` and `linux-lts` kernels), not DKMS. See [Step 3 → multi-kernel pattern](#multi-kernel-pattern-prebuilt-modules-instead-of-dkms).
- **Wayland env vars** (`lib/desktop-hyprland.sh`): writes `/etc/environment.d/logos-nvidia.conf` with `LIBVA_DRIVER_NAME=nvidia`, `__GLX_VENDOR_LIBRARY_NAME=nvidia`, `WLR_NO_HARDWARE_CURSORS=1` whenever an NVIDIA GPU is detected. See [wlroots compositors](#wlroots-compositors-hyprland--sway--required-nvidia-env-vars).
- **mkinitcpio** (`scripts/03-chroot-setup.sh`): keeps the `kms` hook and minimal `MODULES`, relying on `nvidia_drm.modeset=1` rather than early-loading the modules. See the [Step 4 note](#step-4-early-kms--load-the-modules-in-the-initramfs).
- **Secure Boot** (build guide §11): documents NVIDIA + Secure Boot as "fragile / expect pain," recommends **disabling Secure Boot for NVIDIA** (Option A) and the DKMS-MOK-signing flow (Option B) when you must keep it on. Hardware-compat notes rate NVIDIA RTX 3000/4000 as "Fragile — DKMS + Secure Boot."
- **Recovery**: LogOS's troubleshooting appendix uses the same `nomodeset` GRUB edit → `pacman -S nvidia-dkms && mkinitcpio -P` path documented in [recovery.md](recovery.md).

> Source repo: `~/LogOS-Arch` (also `~/LogOS`). The NVIDIA logic lives in `lib/desktop.sh`, `lib/desktop-hyprland.sh`, `lib/detect.sh`, `scripts/03-chroot-setup.sh`, and `docs/appendices/{troubleshooting,hardware-compat}.md`.
