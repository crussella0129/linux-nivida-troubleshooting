# linux-nivida-troubleshooting

My personal guides and hard-won notes on getting NVIDIA cards working on Linux — without black-screening the machine on first login.

These following was written using an **Alienware Optimus laptop** (Intel iGPU + NVIDIA dGPU) where the naïve `install the driver and reboot` path lands you at a black screen. The notes generalize to most NVIDIA + hybrid-graphics setups, but this will work for most any Debian + Nvidia setup.

## The universal principle

> **Order of installation matters, and NVIDIA hates Wayland.**
>
> Across every distro, two things cause most of the pain:
> 1. **Install order** — repos/sources first, then *kernel headers/devel*, then the driver, and the kernel module **must finish building** before you reboot.
> 2. **Wayland on Optimus black-screens after login.** The single most effective fix on a hybrid laptop is logging into the **X11 / Xorg** session, not editing config files. When anything breaks, switch to X11 *first*.

Each distro reaches that same conclusion through different package managers and tooling — the guides capture the per-distro specifics.

## Distros

| Directory | Covers | Defining gotcha |
|-----------|--------|------------------|
| [debian-ubuntu/](debian-ubuntu/) | Debian 13 Trixie, Ubuntu 22.04/24.04 | Driver below the 555 Wayland floor → must use X11; pick the session at SDDM/GDM. `nvidia-prime` is Ubuntu-only. |
| [arch-artix/](arch-artix/) | Arch, Artix (OpenRC/runit/s6/dinit) | **Partial upgrades** mismatch `nvidia-utils` vs the kernel module → always `pacman -Syu`. Artix has no systemd. |
| [fedora-redhat/](fedora-redhat/) | Fedora, RHEL/Rocky/Alma | **Wait for the akmod to build** before rebooting; **Secure Boot is on by default** → enroll the MOK key. |

## How each directory is organized

Every distro folder follows the same shape:

- **`README.md`** — the main step-by-step guide.
- **a sibling distro doc** (`ubuntu.md` / `artix.md` / `rhel.md`) — where the related distro diverges.
- **`recovery.md`** — the "screen is black right now" emergency runbook (decision tree + copy-paste fixes).
- **`quick-reference.md`** — a one-page cheatsheet: full sequence, verify commands, package map, emergency one-liners.

Each family's `README.md` opens with a **📌 Canonical, always-current sources** box linking the authoritative upstream docs (ArchWiki, Debian Wiki, RPM Fusion, KDE/NVIDIA). These guides explain the *why* and the Optimus/Alienware specifics; on rolling-release distros especially, **trust the upstream docs over this repo when they disagree** — and tell me (or open an issue) so the guide gets reconciled, like the 2026-05-21 Arch DRM-modeset update.

## Quick start

1. Find your distro in the table above and open its directory.
2. Following a fresh install? Read that folder's `README.md` top to bottom.
3. Already broken / black screen? Jump straight to that folder's `recovery.md`.
4. Just want the commands? `quick-reference.md`.

## What's *not* the fix (on any distro)

- Immediately purging/reinstalling the driver — usually makes things worse. Change one setting at a time.
- Hand-blacklisting nouveau on a fresh install — the distro packaging handles it; manual blacklists are for *recovery*.
- Assuming Wayland will "just work" on an Optimus laptop — it's the most common failure mode. Verify the driver version and `nvidia_drm.modeset=1` before trusting it.

---

*Personal notes, shared in case they help someone else. Hardware referenced: Alienware (Optimus). Adapt versions/paths to your machine.*
