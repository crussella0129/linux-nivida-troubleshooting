# NVIDIA Setup on Alienware with Debian 13 Trixie

> **Hard-won notes.** This procedure was developed across multiple black-screen incidents on Nighthawk. Every step here exists because a previous shortcut didn't work. The single most important step is **Step 8** — picking X11 at the SDDM login screen. The config file alone is not enough.
>
> On Ubuntu the package names and tooling differ (it *does* have `nvidia-prime` / `prime-select` / `ubuntu-drivers`). See [ubuntu.md](ubuntu.md) for the Ubuntu path. For the emergency "screen is black right now" runbook, jump to [recovery.md](recovery.md).

## TL;DR (the actual minimum that works)

```bash
# 1. Edit /etc/apt/sources.list — add "contrib non-free" to every active deb/deb-src line
sudo nano /etc/apt/sources.list

# 2. Refresh
sudo apt update

# 3. Headers FIRST, separately
sudo apt install linux-headers-amd64

# 4. NVIDIA stack (note: NO nvidia-prime — that package doesn't exist on Debian)
sudo apt install nvidia-driver nvidia-kernel-dkms nvidia-settings \
                 nvidia-cuda-toolkit firmware-misc-nonfree

# 5. Reboot
sudo reboot

# 6. AT THE LOGIN SCREEN — click your username, find the session selector,
#    and pick "Plasma (X11)" instead of "Plasma (Wayland)". This is the actual fix.
```

