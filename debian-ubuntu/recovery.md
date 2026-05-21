# Emergency Recovery — Black Screen / No Login (Debian & Ubuntu)

> **The screen is black right now and you need to fix it.** This is the grab-it-when-on-fire runbook. Work top to bottom; stop as soon as you have a desktop back.
>
> **First, the one fact that solves most cases:** if you reached the login screen and it went black *after* you typed your password, you almost certainly logged into **Wayland**. Reboot, and at the login screen pick the **X11 / Xorg** session. That's it. Don't purge anything yet.

## Decision tree

```
Login screen appears, goes black AFTER login
   └─> You're in Wayland. Reboot → pick X11/Xorg session at login.   ✅ 90% of cases

No login screen at all, but Ctrl+Alt+F3 gives a text prompt
   └─> Driver/module problem. Go to "TTY works" below.

Ctrl+Alt+F3 does nothing, totally frozen
   └─> GPU locked. Go to "GRUB one-shot rescue" below.
```

---

## Switching TTYs

- `Ctrl+Alt+F3` (try F2–F6) → text console
- `Ctrl+Alt+F1` or `F7` → back to the graphical session
- Log in with your normal username/password at the text prompt

---

## TTY works — diagnose, then fix the smallest thing

```bash
nvidia-smi                       # driver loaded & talking?
dkms status                      # module built for this kernel?
lsmod | grep nvidia              # module actually loaded?
echo $XDG_SESSION_TYPE           # (if a session somehow exists)
journalctl -b -p err | tail -40  # recent errors this boot
```

Read the result:

- **`nvidia-smi` works but login is black** → it's a session problem, not a driver problem. Reboot and pick X11.
- **`dkms status` empty / shows `added` not `installed`** → module never built. Go to [DKMS build failed](#dkms-build-failed).
- **`nvidia-smi` says "couldn't communicate with the NVIDIA driver"** → module not loaded. Try `sudo modprobe nvidia` and read the error:
  - `Required key not available` → Secure Boot is on and the module is unsigned. Disable Secure Boot in BIOS, or enroll the MOK key.
  - `No such device` → a blacklist file is still suppressing it; check `/etc/modprobe.d/`.

### Get a working desktop immediately (non-destructive)

If you just need to be back at a usable desktop and decide later, temporarily blacklist NVIDIA so you boot on the Intel/iGPU + nouveau:

```bash
echo "blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_uvm
blacklist nvidia_modeset" | sudo tee /etc/modprobe.d/blacklist-nvidia-temporary.conf
sudo update-initramfs -u
sudo reboot
```

Re-enable later:

```bash
sudo rm /etc/modprobe.d/blacklist-nvidia-temporary.conf
sudo update-initramfs -u
sudo reboot
```

### Restart the display manager (crash-loop / repeating keystrokes)

```bash
# Debian (KDE/SDDM):
sudo systemctl restart sddm
# Ubuntu (GNOME/GDM):
sudo systemctl restart gdm3
```

When the login screen returns, **pick the X11/Xorg session before logging in.**

---

## GRUB one-shot rescue — totally frozen, no TTY

1. Hard power off (hold the power button ~10s).
2. Power on; tap **Esc** (Alienware) repeatedly during the logo to catch the GRUB menu. (On some setups hold **Shift**.)
3. Highlight the top entry, press **`e`** to edit.
4. Find the line starting with `linux …` and move the cursor to its **end**.
5. Append:
   ```
   3 nomodeset modprobe.blacklist=nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset
   ```
   - `3` = boot to multi-user text target (no display manager)
   - `nomodeset` = don't let the kernel set a GPU mode early
   - `modprobe.blacklist=…` = don't load the NVIDIA modules this boot
6. Press **`Ctrl+X`** or **F10** to boot.
7. You land at a text login. These edits are **one-shot** — gone on next normal reboot.

From there, apply the non-destructive blacklist (above) to make it survive a reboot, then fix the root cause calmly.

---

## DKMS build failed

`Error! Bad return status for module build` means the kernel module never compiled.

```bash
uname -r                                   # running kernel
dpkg -l | grep linux-headers               # matching headers present?
sudo dkms autoinstall                      # retry the build
sudo cat /var/lib/dkms/nvidia/*/build/make.log   # read the actual error
```

Common causes & fixes:

- **Headers missing / stale:** `sudo apt install --reinstall linux-headers-amd64` (Debian) or `linux-headers-generic` (Ubuntu), then `sudo dkms autoinstall`.
- **Kernel newer than the driver supports** (e.g. kernel 6.19 vs driver 550): boot an older kernel from GRUB → Advanced options, or move to a newer driver from backports / a newer Ubuntu driver branch.
- **GCC mismatch:** the build log will name it; install the matching `gcc-XX` and retry.

---

## Destructive last resort — purge NVIDIA

> Only when the driver itself is genuinely broken and the steps above failed. A purge can leave the desktop degraded (input lag, repeating keystrokes, compositor instability) **beyond** what nouveau alone explains — so prefer the non-destructive blacklist first.

```bash
sudo apt purge '~nnvidia'
sudo apt autoremove --purge
sudo rm -f /etc/modprobe.d/nvidia*.conf
sudo update-initramfs -u
sudo reboot
```

You'll come back up on nouveau (open-source driver). On Ubuntu, reinstall cleanly afterward with `sudo ubuntu-drivers install`; on Debian, redo Steps 3–4 of the [main guide](debian-trixie.md).

---

## The 30-second checklist

1. Black after login? → reboot, pick **X11/Xorg** at the login screen.
2. Still broken? → TTY (`Ctrl+Alt+F3`), run `nvidia-smi` + `dkms status`, read the error.
3. Need a desktop *now*? → temporary blacklist + `update-initramfs -u` + reboot.
4. Totally frozen? → GRUB `e`, append `3 nomodeset modprobe.blacklist=nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset`.
5. Only as a last resort → purge.

**Change one thing at a time, and check the actual error before changing the system.**
