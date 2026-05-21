# Emergency Recovery — Arch & Artix

> **Screen is black right now.** Work top to bottom; stop when you have a desktop. Artix users: swap `systemctl`/`journalctl` for your init's commands (see [artix.md](artix.md)).

## The two facts that solve most Arch cases

1. **Black after login on an Optimus laptop?** You're in Wayland. Reboot → pick **X11** at the login screen.
2. **Broke right after a `pacman` run?** You probably **partial-upgraded** — `nvidia-utils` no longer matches the kernel module. Fix: full upgrade `sudo pacman -Syu`, then rebuild initramfs and reboot.

---

## Decision tree

```
Black after login (login screen was fine)
   └─> Wayland session. Reboot → pick X11/Xorg.                 ✅ most common

Broke right after pacman -Sy / -S <single pkg>
   └─> Partial upgrade. Chroot or TTY → pacman -Syu.            ✅ 2nd most common

Black after a kernel update
   └─> initramfs stale / module mismatch → mkinitcpio -P.

Ctrl+Alt+F3 gives a text login
   └─> diagnose below.

Totally frozen, no TTY
   └─> GRUB one-shot rescue.
```

---

## TTY works — diagnose

`Ctrl+Alt+F3` (try F2–F6), log in:

```bash
nvidia-smi                                       # version mismatch? driver alive?
cat /sys/module/nvidia_drm/parameters/modeset    # Y = KMS on
pacman -Q nvidia-utils nvidia-dkms linux linux-headers   # versions consistent?
dmesg | grep -i -E 'nvidia|nouveau|drm'          # kernel-side errors
cat /var/log/Xorg.0.log | grep -i -E 'EE|error'  # X errors
lsmod | grep nvidia                              # module loaded?
```

Interpretation:

- **`nvidia-smi`: "Failed to initialize NVML: Driver/library version mismatch"** → classic partial upgrade. Run `sudo pacman -Syu`, then `sudo mkinitcpio -P`, reboot.
- **`nvidia-smi` works, login still black** → session problem, reboot and pick X11.
- **`modprobe nvidia` → `Required key not available`** → Secure Boot on + unsigned module. Disable Secure Boot in BIOS or sign the module.
- **`modeset` prints `N` and you wanted Wayland** → add `nvidia_drm.modeset=1` (see main guide Step 6).

### Fix a partial upgrade (the big one)

```bash
sudo pacman -Syu          # bring the WHOLE system in sync
sudo mkinitcpio -P        # rebuild initramfs with matched modules
sudo reboot
```

If you're offline/mirrors are broken and can't full-upgrade, downgrade the mismatched piece to the other's version with `downgrade` (AUR) or from the pacman cache in `/var/cache/pacman/pkg/` — but `-Syu` is the real fix.

### Get a desktop now (non-destructive blacklist)

```bash
echo "blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_uvm
blacklist nvidia_modeset" | sudo tee /etc/modprobe.d/blacklist-nvidia-temporary.conf
sudo mkinitcpio -P
sudo reboot
```
Undo by removing that file and `sudo mkinitcpio -P` again.

### Restart the display manager

```bash
# systemd (Arch):
sudo systemctl restart sddm        # or gdm
# Artix — pick your init (see artix.md):
sudo rc-service sddm restart       # OpenRC
sudo sv restart sddm               # runit
sudo dinitctl restart sddm         # dinit
```
Pick the **X11** session before logging in.

---

## GRUB one-shot rescue — totally frozen

1. Hard power off (hold ~10s), power on, tap **Esc**/hold **Shift** for the GRUB menu.
2. Highlight the entry, press **`e`**.
3. On the `linux` line, append:
   ```
   nomodeset modprobe.blacklist=nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset
   ```
   (Add `3` on Arch/systemd for text-only; on Artix use `single` or just don't enable the DM.)
4. **`Ctrl+X`** / **F10** to boot. One-shot only.

---

## Chroot rescue — system won't boot at all

Boot the Arch/Artix ISO, then:

```bash
# identify and mount your root (and /boot, /efi as applicable)
lsblk
mount /dev/nvmeXnYpZ /mnt
mount /dev/nvmeXnYpW /mnt/boot      # if separate
arch-chroot /mnt                    # (Artix ISO: artix-chroot /mnt)

# now fix from inside:
pacman -Syu                         # resolve partial upgrade
mkinitcpio -P                       # rebuild initramfs
# verify driver/kernel match:
pacman -Q nvidia-utils nvidia-dkms linux
exit
reboot
```

---

## DKMS build failed

```bash
uname -r
pacman -Q linux-headers linux-lts-headers 2>/dev/null   # headers for your kernel present?
sudo dkms status
sudo dkms autoinstall
cat /var/lib/dkms/nvidia/*/build/make.log               # the real error
```

- Missing headers → `sudo pacman -S linux-headers` (match each kernel), retry.
- Kernel newer than driver supports → boot the LTS kernel, or wait for the driver to catch up (rolling release usually fixes within days).

---

## Last resort — remove NVIDIA, boot on nouveau

```bash
sudo pacman -Rns nvidia-dkms nvidia-utils lib32-nvidia-utils nvidia-settings
sudo rm -f /etc/modprobe.d/nvidia*.conf
# remove the nvidia MODULES entries from /etc/mkinitcpio.conf, then:
sudo mkinitcpio -P
sudo reboot
```

You'll come up on nouveau. Reinstall cleanly later following the [main guide](README.md).

---

## 30-second checklist

1. Black after login → reboot, pick **X11**.
2. Broke after pacman → `sudo pacman -Syu && sudo mkinitcpio -P && reboot`.
3. Need desktop now → temporary blacklist + `mkinitcpio -P` + reboot.
4. Won't boot → ISO + `arch-chroot` → `pacman -Syu` + `mkinitcpio -P`.
5. Frozen → GRUB `e`, append `nomodeset modprobe.blacklist=nvidia,...`.