If anything goes wrong, see [Recovery from Black Screen](#recovery-from-black-screen) (or the standalone [recovery.md](recovery.md) runbook).

---

## Why this guide exists

Naïve install (`sudo apt install nvidia-driver`) **black-screens an Alienware laptop after login**. Two reasons stack:

1. **KDE Plasma 6 defaults to Wayland on Trixie.** NVIDIA's Wayland support requires driver 555+ for stability (per KDE's own documentation — that's the explicit-sync floor). Debian Trixie ships **550.163.01** in both main and backports — **still below the 555 floor.**
2. **Optimus hybrid graphics.** Intel iGPU drives the display; NVIDIA dGPU renders to an offscreen buffer that gets composited over. Wayland + Optimus + sub-555 NVIDIA = broken session.

The fix isn't a complex config — it's **logging into the X11 session instead of Wayland**. The SDDM config file we add helps but isn't always sufficient on first boot; the session selector at login is the actual control surface.

### Things that are NOT the fix (despite being widely suggested)

- **`nvidia-prime`** — Ubuntu-only package. Doesn't exist on Debian. PRIME offloading is built into Debian's `nvidia-driver` directly. Don't try to install it.
- **`prime-select on-demand`** — Ubuntu command. Doesn't exist on Debian. Hybrid mode is the default after install.
- **Blacklisting nouveau manually** — Debian's installer does this automatically. Don't add custom blacklist files unless recovering from a broken install.

---

## Prerequisites

- Fresh or working Debian 13 Trixie install
- Ethernet connection preferred (WiFi setup via `nmcli` on fresh install is fiddly with WPA-PSK; ethernet just works)
- Sudo access
- Know which DE you installed: KDE, GNOME, XFCE, or other

### Verify before starting

```bash
cat /etc/os-release | grep VERSION    # Confirm Trixie
lspci | grep -i -E "vga|3d|display"   # Confirm Intel + NVIDIA present
mokutil --sb-state                    # Secure Boot status
echo $XDG_SESSION_TYPE                # Current session type
uname -r                              # Current kernel
```

You should see Intel graphics AND NVIDIA in `lspci`. If only NVIDIA shows, you don't have Optimus.

---

## Step 1: Enable contrib and non-free repos

NVIDIA's proprietary blob lives in `non-free`. The wrapper scripts live in `contrib`. Default Trixie enables only `main non-free-firmware`.

```bash
sudo nano /etc/apt/sources.list
```

For every active `deb` and `deb-src` line, append `contrib non-free`. The file should end up looking like:

```text
deb http://deb.debian.org/debian/ trixie main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian/ trixie main contrib non-free non-free-firmware

deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
deb-src http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian/ trixie-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian/ trixie-updates main contrib non-free non-free-firmware
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

> **Critical:** `contrib`, `non-free`, and `non-free-firmware` are **three separate components**, not hyphenated names. All three are needed. Watch for typos — `contib` instead of `contrib` will silently fail with no package candidate.

> **Trixie note:** newer installs may use the deb822 format at `/etc/apt/sources.list.d/debian.sources` instead of the one-line `sources.list`. If `/etc/apt/sources.list` is empty or missing, edit `debian.sources` and add the words to the `Components:` line: `Components: main contrib non-free non-free-firmware`.

### Why no `update-grub` here?

`/etc/apt/sources.list` is only read by apt. GRUB reads `/etc/default/grub` and `/etc/grub.d/`. Different subsystems, different config files. You only run `sudo update-grub` after kernel parameter changes, new kernel installs, or custom boot entries.

(Arch equivalent: `grub-mkconfig` is similarly unrelated to `/etc/pacman.conf` edits.)

---

## Step 2: Refresh package indexes

```bash
sudo apt update
```

Verify NVIDIA packages are visible:

```bash
apt-cache policy nvidia-driver
```

You should see a `Candidate:` version listed. If it says `Candidate: (none)`, the sources file didn't save correctly — back to Step 1.

---

## Step 3: Install kernel headers FIRST, separately

**Debian 13's DKMS won't build the NVIDIA kernel module without headers present at install time.** Bundling headers with driver packages in one command can race.

```bash
sudo apt install linux-headers-amd64
```

Use `linux-headers-amd64` (the metapackage) instead of `linux-headers-$(uname -r)` — the metapackage tracks future kernel upgrades automatically; a version-pinned package goes stale after the next `apt upgrade`.

Wait for this to complete fully before moving on.

---

## Step 4: Install the NVIDIA stack

```bash
sudo apt install nvidia-driver nvidia-kernel-dkms nvidia-settings \
                 nvidia-cuda-toolkit firmware-misc-nonfree
```

What each package does:

| Package | Purpose |
|---------|---------|
| `nvidia-driver` | Proprietary driver metapackage — pulls in `libnvidia-*` and X11 driver |
| `nvidia-kernel-dkms` | DKMS source; rebuilds kernel module on every kernel upgrade |
| `nvidia-settings` | GUI control panel for power state, performance, monitor config |
| `nvidia-cuda-toolkit` | CUDA runtime + `nvcc` compiler. Debian's version lags upstream by ~1 release. |
| `firmware-misc-nonfree` | Proprietary firmware blobs (GPU + various others) |

**Notably absent:** `nvidia-prime`. That's an Ubuntu package. PRIME render offload functionality is built into Debian's `nvidia-driver` directly — no separate package needed.

Expect ~3–5 GB of downloads. DKMS module build at the end spins the CPU for 1–2 minutes.

### Expected dialog during install

You'll see a blue ncurses dialog: **"Conflicting nouveau kernel module loaded."** This is **informational, not an error.** It's telling you nouveau is currently running (driving your screen as you read this) and will be replaced by NVIDIA at the next reboot. Press Enter to dismiss.

### Optional but useful

```bash
sudo apt install vulkan-tools mesa-utils
```

- `vulkan-tools` → `vulkaninfo`
- `mesa-utils` → `glxinfo`, `glxgears`

---

## Step 5: (Optional) SDDM X11 default config

> **This step is helpful but NOT sufficient on its own.** It sets the *default* session for SDDM but doesn't always override what Plasma picks at first boot. **Step 8 (picking X11 at the login screen) is the actual control surface.**

For KDE Plasma (SDDM):

```bash
sudo mkdir -p /etc/sddm.conf.d
echo -e "[General]\nDisplayServer=x11" | sudo tee /etc/sddm.conf.d/10-x11.conf
```

For GNOME (GDM3):

```bash
sudo nano /etc/gdm3/daemon.conf
```

Find `[daemon]` section, uncomment or add:
```ini
[daemon]
WaylandEnable=false
```

For XFCE / LXQt / others: X11 by default. Skip this step.

---

## Step 6: Secure Boot (only if `mokutil --sb-state` shows enabled)

If Secure Boot is on, the DKMS-built NVIDIA module is unsigned and the kernel will refuse to load it. Two options:

### Option A — Disable in BIOS (simplest)

1. Reboot, F2/F12/Del during Alienware logo to enter BIOS
2. Security → Secure Boot → Disabled
3. Save and exit

### Option B — Enroll MOK key

```bash
sudo update-secureboot-policy --new-key
sudo update-secureboot-policy --enroll-key
```

On next reboot you'll see a blue **MOK Manager** screen:
1. Enroll MOK → Continue → Yes
2. Enter the password you set
3. Reboot

If Secure Boot was already off, skip this step entirely.

---

## Step 7: Reboot

```bash
sudo reboot
```

---

## Step 8: At the SDDM login screen — DO NOT log in yet

**This is the step that actually makes everything work.** You should see SDDM (the login screen). Before typing your password:

1. Click your username (charles)
2. Look at the **bottom-left of the screen** (sometimes bottom-right depending on theme) for a small icon — usually a **gear icon**, or a label that says "Plasma" or shows session type
3. Click it. A dropdown appears with session options
4. **Pick "Plasma (X11)"** — NOT "Plasma" or "Plasma (Wayland)"
5. *Now* type your password and log in

If you skip this and let SDDM use its default, you may still land in a Wayland session despite the Step 5 config file, and the screen will go black after login.

> Confirmed working on Nighthawk 2026-05-15: SDDM config alone left the system on Wayland → black screen. Picking "Plasma (X11)" from the SDDM dropdown produced a fully working desktop immediately.

### What if there's no session selector?

The X11 Plasma session may not be installed. From a TTY (Ctrl+Alt+F3) or recovery mode:

```bash
sudo apt install plasma-workspace-x11
sudo reboot
```

The session selector should now have X11 listed.

---

## Step 9: Verify everything works

After logging into the X11 session:

```bash
nvidia-smi
```
Should print a table showing your GPU model, driver version, and processes. If you see `NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver`, the module didn't load — see [Module Not Loading](#module-not-loading) (or [recovery.md](recovery.md)).

```bash
echo $XDG_SESSION_TYPE
```
Must print `x11`. If it prints `wayland`, you're in the wrong session — log out and reselect.

```bash
dkms status
```
Should show `nvidia/<version>, <kernel>: installed`.

```bash
glxinfo | grep "OpenGL renderer"
```
By default shows **Intel** — that's correct, you're in hybrid mode with Intel driving the display.

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"
```
Shows NVIDIA. Confirms PRIME offload works.

```bash
nvcc --version
```
Shows CUDA compiler version (likely 12.x range on Trixie).

### CUDA from Python

PyTorch (when installed via pip with CUDA support):
```bash
python3 -c "import torch; print(torch.cuda.is_available(), torch.cuda.device_count())"
```

CUDA workloads use NVIDIA automatically via `/dev/nvidia*` — no env vars needed. The PRIME offload env vars only apply to OpenGL/Vulkan rendering, not compute.

---

## Using NVIDIA GPU for specific apps

To run a graphical app on NVIDIA instead of Intel:
```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <command>
```

For convenience, alias it:
```bash
echo 'alias nvrun="__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia"' >> ~/.bashrc
source ~/.bashrc
# Now: nvrun blender, nvrun glxgears, etc.
```

GNOME users get a context menu option: right-click app → "Launch using Discrete Graphics Card." KDE doesn't have this by default.

For CUDA Python work, no special invocation needed.

---

## Recovery from Black Screen

> The full emergency runbook (decision tree, GRUB one-liners, copy-paste blocks) lives in [recovery.md](recovery.md). The condensed version follows.

### If TTY works (Ctrl+Alt+F3)

Log in at the text prompt. Diagnose:

```bash
nvidia-smi              # Driver loaded?
dkms status             # Module built?
lsmod | grep nvidia     # Module actually loaded?
journalctl -b -p err | tail -40   # Recent errors
```

**Most likely cause if SDDM appears but login goes black:** You logged into Wayland instead of X11. Reboot, pick "Plasma (X11)" at the login screen.

### If TTY also fails

The GPU is locked up. GRUB recovery is the only path.

1. Hard power off (hold button 10s)
2. Power on, tap **Esc** during Alienware logo to catch GRUB
3. Highlight top Debian entry, press **`e`** to edit
4. Find line starting with `linux`, navigate to end
5. Add at the end:
   ```
   3 nomodeset modprobe.blacklist=nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset
   ```
6. **`Ctrl+X`** or **F10** to boot
7. Log in at text prompt

These edits are **one-shot only** — they don't persist past the next normal reboot.

### Non-destructive fix (preferred): blacklist NVIDIA temporarily

Doesn't remove the driver, just prevents loading. Lets you boot to a working desktop, then choose what to do.

```bash
echo "blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_uvm
blacklist nvidia_modeset" | sudo tee /etc/modprobe.d/blacklist-nvidia-temporary.conf

sudo update-initramfs -u
sudo reboot
```

To re-enable later:
```bash
sudo rm /etc/modprobe.d/blacklist-nvidia-temporary.conf
sudo update-initramfs -u
sudo reboot
```

### Destructive fix (last resort): purge NVIDIA

> **Note from experience on Nighthawk:** The purge cleanly removes packages but may leave the system in a degraded state where the desktop session has issues beyond what nouveau alone explains. Symptoms included input lag, repeating keystrokes, and Plasma session instability. **Prefer the blacklist approach above unless the driver itself is genuinely broken.**

```bash
sudo apt purge '~nnvidia'
sudo apt autoremove --purge
sudo rm -f /etc/modprobe.d/nvidia*.conf
sudo update-initramfs -u
sudo reboot
```

You'll come back up on nouveau.

---

## Known Issues

### DKMS Build Failed

If you see `Error! Bad return status for module build`:

```bash
uname -r                              # Current kernel
dpkg -l | grep linux-headers          # Are matching headers installed?
sudo dkms autoinstall                 # Retry build
sudo cat /var/lib/dkms/nvidia/*/build/make.log   # See build error
```

Common causes:
- Kernel upgraded but `linux-headers-amd64` not refreshed → `sudo apt install --reinstall linux-headers-amd64`
- Kernel too new (6.19+) for driver 550.x → wait for backports update or use older kernel

### Module Not Loading

```bash
sudo modprobe nvidia
```

- `Required key not available` → Secure Boot enabled and module unsigned. See [Step 6: Secure Boot](#step-6-secure-boot-only-if-mokutil---sb-state-shows-enabled).
- `No such device` → blacklist file still present. Check `/etc/modprobe.d/`.

### Black Screen After Login (the most common failure)

**You logged into Wayland instead of X11.** See [Recovery from Black Screen](#recovery-from-black-screen). The fix is to pick the X11 session at SDDM, not to change config files.

### Wayland Re-enabled After Update

Some Plasma updates reset session preferences. Re-pick Plasma (X11) at the login screen, or re-create `/etc/sddm.conf.d/10-x11.conf` from Step 5.

### Repeating Keystrokes / System Unresponsive

KWin/plasmashell crash-restart loop. Get to TTY:

```bash
sudo systemctl stop sddm
sleep 3
sudo systemctl start sddm
```

Login screen returns. **Pick X11 session before logging in.**

---

## Wayland: When It Becomes Viable

NVIDIA + Wayland requires driver **555+** per [KDE's own documentation](https://community.kde.org/Plasma/Wayland/Nvidia) (the 555.58 release added explicit GPU sync via `linux-drm-syncobj-v1`, which Plasma 6.1 needs to stop flickering). Debian Trixie ships **550.163.01** in both main *and* backports — below the 555 floor, so X11 remains the recommendation on Trixie.

> **Kernel-compat gotcha (Trixie):** Trixie's base kernel is **6.12 LTS**, which the 550 driver builds against fine. But `trixie-backports` has shipped newer kernels, and as of early 2026 the backports 550.163.01 driver **no longer compiles on kernel 6.19+** (DKMS build fails). If you run a backports kernel, either hold it at a 550-compatible version or wait for a newer backports driver. Check `uname -r` against what the driver supports before upgrading the kernel.

When Trixie backports ships 565+ (which has the major Plasma compositor crash fix):

```bash
# Add backports if not already enabled
echo "deb http://deb.debian.org/debian trixie-backports main contrib non-free non-free-firmware" | \
  sudo tee /etc/apt/sources.list.d/backports.list
sudo apt update

# Upgrade to backports driver
sudo apt install -t trixie-backports nvidia-driver nvidia-kernel-dkms

# Verify version
nvidia-smi | head -3

# Remove X11 forcing (if you set it)
sudo rm /etc/sddm.conf.d/10-x11.conf

sudo reboot
```

Then at the login screen, pick **Plasma (Wayland)** instead of X11.

Even on 565+, Optimus laptops report occasional flicker and XWayland sync issues. X11 remains the most stable path. Don't switch unless you have a specific Wayland-only feature you need (fractional scaling, mixed refresh rates).

---

## Workflow References

- [[Animus Prion]] — benefits from CUDA via `torch.cuda`. Step 9 verification applies.
- [[Diploid]] — dual-model inference assumes CUDA available. Same.
- [[LogOS DaCha]] — ZimaBoard NAS, no GPU, doesn't apply.
- [[Klawdeck]] — Pi cluster, doesn't apply.
- Sapphire Throne — security model: Secure Boot + signed DKMS aligns with hardened LogOS profiles.

---

## Lessons Learned (the actual hard-won notes)

1. **The session selector at SDDM login is the actual control surface, not the config file.** The `DisplayServer=x11` config in `/etc/sddm.conf.d/` biases the default but doesn't force the runtime session. KDE's first-boot logic can pick Wayland anyway. The dropdown menu at SDDM where you choose "Plasma (X11)" is the actual switch that works.

2. **`nvidia-prime` is Ubuntu-only.** It doesn't exist on Debian. PRIME functionality is built into `nvidia-driver`. Don't waste time looking for the Debian package — there isn't one. (`envycontrol` is a third-party option if you want GUI hybrid switching, but you don't need it for CUDA work.)

3. **The "Conflicting nouveau kernel module loaded" dialog during install is informational.** Press Enter and continue. It's just telling you the swap happens at reboot.

4. **A purge can leave the system in a worse state than the original problem.** It cleanly removes packages, but the subsequent desktop experience may have issues — input lag, repeating keystrokes, Plasma instability — beyond what nouveau alone explains. **Prefer the non-destructive blacklist approach** (see [recovery.md → blacklist](recovery.md#non-destructive-fix-temporarily-blacklist-nvidia)).

5. **Debug the symptom before changing the system.** When a login goes black, first try: log out, pick different session at login screen. Don't immediately reach for purges, reinstalls, or kernel parameter changes. The cheapest fix is almost always: change one setting at the login screen.

6. **`uname` not `usname`.** Linux command names are case-sensitive and spelling-sensitive. A typo in `$(uname -r)` becomes "package not found" further down the chain. Always check the actual error path.

7. **`contrib` and `non-free` are separate components**, not a hyphenated name. And `non-free-firmware` is a third separate component. All three are needed for full NVIDIA support.

8. **The session selector at SDDM is small and easy to miss.** It's often an unlabeled gear icon in a corner. Look carefully — clicking it reveals the dropdown.

9. **Watch for typos in nano edits.** `contib` (missing the `r`) is a silent failure — apt update succeeds, candidate stays `(none)`, install fails with "no installation candidate." Re-read every line after editing.
