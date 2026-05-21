# Quick Reference — Fedora & RHEL NVIDIA

Cheatsheet. Detail: [README.md](README.md) (Fedora), [rhel.md](rhel.md) (RHEL/Rocky/Alma), [recovery.md](recovery.md).

## The rules

> - **Order: RPM Fusion → akmod → WAIT for build → enroll MOK → reboot.**
> - **Wait for the akmod build** before rebooting (`modinfo -F version nvidia` must print a version).
> - **Secure Boot is ON by default** → enroll the MOK key or the module won't load.
> - **X11 first** on Optimus (GNOME on Xorg).

---

## Fedora — full sequence

```bash
# 1. RPM Fusion (free + nonfree)
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
sudo dnf makecache

# 2. Driver (akmod = auto-rebuild) + CUDA libs
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda

# 3. WAIT for the build — do NOT reboot until this prints a version:
sudo akmods --force && sudo dracut --force
modinfo -F version nvidia

# 4. Secure Boot (if mokutil --sb-state = enabled):
sudo kmodgenca -a
sudo mokutil --import /etc/pki/akmods/certs/public_key.der   # → reboot → MOK Manager → Enroll

# 5. KMS / kernel args
sudo grubby --update-kernel=ALL --args="nvidia-drm.modeset=1 rd.driver.blacklist=nouveau modprobe.blacklist=nouveau"

sudo reboot
# → pick "GNOME on Xorg" at GDM
```

## RHEL / Rocky / Alma — driver source differs (no RPM Fusion)

```bash
# Route A: NVIDIA CUDA repo (EL9)
sudo dnf install epel-release
sudo dnf config-manager --add-repo \
  https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo
sudo dnf module install nvidia-driver:latest-dkms

# Route B: ELRepo
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo dnf install https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm
sudo dnf install kmod-nvidia
```
Secure Boot, KMS args, session trap, recovery: same as Fedora. **Don't mix Route A and B.**

---

## Verify

```bash
nvidia-smi
modinfo -F version nvidia                          # driver version
echo $XDG_SESSION_TYPE                               # x11
cat /sys/module/nvidia_drm/parameters/modeset        # Y
glxinfo | grep "OpenGL renderer"                     # Intel (hybrid)
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"  # NVIDIA
python3 -c "import torch; print(torch.cuda.is_available())"   # True
```

## Optimus — PRIME offload (no prime-select on Fedora)

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <command>
# GNOME: right-click app → "Launch using Discrete Graphics Card"
sudo systemctl enable --now nvidia-powerd            # laptop power mgmt
```
CUDA/compute needs no env vars.

---

## Package map: Fedora vs RHEL family

| | Fedora | RHEL / Rocky / Alma |
|---|---|---|
| Driver source | RPM Fusion nonfree | CUDA repo *or* ELRepo |
| Driver package | `akmod-nvidia` | `nvidia-driver:latest-dkms` / `kmod-nvidia` |
| CUDA libs | `xorg-x11-drv-nvidia-cuda` | `cuda-toolkit` (CUDA repo) |
| Prereq repos | RPM Fusion | EPEL + CodeReady/CRB |
| Rebuild on kernel | akmods (auto) | dkms (auto) / kABI (ELRepo) |
| Initramfs | `dracut --force` | `dracut --force` |
| Kernel args | `grubby` | `grubby` |

⚠️ No `nvidia-prime`/`prime-select` on Fedora/RHEL — hybrid is PRIME offload via env vars or GNOME's right-click.

---

## Emergency one-liners

```bash
# Module not built (the #1 Fedora black-screen)
sudo akmods --force && sudo dracut --force && modinfo -F version nvidia

# Secure Boot rejected the module
sudo mokutil --import /etc/pki/akmods/certs/public_key.der && sudo reboot   # → MOK Manager

# Missing kernel-devel after a kernel bump
sudo dnf install "kernel-devel-$(uname -r)" && sudo akmods --force && sudo dracut --force

# Temporary blacklist
echo "blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_uvm
blacklist nvidia_modeset" | sudo tee /etc/modprobe.d/blacklist-nvidia-temporary.conf
sudo dracut --force && sudo reboot

# GRUB one-shot (press 'e', append to linux line):
3 rd.driver.blacklist=nouveau,nvidia modprobe.blacklist=nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset nomodeset
```
