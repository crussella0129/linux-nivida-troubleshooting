# Fedora & RHEL — NVIDIA Troubleshooting

`dnf`/`rpm`-based. Fedora ships **recent drivers** (via RPM Fusion) and uses **akmods** (auto-rebuild kernel modules, akin to DKMS). Two Fedora-specific traps dominate here:

1. **You must wait for the akmod to finish building before you reboot.** Reboot too early and the module isn't ready → you boot on nouveau or black-screen. This is the install-order rule in its sharpest form.
2. **Secure Boot is ON by default on Fedora.** An unsigned kmod won't load. You must enroll the akmods signing key (MOK) or disable Secure Boot — otherwise `nvidia-smi` fails after a clean install.

RHEL / Rocky / Alma don't use RPM Fusion — they go through NVIDIA's CUDA repo or ELRepo. See [rhel.md](rhel.md).

## The principles

> - **Order: enable RPM Fusion → install akmod → WAIT for the build → enroll MOK (if Secure Boot) → reboot.**
> - **NVIDIA + Wayland on Optimus still black-screens.** Fedora's driver is new enough that Wayland *can* work, but on a hybrid laptop, log into **X11 (GNOME on Xorg)** first when anything misbehaves.

## Contents

| File | What it's for |
|------|---------------|
| [README.md](README.md) (this file) | **Main Fedora guide** — RPM Fusion, akmods, Secure Boot signing, Optimus. |
| [rhel.md](rhel.md) | RHEL / Rocky / AlmaLinux — EPEL + ELRepo or NVIDIA's CUDA repo (no RPM Fusion). |
| [recovery.md](recovery.md) | Emergency runbook — black screen / no login right now. |
| [quick-reference.md](quick-reference.md) | One-page cheatsheet. |

