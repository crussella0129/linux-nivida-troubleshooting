# Sprint 0 End-to-End Tests

**Status:** not-yet-possible (deferred to hardware — explicit user instruction this sprint).

Real end-to-end validation requires the actual Debian 13, Arch/Artix, and Fedora/RHEL machines and cannot run in this doc repo. The deferred E2E suite (to run on the target hardware):

- **Debian:** fresh install → enable contrib/non-free → headers → `nvidia-driver` (550.163.01) → reboot → confirm `nvidia-smi` table, `echo $XDG_SESSION_TYPE` == `x11`, PRIME offload renders NVIDIA.
- **Arch (Turing+ GPU):** install `nvidia-open-dkms` per the corrected picker → `mkinitcpio -P` → confirm module loads, `nvidia-smi`, and (if Hyprland 0.42+) cursor is fine **without** `WLR_NO_HARDWARE_CURSORS`, using `cursor { no_hardware_cursors = true }` only if needed.
- **Fedora:** `akmod-nvidia` → wait for build (`modinfo -F version nvidia`) → MOK enroll → reboot → `nvidia-smi`, GNOME on Xorg.

**Unlocked by:** a future "real-install validation" sprint run on the target hardware, which will annotate the guides with `confirmed on <machine> <date>` notes (matching the existing Nighthawk-confirmed Debian annotations).
