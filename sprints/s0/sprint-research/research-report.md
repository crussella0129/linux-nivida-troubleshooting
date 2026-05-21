# Sprint 0 Research Report

## Decisions Reviewed

`decisions.md` has no entries yet. No prior decision is being violated.

## 1. Sprint Goal

Review every guide in the `linux-nivida-troubleshooting` repo (the Debian/Ubuntu, Arch/Artix, and Fedora/RHEL families), verify their concrete technical claims against current external sources, and produce a build plan to correct any drift. The repo's value is accuracy under a hostile failure mode (NVIDIA black-screens), so wrong version numbers or deprecated commands are real defects. Hardware testing is explicitly out of scope this sprint (it happens on other machines), so the Test phase is skipped.

## 2. Existing Code Survey

| File | Relevance | Notes |
|------|-----------|-------|
| `README.md` | high | Top-level distro matrix; repeats the "535 in main / 550 backports" Debian claim that is now wrong. |
| `debian-ubuntu/debian-trixie.md` | high | States Trixie ships **535.x in main, 550.x in backports**. Verified wrong — both are 550.163.01. Two locations ("Why this guide exists", "Wayland: When It Becomes Viable"). |
| `debian-ubuntu/ubuntu.md` | low | Ubuntu version claims (550–570 in restricted) are plausible/current; no correction needed. |
| `debian-ubuntu/recovery.md` | low | Command-level; no version claims to correct. |
| `debian-ubuntu/quick-reference.md` | medium | Mirrors driver-version framing; check for the 535 number. |
| `arch-artix/README.md` | high | Driver-picker treats `nvidia-open` as secondary; reality: open is now default/recommended for Turing+. Also recommends `WLR_NO_HARDWARE_CURSORS=1` (deprecated). |
| `arch-artix/artix.md` | medium | Repeats the `WLR_NO_HARDWARE_CURSORS` env var; same deprecation applies. |
| `arch-artix/quick-reference.md` | medium | Has the wlroots env-var block with `WLR_NO_HARDWARE_CURSORS=1`. |
| `arch-artix/recovery.md` | low | Command-level; fine. |
| `fedora-redhat/README.md` | medium | akmods + Secure Boot MOK flow verified correct (`kmodgenca -a`, `mokutil --import /etc/pki/akmods/certs/public_key.der`). No correction. |
| `fedora-redhat/rhel.md` | low | CUDA-repo / ELRepo routes plausible; no high-risk claim flagged. |
| `fedora-redhat/{recovery,quick-reference}.md` | low | Command-level; fine. |
| `~/LogOS-Arch/lib/desktop-hyprland.sh` | medium | Source of the `WLR_NO_HARDWARE_CURSORS=1` advice I imported — itself now dated vs Hyprland 0.42+. |

## 3. External Sources

- [Debian — package nvidia-driver in trixie / trixie-backports](https://packages.debian.org/trixie/nvidia-driver) — Trixie main ships **550.163.01**; backports same line. Refutes the "535 in main" claim. (Also surfaced via LinuxConfig's Debian 13 NVIDIA guide.)
- [NVIDIA driver compatibility with kernel 6.17/6.19 — NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/nvidia-driver-compatibility-with-linux-kernel-6-17-debian-trixie/354354) — the trixie-backports 550.163.01-4~bpo13+1 driver no longer compiles on kernel 6.19; Trixie base kernel is 6.12 LTS. New gotcha to document.
- [NVidia — Hyprland Wiki](https://wiki.hypr.land/0.42.0/Nvidia/) + [ML4W dotfiles issue #327](https://gitlab.com/stephan-raabe/dotfiles/-/issues/327) — `WLR_NO_HARDWARE_CURSORS` is **deprecated** since Hyprland 0.42; HW-cursor bug fixed on recent drivers; use `cursor { no_hardware_cursors = true }` in `hyprland.conf` only if still needed.
- [NVIDIA Transitions Fully Towards Open-Source GPU Kernel Modules](https://developer.nvidia.com/blog/nvidia-transitions-fully-towards-open-source-gpu-kernel-modules/) + [Phoronix: Arch's main NVIDIA packages now use open modules](https://www.phoronix.com/news/Arch-LInux-NVIDIA-Open-Default) — open modules are default from driver 560+ and recommended for Turing+; proprietary only for Maxwell/Pascal/Volta.
- [Plasma/Wayland/Nvidia — KDE Community Wiki](https://community.kde.org/Plasma/Wayland/Nvidia) + [NVIDIA 555.58 explicit sync — 9to5Linux](https://9to5linux.com/nvidia-555-58-linux-graphics-driver-released-with-explicit-sync-on-wayland) — confirms the **555 floor** for usable Wayland (explicit sync, Plasma 6.1, XWayland 24.1). My claim stands.
- [Fedora akmods README.secureboot + Fedora Discussion](https://discussion.fedoraproject.org/t/unable-to-load-nvidia-drivers-with-secure-boot-enabled/85426) — confirms `kmodgenca -a` → `mokutil --import /etc/pki/akmods/certs/public_key.der` → reboot → Enroll MOK. My Fedora guide is correct.

## 4. Risks, Unknowns, Dependencies

- **Risk (low):** correcting the Debian driver number to 550 must not undercut the "use X11" thesis — 550 is still < 555, so the conclusion is unchanged; only the figure changes. Keep the reasoning intact.
- **Risk (low):** the `WLR_NO_HARDWARE_CURSORS` env var still *works* (deprecated, not removed) and is still the right lever for **Sway** and older wlroots. Don't delete it outright — reframe as "deprecated for Hyprland 0.42+, prefer the config option; still valid for Sway/older."
- **Unknown:** whether Arch's `nvidia` package being open-by-default changes the `nvidia`-vs-`nvidia-open` guidance enough to restructure the picker table, or just add a note. Lean toward a note + updated recommendation to avoid churn.
- **Unknown (deferred):** did not re-verify that Arch's `nvidia-prime` package provides `prime-run` (high confidence it does); low risk, can spot-check during build.
- **Dependency:** none external; all changes are doc edits. Git is the only tool dependency. No hardware needed.

## 5. Recommended Approach

**Primary:** Targeted in-place corrections to the four affected files plus the top README, scoped as discrete build tasks (one per claim-cluster), each committed separately. Preserve the hard-won prose and the X11-first thesis; change only the inaccurate facts and add the two newly-discovered gotchas (Debian kernel 6.12 / backports 6.19 compile break; Hyprland cursor-config migration).

**Alternative considered:** A full rewrite pass of every guide for consistency. Rejected — higher risk of introducing new errors, larger diff to review, and most content verified accurate. Surgical edits match the repo's "annotate as confirmed" ethos better.

**Rationale:** Three confirmed factual defects (Debian version, wlroots cursor var, open-module default) and two confirmed-correct areas (KDE 555 floor, Fedora MOK). Fix what's wrong, leave what's right, document the two new gotchas. Note where LogOS-sourced advice has since aged.

## Artifacts

No separate evidence files saved; all external findings are cited inline in §3 with URLs (within the 5-source budget).