> ### 📌 Canonical, always-current sources
> Fedora moves fast (new kernels mid-release) — **when this guide and RPM Fusion's docs disagree, trust RPM Fusion.** This guide adds the wait-for-akmod and Optimus specifics.
>
> - **[RPM Fusion: Howto/NVIDIA](https://rpmfusion.org/Howto/NVIDIA)** — the authoritative Fedora procedure (akmod, Secure Boot, CUDA)
> - **[RPM Fusion: Howto/Secure Boot](https://rpmfusion.org/Howto/Secure%20Boot)** — current MOK signing steps
> - **[NVIDIA CUDA on RHEL/Rocky/Alma](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)** — the RHEL-family driver/CUDA repo
> - *Last reconciled: 2026-05-21.*

---

## Step 0: Verify hardware & state

```bash
cat /etc/fedora-release                  # confirm Fedora version
lspci | grep -i -E "vga|3d|display"      # Intel + NVIDIA present? (Optimus)
mokutil --sb-state                       # Secure Boot status — Fedora defaults to ENABLED
echo $XDG_SESSION_TYPE                    # wayland by default on Fedora
uname -r
```

If `mokutil --sb-state` says **enabled**, you *will* need Step 4 (key enrollment) or the module won't load.

---

## Step 1: Enable RPM Fusion (free + nonfree)

The NVIDIA driver lives in **rpmfusion-nonfree**. Enable both:

```bash
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

Refresh metadata:

```bash
sudo dnf makecache
```

> RPM Fusion's `akmod-nvidia` package handles the nouveau blacklist for you (it drops the modprobe + kernel-arg config). You don't blacklist nouveau by hand on Fedora.

---

## Step 2: Install the NVIDIA stack (akmod)

```bash
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda
```

What you get:

| Package | Purpose |
|---------|---------|
| `akmod-nvidia` | Driver + the **akmods** machinery that auto-rebuilds the kernel module on every kernel update |
| `xorg-x11-drv-nvidia-cuda` | CUDA libraries + `nvidia-smi` (pulls the matching userspace) |

For 32-bit GL (Steam/Wine), add:

```bash
sudo dnf install xorg-x11-drv-nvidia-libs.i686
```

---

## Step 3: WAIT for the kmod to build — do NOT reboot yet

This is the Fedora trap. `akmods` builds the kernel module **in the background after install**. Rebooting before it finishes lands you on nouveau or a black screen.

Force the build now and watch it:

```bash
sudo akmods --force            # build the module immediately
sudo dracut --force            # regenerate initramfs with the new module
```

Confirm the module is actually built for your running kernel:

```bash
modinfo -F version nvidia      # prints the driver version (e.g. 570.xx) when ready
```

If `modinfo` errors with "module not found", the build hasn't finished or failed — **do not reboot.** Check `/var/cache/akmods/nvidia/` for a `.log` and see [recovery.md → akmod build](recovery.md#akmod--kmod-build-failed).

A typical build takes 2–5 minutes. The general rule: after a fresh install, **wait ~5 minutes or until `modinfo -F version nvidia` prints a version** before rebooting.

---

## Step 4: Secure Boot — enroll the akmods signing key (Fedora defaults to SB on)

If `mokutil --sb-state` showed **enabled**, the akmod-built module is signed with a locally-generated key that the firmware doesn't trust yet. Enroll it:

```bash
sudo kmodgenca -a                                  # ensure the akmods CA key exists
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
# set a one-time password when prompted
```

Reboot once now — the blue **MOK Manager** (Perform MOK Management) screen appears:

1. **Enroll MOK** → **Continue** → **Yes**
2. Enter the password you just set
3. **Reboot**

After enrollment, the signed module loads on every boot. If you skip this with Secure Boot on, `nvidia-smi` reports "couldn't communicate with the NVIDIA driver" and `modprobe nvidia` says `Required key not available`.

> Alternative: disable Secure Boot in BIOS (F2 at the Alienware logo → Security → Secure Boot → Disabled). Simpler, but enrolling the key is the cleaner, more secure path and aligns with keeping Secure Boot on.

---

## Step 5: KMS / kernel arguments

RPM Fusion sets the needed args automatically, but to be explicit (and to enable Wayland), make sure `nvidia-drm.modeset=1` is present and nouveau is blacklisted:

```bash
sudo grubby --update-kernel=ALL --args="nvidia-drm.modeset=1 rd.driver.blacklist=nouveau modprobe.blacklist=nouveau"
```

`grubby` edits the bootloader entries directly (works for GRUB and systemd-boot/BLS). On Fedora you generally **don't** hand-edit `/etc/default/grub` + `grub2-mkconfig` for this — `grubby` is the supported tool.

Verify after reboot:

```bash
cat /sys/module/nvidia_drm/parameters/modeset    # Y
```

---

## Step 6: Reboot

```bash
sudo reboot
```

(If you enrolled a MOK key in Step 4, you already rebooted through MOK Manager once — reboot again normally now.)

---

## Step 7: The session trap (still applies)

Fedora's default is **GNOME on Wayland (GDM)**. Modern Fedora *enables* the Wayland session even with NVIDIA, and with a 550+ driver it often works — but on an **Optimus laptop** it can still black-screen or glitch. For first setup:

1. At the GDM login screen, click your name, click the **gear** (bottom-right)
2. Pick **"GNOME on Xorg"** (X11), not "GNOME" (Wayland)
3. Log in

To bias GDM toward X11 system-wide:

```bash
sudo nano /etc/gdm/custom.conf
```
```ini
[daemon]
WaylandEnable=false
```
then `sudo systemctl restart gdm` (or reboot).

> Older guidance to delete `/usr/lib/udev/rules.d/61-gdm.rules` is mostly obsolete on current Fedora — prefer the session selector or `WaylandEnable=false`.

---

## Step 8: Verify

```bash
nvidia-smi                                       # GPU table — driver alive
echo $XDG_SESSION_TYPE                             # x11 (until you choose Wayland)
modinfo -F version nvidia                        # driver version
cat /sys/module/nvidia_drm/parameters/modeset     # Y if KMS on
glxinfo | grep "OpenGL renderer"                 # Intel (correct in hybrid)
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"  # NVIDIA
nvidia-smi -q | grep -i driver                   # confirm version
python3 -c "import torch; print(torch.cuda.is_available())"   # True
```

(`glxinfo` from `glx-utils`/`mesa-demos`; `vulkaninfo` from `vulkan-tools`.)

---

## Optimus / hybrid graphics on Fedora

RPM Fusion's driver supports **PRIME render offload** out of the box — Intel drives the display, NVIDIA renders on request:

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <command>
```

GNOME adds a right-click **"Launch using Discrete Graphics Card"** entry on app icons. There's no `prime-select` on Fedora; offload is the model. CUDA/compute uses the dGPU automatically with no env vars.

For laptops, also install the power-management bits so the dGPU sleeps when idle (usually pulled in by `akmod-nvidia` via `nvidia-powerd`):

```bash
sudo systemctl enable --now nvidia-powerd        # dynamic boost / power mgmt
systemctl status nvidia-suspend nvidia-resume nvidia-hibernate   # VRAM-preserve services (Fedora ships these)
```

---

## Wayland on Fedora

Fedora's 550+ driver is above the 555-ish stability floor, so Wayland is genuinely usable — provided `nvidia-drm.modeset=1` is set (Step 5). To switch: confirm `modeset` is `Y`, remove any `WaylandEnable=false`, and pick the Wayland session at GDM. Optimus laptops may still see flicker/XWayland sync issues; X11 remains the fallback. Same conclusion as the other distros — newer driver, same caution on hybrid hardware.

## Fedora Atomic (Silverblue / Kinoite)

These use `rpm-ostree`, not `dnf`. The flow differs: layer the driver with `rpm-ostree install akmod-nvidia` (or use the `nvidia` packages from a layered RPM Fusion), reboot into the new deployment, and signing/akmods work through ostree. This guide targets traditional Fedora Workstation; for Atomic see the RPM Fusion "Howto/NVIDIA" ostree section.

---

## Lessons Learned

1. **Wait for the akmod build before rebooting.** `sudo akmods --force && sudo dracut --force`, then confirm `modinfo -F version nvidia` prints a version. Rebooting early is the #1 Fedora black-screen. This *is* the install-order rule.
2. **Secure Boot is on by default — enroll the MOK key.** `mokutil --import /etc/pki/akmods/certs/public_key.der` + MOK Manager. Skipping it = `Required key not available`, module won't load.
3. **`akmod-nvidia`, not `kmod-nvidia`, for most people.** `akmod-*` rebuilds automatically on kernel updates (DKMS-like); the plain `kmod-*` is pinned to one kernel and goes stale.
4. **Use `grubby` for kernel args**, not hand-edited GRUB configs — it's the Fedora-supported path and handles BLS entries.
5. **No `prime-select` on Fedora.** Hybrid is PRIME offload via env vars (or GNOME's right-click). Don't look for the Ubuntu tooling.
6. **When a session black-screens, switch to X11 (GNOME on Xorg) first** — cheapest fix, same as every other distro on Optimus.
7. **Don't blacklist nouveau by hand** — RPM Fusion's package already does it. Manual blacklists are for recovery only.
