# Artix Differences (no systemd)

Artix is Arch with a different init system. **The entire NVIDIA install — driver packages, `mkinitcpio`, KMS, PRIME, the session trap — is identical to the [Arch guide](README.md).** What changes is *service management* and *log access*, because there is no `systemd`/`systemctl`/`journalctl`.

Pick the section matching your init system: **OpenRC**, **runit**, **s6**, or **dinit**.

---

## 1. Starting / enabling the display manager

Where the Arch guide says `sudo systemctl enable --now sddm`, do this instead:

### OpenRC
```bash
sudo rc-update add sddm default      # or gdm / lightdm / lxdm
sudo rc-service sddm start
sudo rc-service sddm status
```
The DM package on Artix is usually suffixed with the init, e.g. `sddm-openrc`, `gdm-openrc`, `elogind-openrc`. Install the matching `*-openrc` service package alongside the DM.

### runit
```bash
sudo ln -s /etc/runit/sv/sddm /run/runit/service   # enable
sudo sv up sddm                                      # start
sudo sv status sddm
```
Install `sddm-runit` (etc.) for the service definition.

### s6
```bash
sudo s6-rc-bundle-update add default sddm     # roughly; via the s6 helper
sudo s6-service-enable sddm
```
Install `sddm-s6`. s6 management uses the `s6-rc` / `66` tooling depending on your setup.

### dinit
```bash
sudo dinitctl enable sddm
sudo dinitctl start sddm
sudo dinitctl status sddm
```
Install `sddm-dinit`.

---

## 2. seatd / elogind — needed for session & GPU access

Wayland and modern X need a session/seat manager. On Artix that's usually **elogind** (logind shim) or **seatd**. Make sure it's installed and its service is enabled for your init *before* the display manager, or logins fail / the GPU device isn't accessible to your user.

```bash
# OpenRC example:
sudo pacman -S elogind-openrc        # or seatd-openrc
sudo rc-update add elogind boot
sudo rc-service elogind start
```

If `nvidia-smi` works as root but apps can't reach the GPU as your user, a missing seat manager is a prime suspect.

---

## 3. Reading logs (no journalctl)

| systemd | Artix equivalent |
|---------|------------------|
| `journalctl -b -p err` | read `/var/log/Xorg.0.log`, `~/.local/share/sddm/*.log`, and your init's logs |
| OpenRC logs | `/var/log/rc.log` (if `rc_logger="YES"` in `/etc/rc.conf`) |
| runit logs | `/var/log/<service>/current` (via `svlogd`) |
| dmesg (works everywhere) | `dmesg | grep -i -E 'nvidia|nouveau|drm'` |

`dmesg`, `/var/log/Xorg.0.log`, and the DM's own log cover almost everything you'd have used `journalctl` for.

---

## 4. Suspend / resume / hibernate

This is the **biggest functional gap** vs Arch-on-systemd. The proprietary driver normally relies on the systemd services `nvidia-suspend.service`, `nvidia-resume.service`, and `nvidia-hibernate.service` to preserve VRAM across sleep. **Those services don't exist on Artix.**

Consequences and fixes:

1. Enable the NVIDIA power-management option so VRAM is saved to system RAM:
   ```bash
   echo 'options nvidia NVreg_PreserveVideoMemoryAllocations=1' | \
     sudo tee /etc/modprobe.d/nvidia-power-management.conf
   sudo mkinitcpio -P
   ```
2. You must hook the equivalent of those systemd services into your init's suspend/resume path manually. Most people use **elogind**'s sleep hooks: drop scripts in `/usr/lib/elogind/system-sleep/` (or `/etc/elogind/system-sleep/`) that run `nvidia-sleep.sh suspend|resume`. The `nvidia-utils` package ships `/usr/bin/nvidia-sleep.sh` — wire it into your sleep handler.
3. If suspend produces a black screen or corrupted display on resume, this missing wiring is almost always why.

---

## 5. Recovery on Artix

The [recovery runbook](recovery.md) applies, with init substitutions:

| Recovery step | systemd | Artix |
|---------------|---------|-------|
| Restart the DM | `systemctl restart sddm` | `rc-service sddm restart` / `sv restart sddm` / `dinitctl restart sddm` |
| Boot to text-only | kernel arg `3` (systemd target) | kernel arg `single` or your init's runlevel; OpenRC: boot without the DM service |
| Rebuild initramfs | `mkinitcpio -P` | `mkinitcpio -P` (same) |

The GRUB one-shot rescue (`nomodeset modprobe.blacklist=nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset`) works identically — it's a kernel feature, not an init feature. Note the systemd `3` target may not apply; on OpenRC just don't let the DM start (remove it from the `default` runlevel temporarily, or boot to `single`).

---

## Summary

- **Install steps:** identical to [Arch](README.md).
- **Service commands:** `rc-service`/`rc-update` (OpenRC), `sv`/symlink (runit), `s6-*` (s6), `dinitctl` (dinit) — never `systemctl`.
- **Logs:** `dmesg` + `/var/log/Xorg.0.log` + DM logs — never `journalctl`.
- **Suspend:** wire `nvidia-sleep.sh` into elogind sleep hooks yourself; set `NVreg_PreserveVideoMemoryAllocations=1`.
- **Everything about the GPU, KMS, Optimus, and the X11-first rule is the same.**
