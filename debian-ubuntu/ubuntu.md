# NVIDIA Setup on Ubuntu (22.04 / 24.04 LTS)

> Ubuntu and Debian share `apt`, but the NVIDIA story is **different enough that copying the Debian steps will mislead you**. The big divergences:
>
> - Ubuntu **has** `nvidia-prime` + `prime-select` (the Debian guide tells you these don't exist — true on Debian, false here).
> - Ubuntu ships `ubuntu-drivers`, an autodetect helper that picks the right driver branch for you.
> - Ubuntu's recommended drivers are **much newer** (550/560/570+ in `restricted`), so Wayland is closer to viable — but the same "wrong session → black screen" failure mode still bites Optimus laptops.
>
> The universal rule from the [Debian guide](debian-trixie.md) still holds: **install order matters, and NVIDIA + Wayland on Optimus is the thing that black-screens you.** When in doubt, log into X11.

## TL;DR

```bash
# 1. Make sure restricted + multiverse are enabled (default on desktop ISO)
sudo add-apt-repository restricted multiverse
sudo apt update

# 2. Let Ubuntu pick the driver for you
sudo ubuntu-drivers devices          # see what it recommends
sudo ubuntu-drivers install          # installs the recommended branch + DKMS

# 3. Reboot
sudo reboot

# 4. AT THE LOGIN SCREEN (GDM): click the gear, pick "Ubuntu on Xorg"
#    (NOT "Ubuntu" which is Wayland). Then log in.
```

That's the happy path. Everything below is the detail and the failure modes.

---

## Why Ubuntu differs from Debian

| Concern | Debian 13 | Ubuntu 24.04 |
|---------|-----------|--------------|
| Non-free location | `contrib non-free` components | `restricted` + `multiverse` |
| Driver version | 535 (main) / 550 (backports) | 550–570+ in `restricted` |
| Autodetect tool | none | `ubuntu-drivers` |
| Hybrid switching | built into `nvidia-driver` | `prime-select` (the `nvidia-prime` package) |
| Default DE | KDE Plasma (SDDM) on your install | GNOME (GDM3) |
| Wayland floor | below 555 → must use X11 | 555+ available → Wayland *can* work, still flaky on Optimus |

---

## Step 1: Repos

On a standard Ubuntu **Desktop** install, `restricted` and `multiverse` are already enabled. Confirm / enable:

```bash
sudo add-apt-repository restricted multiverse
sudo apt update
```

(`restricted` holds the NVIDIA driver; `multiverse` holds CUDA toolkit and some firmware.)

---

## Step 2: Pick a driver — the `ubuntu-drivers` way (recommended)

```bash
ubuntu-drivers devices
```

Output looks like:

```
vendor   : NVIDIA Corporation
model    : AD107M [GeForce RTX 4060 Max-Q / Mobile]
driver   : nvidia-driver-570 - distro non-free recommended
driver   : nvidia-driver-560 - distro non-free
driver   : nvidia-driver-550 - distro non-free
driver   : xserver-xorg-video-nouveau - distro free builtin
```

Install the recommended one automatically:

```bash
sudo ubuntu-drivers install
```

…or pin a specific branch:

```bash
sudo apt install nvidia-driver-570
```

`ubuntu-drivers install` pulls the driver, the DKMS source, and `nvidia-prime` for you. Kernel headers are pulled in as a dependency (unlike Debian, you usually don't have to install `linux-headers-generic` separately — but if a DKMS build fails, that's the first thing to check).

> **Server / minimal installs:** add `--gpgpu` to get the headless/compute variant: `sudo ubuntu-drivers install --gpgpu nvidia:570-server`.

---

## Step 3: PRIME / hybrid graphics — this is where Ubuntu has tools Debian lacks

Ubuntu's `nvidia-prime` package provides `prime-select`:

```bash
prime-select query        # current mode: nvidia | intel | on-demand
sudo prime-select on-demand   # hybrid: Intel drives display, NVIDIA on request (best for laptops)
sudo prime-select nvidia      # NVIDIA drives everything (max perf, worst battery)
sudo prime-select intel       # NVIDIA powered down (max battery)
```

For an Optimus Alienware laptop, **`on-demand`** is the right default — same behavior as Debian's built-in hybrid mode. After changing modes, **reboot**.

In `on-demand` mode, launch a single app on the dGPU exactly like Debian:

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <command>
# or the Ubuntu convenience var:
prime-run <command>
```

`prime-run` is a wrapper Ubuntu ships that sets those env vars for you.

---

## Step 4: The login-session trap (still applies)

Ubuntu's default DE is **GNOME with GDM3**, and modern Ubuntu defaults the session to **Wayland**. On an Optimus laptop with the proprietary driver, GDM often *hides* the Wayland option and falls back to Xorg automatically — but not always, and KDE/other spins don't.

At the GDM login screen:

1. Click your username
2. Click the **gear icon** (bottom-right)
3. Pick **"Ubuntu on Xorg"** (X11) rather than **"Ubuntu"** (Wayland)
4. Log in

To force GDM off Wayland system-wide:

```bash
sudo nano /etc/gdm3/custom.conf      # note: custom.conf on Ubuntu, daemon.conf on Debian
```

```ini
[daemon]
WaylandEnable=false
```

Then `sudo systemctl restart gdm3` (or reboot).

> Ubuntu 24.04's driver is new enough (550+) that Wayland *can* work — but Optimus laptops still report flicker, XWayland sync glitches, and the occasional post-login black screen. If you hit a black screen, **switch to Xorg first** before touching anything else. Same lesson as Debian.

---

## Step 5: CUDA toolkit

Two options:

**A. Ubuntu's packaged toolkit (simplest, lags upstream):**
```bash
sudo apt install nvidia-cuda-toolkit
nvcc --version
```

**B. NVIDIA's CUDA repo (latest, recommended for ML work):**
```bash
# Example for 24.04 — check developer.nvidia.com/cuda-downloads for current pin file
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install cuda-toolkit
```

For PyTorch/TensorFlow, you usually **don't need the system CUDA toolkit at all** — `pip install torch` ships its own CUDA runtime. The system toolkit only matters when you compile CUDA code with `nvcc`.

---

## Step 6: Verify

```bash
nvidia-smi                              # driver table
prime-select query                      # on-demand
echo $XDG_SESSION_TYPE                   # x11 (or wayland if you chose it)
dkms status                              # nvidia/<ver>: installed
prime-run glxinfo | grep "OpenGL renderer"   # NVIDIA
python3 -c "import torch; print(torch.cuda.is_available())"   # True
```

---

## Secure Boot on Ubuntu

Ubuntu's installer handles MOK enrollment more smoothly than Debian: during driver install you're prompted to **set a one-time password**, and on the next reboot the blue **MOK Manager** screen lets you enroll the key. If you skipped it or the module won't load:

```bash
mokutil --sb-state                       # enabled?
sudo mokutil --import /var/lib/shim-signed/mok/MOK.der   # re-enroll Ubuntu's DKMS key
# set a password, reboot, enroll in MOK Manager
```

`Required key not available` from `modprobe nvidia` = Secure Boot on + unsigned module. Either enroll the key (above) or disable Secure Boot in BIOS.

---

## Recovery

The black-screen recovery runbook in [recovery.md](recovery.md) applies almost verbatim — substitute:

- `update-initramfs -u` → same on Ubuntu
- `gdm3` for `sddm` in the "restart the display manager" commands
- `apt purge '~nnvidia'` works the same; on Ubuntu you can also `sudo ubuntu-drivers autoinstall` to cleanly reinstall afterward

The GRUB one-shot edit (`3 nomodeset modprobe.blacklist=nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset`) is identical.

---

## Ubuntu-specific gotchas

1. **`ubuntu-drivers install` is the path of least resistance.** It picks a sane driver, pulls DKMS + prime + headers, and is the officially supported route. Reach for manual `apt install nvidia-driver-XXX` only when you need a specific branch.
2. **`prime-select` requires a reboot to take effect** — not a logout. Don't trust `nvidia-smi` immediately after switching modes.
3. **GDM config is `custom.conf` on Ubuntu, `daemon.conf` on Debian.** Editing the wrong file silently does nothing.
4. **HWE kernels move fast.** The Hardware Enablement stack ships newer kernels mid-LTS; a kernel jump can outrun your driver branch and break the DKMS build. Pin to a known-good driver branch or hold the kernel if you depend on the GPU for work.
5. **Wayland is closer than on Debian but still not the safe default on Optimus.** When something breaks, X11 (Ubuntu on Xorg) is the first thing to try, every time.
