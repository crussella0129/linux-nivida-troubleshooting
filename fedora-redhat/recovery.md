# Emergency Recovery — Fedora & RHEL

> **Screen is black right now.** Work top to bottom; stop when you have a desktop.

## The three facts that solve most Fedora/RHEL cases

1. **Just installed and rebooted into black?** The **akmod/kmod hadn't finished building** before you rebooted, *or* **Secure Boot blocked the unsigned module.** Both are caught below.
2. **Black after login (login screen was fine)?** You're in Wayland. Reboot → pick **GNOME on Xorg (X11)**.
3. **`nvidia-smi` says it can't talk to the driver?** Either the module didn't build, or Secure Boot rejected it (`Required key not available`).

---

## Decision tree

```
Black right after a fresh install + reboot
   ├─ Secure Boot enabled? → unsigned module rejected. Enroll MOK (below).
   └─ Module not built yet?  → akmods --force, dracut --force (below).

Black after login, login screen was fine
   └─> Wayland session. Reboot → pick GNOME on Xorg.

Black after a kernel update
   └─> akmod didn't rebuild / initramfs stale → akmods --force + dracut --force.

Ctrl+Alt+F3 gives a text login
   └─> diagnose below.

Totally frozen, no TTY
   └─> GRUB one-shot rescue.
```

---

## TTY works — diagnose

`Ctrl+Alt+F3` (try F2–F6), log in:

```bash
mokutil --sb-state                               # Secure Boot on? (Fedora default = yes)
modinfo -F version nvidia                        # built for this kernel? prints version if OK
nvidia-smi                                       # driver alive?
cat /sys/module/nvidia_drm/parameters/modeset    # Y = KMS on
lsmod | grep nvidia                              # loaded?
dmesg | grep -i -E 'nvidia|nouveau|drm'          # kernel-side errors
journalctl -b -p err | tail -40                  # recent errors (Fedora/RHEL have systemd)
ls /var/cache/akmods/nvidia/*.log 2>/dev/null    # akmod build logs
```

Interpretation:

- **`modinfo` errors "module not found"** → the kmod isn't built. Go to [akmod build](#akmod--kmod-build-failed).
- **`modprobe nvidia` → `Required key not available`** → Secure Boot rejected the unsigned module. Go to [Secure Boot / MOK](#secure-boot--mok-enrollment).
- **`nvidia-smi` works but login is black** → session problem; reboot and pick X11.
- **`modeset` prints `N` and you wanted Wayland** → `sudo grubby --update-kernel=ALL --args="nvidia-drm.modeset=1"`.

---

## akmod / kmod build failed

```bash
sudo akmods --force          # force rebuild for the running kernel
sudo dracut --force          # rebuild initramfs with the module
modinfo -F version nvidia    # should now print a version
```

If it still fails, read the build log:

```bash
cat /var/cache/akmods/nvidia/*.log
rpm -q kernel-devel kernel-headers     # must match the running kernel (uname -r)
sudo dnf install "kernel-devel-$(uname -r)"   # install matching devel package
sudo akmods --force && sudo dracut --force
```

Common cause: kernel updated but `kernel-devel` for the *new* kernel wasn't installed, so akmods can't compile. Installing the matching `kernel-devel` and re-forcing fixes it.

---

## Secure Boot / MOK enrollment

If `mokutil --sb-state` is **enabled** and the module won't load:

```bash
sudo kmodgenca -a                                          # ensure CA key exists (Fedora/akmods)
sudo mokutil --import /etc/pki/akmods/certs/public_key.der  # enroll the signing key
# set a one-time password, then:
sudo reboot
```

On reboot the blue **MOK Manager** appears → **Enroll MOK** → **Continue** → **Yes** → enter password → reboot. The signed module now loads.

(RHEL via DKMS: the key may live at `/var/lib/dkms/mok.pub` — import that path instead. See [rhel.md](rhel.md).)

Faster but less secure: disable Secure Boot in BIOS (F2 at the Alienware logo → Security → Secure Boot → Disabled).

---

## Get a working desktop now (non-destructive)

Boot on Intel/nouveau by blacklisting NVIDIA temporarily:

```bash
echo "blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_uvm
blacklist nvidia_modeset" | sudo tee /etc/modprobe.d/blacklist-nvidia-temporary.conf
sudo dracut --force
sudo reboot
```
Undo: remove that file, `sudo dracut --force`, reboot.

## Restart the display manager

```bash
sudo systemctl restart gdm        # Fedora/RHEL GNOME
```
Pick **GNOME on Xorg** before logging in.

---

## GRUB one-shot rescue — totally frozen

1. Hard power off (~10s), power on, tap **Esc** for the GRUB menu.
2. Highlight the entry, press **`e`**.
3. On the `linux`/`linuxefi` line, append:
   ```
   3 rd.driver.blacklist=nouveau,nvidia modprobe.blacklist=nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset nomodeset
   ```
   (`3` = multi-user text target.)
4. **`Ctrl+X`** / **F10** to boot. One-shot only.

For a persistent boot-arg change once you're in:

```bash
sudo grubby --update-kernel=ALL --args="nomodeset"     # add
sudo grubby --update-kernel=ALL --remove-args="nomodeset"  # remove later
```

---

## Last resort — remove NVIDIA, boot on nouveau

```bash
# RPM Fusion (Fedora):
sudo dnf remove '*nvidia*' --exclude=nvidia-gpu-firmware
# CUDA repo / RHEL DKMS:
sudo dnf module remove nvidia-driver:latest-dkms
# then:
sudo rm -f /etc/modprobe.d/*nvidia*.conf
sudo dracut --force
sudo reboot
```

You'll come up on nouveau. Reinstall cleanly later per the [main guide](README.md).

---

## 30-second checklist

1. Fresh install → black → **Secure Boot MOK** not enrolled, or **module not built** (`akmods --force && dracut --force`).
2. Black after login → reboot, pick **GNOME on Xorg**.
3. Black after kernel update → `sudo akmods --force && sudo dracut --force && reboot`.
4. Need desktop now → temporary blacklist + `dracut --force` + reboot.
5. Frozen → GRUB `e`, append `3 ... modprobe.blacklist=nvidia,...`.
