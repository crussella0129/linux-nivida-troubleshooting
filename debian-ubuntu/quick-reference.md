# Quick Reference — Debian & Ubuntu NVIDIA

A printable cheatsheet. Full explanations live in [debian-trixie.md](debian-trixie.md) and [ubuntu.md](ubuntu.md). Emergencies: [recovery.md](recovery.md).

## The one rule

> **Order: repos → headers → driver → reboot → pick X11 at login.**
> NVIDIA + Wayland on an Optimus laptop black-screens. When something breaks, log into **X11/Xorg** first.

---

## Debian 13 Trixie — full sequence

```bash
sudo nano /etc/apt/sources.list          # add: contrib non-free  (3 components, watch typos)
sudo apt update
apt-cache policy nvidia-driver           # confirm Candidate: is not (none)
sudo apt install linux-headers-amd64     # HEADERS FIRST, separately
sudo apt install nvidia-driver nvidia-kernel-dkms nvidia-settings \
                 nvidia-cuda-toolkit firmware-misc-nonfree
sudo reboot
# → at SDDM, pick "Plasma (X11)" before logging in
```

## Ubuntu 22.04 / 24.04 — full sequence

```bash
sudo add-apt-repository restricted multiverse
sudo apt update
sudo ubuntu-drivers devices              # see recommended branch
sudo ubuntu-drivers install              # driver + DKMS + prime + headers
sudo prime-select on-demand              # hybrid mode for laptops; reboot after
sudo reboot
# → at GDM, pick "Ubuntu on Xorg" before logging in
```

---

## Verify (both)

```bash
nvidia-smi                               # GPU table — driver alive
echo $XDG_SESSION_TYPE                   # must say: x11
dkms status                              # nvidia/<ver>, <kernel>: installed
glxinfo | grep "OpenGL renderer"         # Intel (correct in hybrid mode)
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"  # NVIDIA
nvcc --version                           # CUDA compiler
python3 -c "import torch; print(torch.cuda.is_available())"   # True
```

## Run an app on the dGPU

```bash
# Debian + Ubuntu:
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <command>
# Ubuntu shortcut:
prime-run <command>
```

CUDA/compute needs **no** env vars — `/dev/nvidia*` is used automatically.

---

## Package / file map: Debian vs Ubuntu

| | Debian 13 | Ubuntu 24.04 |
|---|---|---|
| Non-free repo | `contrib non-free` | `restricted multiverse` |
| Headers | `linux-headers-amd64` | `linux-headers-generic` (auto-pulled) |
| Driver | `nvidia-driver` | `nvidia-driver-XXX` / `ubuntu-drivers install` |
| Hybrid switch | built-in (no package) | `prime-select` (`nvidia-prime`) |
| dGPU launcher | env vars | env vars or `prime-run` |
| Display mgr | SDDM (KDE) | GDM3 (GNOME) |
| Force X11 | `/etc/sddm.conf.d/10-x11.conf` | `/etc/gdm3/custom.conf` |
| Initramfs | `update-initramfs -u` | `update-initramfs -u` |

⚠️ `nvidia-prime` and `prime-select` exist on **Ubuntu only** — not Debian.

---

## Emergency one-liners

```bash
# Temporary blacklist (boot to working desktop on Intel/nouveau)
echo "blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_uvm
blacklist nvidia_modeset" | sudo tee /etc/modprobe.d/blacklist-nvidia-temporary.conf
sudo update-initramfs -u && sudo reboot

# Undo it
sudo rm /etc/modprobe.d/blacklist-nvidia-temporary.conf
sudo update-initramfs -u && sudo reboot

# GRUB one-shot (press 'e' on the boot entry, append to the linux line):
3 nomodeset modprobe.blacklist=nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset

# Restart display manager from TTY
sudo systemctl restart sddm     # Debian/KDE
sudo systemctl restart gdm3     # Ubuntu/GNOME

# Retry DKMS build
sudo dkms autoinstall
sudo cat /var/lib/dkms/nvidia/*/build/make.log
```
