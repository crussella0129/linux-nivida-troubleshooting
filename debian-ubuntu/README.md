# Debian & Ubuntu — NVIDIA Troubleshooting

The `apt`-based family. Debian and Ubuntu share the package manager but **differ in driver versions, repo names, and hybrid-graphics tooling** — read the right guide for your distro.

## The principle that drives everything here

> **Install order matters: repos → kernel headers → driver → reboot → choose X11 at login.**
> NVIDIA's proprietary driver + Wayland on an Optimus laptop reliably **black-screens after login**. The single most effective fix is logging into the **X11 / Xorg** session, not editing config. When something breaks, switch to X11 *first*.

## Contents

| File | What it's for |
|------|---------------|
| [debian-trixie.md](debian-trixie.md) | **Main guide.** Full Debian 13 Trixie walkthrough for an Alienware Optimus laptop. Hard-won, step-by-step. |
| [ubuntu.md](ubuntu.md) | Ubuntu 22.04 / 24.04 path — `ubuntu-drivers`, `prime-select`, GDM, and where it diverges from Debian. |
| [recovery.md](recovery.md) | **Emergency runbook.** Black screen / no login right now — decision tree and copy-paste fixes. |
| [quick-reference.md](quick-reference.md) | One-page cheatsheet: full sequences, verify commands, Debian-vs-Ubuntu package map, emergency one-liners. |

> ### 📌 Canonical, always-current sources
> Driver versions and defaults change between releases — **when this guide and the official docs disagree, trust the official docs.** This guide adds the Optimus/Alienware specifics and the hard-won failure modes.
>
> - **[Debian Wiki: NvidiaGraphicsDrivers](https://wiki.debian.org/NvidiaGraphicsDrivers)** — the authoritative Debian procedure & current driver version
> - **[Ubuntu: NVIDIA drivers installation](https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/)** — `ubuntu-drivers` / `prime-select`
> - **[KDE: Plasma/Wayland/Nvidia](https://community.kde.org/Plasma/Wayland/Nvidia)** — the Wayland driver-version floor (555+)
> - *Last reconciled: 2026-05-21.*

## Start here

- **Debian 13 (KDE/SDDM):** → [debian-trixie.md](debian-trixie.md)
- **Ubuntu (GNOME/GDM):** → [ubuntu.md](ubuntu.md)
- **Already broken / black screen:** → [recovery.md](recovery.md)
- **Just need the commands:** → [quick-reference.md](quick-reference.md)

## Key differences at a glance

| | Debian 13 | Ubuntu 24.04 |
|---|---|---|
| Non-free repo | `contrib non-free` | `restricted multiverse` |
| Driver branch | 550.163.01 (main & backports) | 550–570+ |
| Autodetect helper | none | `ubuntu-drivers` |
| Hybrid switching | built into `nvidia-driver` | `prime-select` / `nvidia-prime` |
| Default DM | SDDM (KDE) | GDM3 (GNOME) |

`nvidia-prime` and `prime-select` are **Ubuntu-only**. Don't go looking for them on Debian — PRIME offload is built into Debian's `nvidia-driver`.
