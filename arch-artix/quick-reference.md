# Quick Reference — Arch & Artix NVIDIA

Cheatsheet. Full detail: [README.md](README.md) (Arch), [artix.md](artix.md) (no-systemd), [recovery.md](recovery.md).

## The rules

> 1. **Never partial-upgrade** → always `sudo pacman -Syu`. `nvidia-utils` must match the kernel module.
> 2. **Match the driver to the kernel:** `nvidia`↔`linux`, `nvidia-lts`↔`linux-lts`, `nvidia-dkms` for custom/multiple.
> 3. **X11 first** on Optimus; move to Wayland only after `nvidia_drm.modeset=1` is confirmed.

---

## Driver package picker

| Kernel | Proprietary | Open (Turing/RTX 20+) |
|--------|-------------|------------------------|
| `linux` | `nvidia` | `nvidia-open` |
| `linux-lts` | `nvidia-lts` | — |
| custom / zen / multiple | `nvidia-dkms` | `nvidia-open-dkms` |

---

## Full install (Arch, DKMS path)

```bash
# multilib in /etc/pacman.conf (uncomment [multilib]) then:
sudo pacman -Syu
sudo pacman -S linux-headers                       # headers per kernel (DKMS only)
sudo pacman -S nvidia-dkms nvidia-utils nvidia-settings \
               lib32-nvidia-utils opencl-nvidia cuda nvidia-prime

# /etc/mkinitcpio.conf:
#   MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
#   remove 'kms' from HOOKS
sudo mkinitcpio -P

# KMS for Wayland — add to kernel cmdline:
#   nvidia_drm.modeset=1
sudo grub-mkconfig -o /boot/grub/grub.cfg          # GRUB; or edit systemd-boot entry

# enable display manager
sudo systemctl enable --now sddm                   # Artix: rc-service / sv / dinitctl
sudo reboot
# → pick the X11 session at login
```

## Pacman hook (prebuilt `nvidia`/`nvidia-open` only — NOT for -dkms)

`/etc/pacman.d/hooks/nvidia.hook` rebuilding initramfs on `nvidia`/`linux` upgrades — see [README.md Step 5](README.md#step-5-the-pacman-hook-only-for-the-prebuilt-nvidia--nvidia-open-packages).

---

## Optimus / hybrid

```bash
sudo pacman -S nvidia-prime          # provides prime-run
prime-run <command>                  # run one app on the dGPU
# full switching: optimus-manager or envycontrol (AUR)
```
CUDA/compute needs no env vars in any mode.

## Verify

```bash
nvidia-smi
echo $XDG_SESSION_TYPE                              # x11
cat /sys/module/nvidia_drm/parameters/modeset       # Y
glxinfo | grep "OpenGL renderer"                    # Intel (hybrid)
prime-run glxinfo | grep "OpenGL renderer"          # NVIDIA
python -c "import torch; print(torch.cuda.is_available())"   # True
```

---

## Artix service commands (no systemctl)

| Action | OpenRC | runit | dinit |
|--------|--------|-------|-------|
| Enable DM | `rc-update add sddm default` | symlink to `/run/runit/service` | `dinitctl enable sddm` |
| Start | `rc-service sddm start` | `sv up sddm` | `dinitctl start sddm` |
| Restart | `rc-service sddm restart` | `sv restart sddm` | `dinitctl restart sddm` |

Logs: `dmesg`, `/var/log/Xorg.0.log`, DM logs — **not** `journalctl`. Suspend needs `nvidia-sleep.sh` wired into elogind hooks + `NVreg_PreserveVideoMemoryAllocations=1`. See [artix.md](artix.md).

---

## Emergency one-liners

```bash
# Partial-upgrade fix (the #1 Arch black-screen)
sudo pacman -Syu && sudo mkinitcpio -P && sudo reboot

# Temporary blacklist
echo "blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_uvm
blacklist nvidia_modeset" | sudo tee /etc/modprobe.d/blacklist-nvidia-temporary.conf
sudo mkinitcpio -P && sudo reboot

# GRUB one-shot (press 'e', append to linux line):
nomodeset modprobe.blacklist=nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset

# Chroot rescue from ISO
arch-chroot /mnt        # artix-chroot on Artix ISO
pacman -Syu && mkinitcpio -P

# DKMS retry
sudo dkms autoinstall && cat /var/lib/dkms/nvidia/*/build/make.log
```
