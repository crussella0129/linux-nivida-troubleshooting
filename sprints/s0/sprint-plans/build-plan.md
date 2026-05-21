Finalized - DO NOT EDIT

# Sprint 0 Build Plan

## Schema Tree
- Sprint Goal: correct verified factual drift in the NVIDIA guides
  - Component A — Debian driver-version accuracy
    - T-001: fix version + add kernel gotcha in the main Debian guide
    - T-002: propagate version fix to index/reference docs
  - Component B — Arch driver guidance currency
    - T-003: driver picker → open modules default for Turing+
    - T-004: deprecate WLR_NO_HARDWARE_CURSORS, add Hyprland cursor config
  - Component C — provenance / decision record
    - T-005: append ADRs to decisions.md + set sprint-meta summary

## Execution Sequence

### T-001: Correct Debian driver version and add kernel-compat gotcha in the main guide
- **Touches:** `debian-ubuntu/debian-trixie.md`
- **Depends on:** (none)
- **Success criterion (EARS):**
  - **WHEN** a reader reads *Why this guide exists* / *Wayland: When It Becomes Viable*, **THEN** the guide **SHALL** state the Trixie main driver as `550.163.01` (main and backports), not `535.x`.
  - **WHEN** a reader consults the Wayland/backports section, **THEN** the guide **SHALL** note Trixie's base kernel is 6.12 LTS and the backports 550 driver fails to build on kernel 6.19+.
  - **WHEN** the version is corrected, **THEN** the guide **SHALL** retain the "550 < 555 → use X11" conclusion unchanged.
- **Notes:** two edit sites in one file; keep prose voice intact.

### T-002: Propagate the corrected Debian version to the docs that carry the version table
- **Touches:** `debian-ubuntu/README.md`, `debian-ubuntu/ubuntu.md`
- **Depends on:** T-001
- **Success criterion (EARS):**
  - **WHEN** a reader views the Debian column of the comparison table in `debian-ubuntu/README.md` or `debian-ubuntu/ubuntu.md`, **THEN** it **SHALL** show `550` for the Debian main driver rather than `535 (main) / 550 (backports)`.
  - **WHEN** `grep -rn "535" debian-ubuntu/README.md debian-ubuntu/ubuntu.md debian-ubuntu/debian-trixie.md` is run after T-001+T-002, **THEN** there **SHALL** be exactly zero matches (no legitimate historical 535 lives in these three files).
- **Notes:** Corrected scope per critique C-001/C-002 — the root `README.md` carries no `535` (only a "555 floor" mention, left untouched), and `debian-ubuntu/quick-reference.md` carries no driver-version number. The stale `535 (main) / 550 (backports)` table is in BOTH `debian-ubuntu/README.md:31` and `debian-ubuntu/ubuntu.md:38`; ubuntu.md's Debian column must be fixed too.

### T-003: Update the Arch driver picker to make open modules the default for Turing+
- **Touches:** `arch-artix/README.md`, `arch-artix/quick-reference.md`
- **Depends on:** (none)
- **Success criterion (EARS):**
  - **WHEN** a reader on a Turing+ GPU (RTX 20-series and newer) consults the driver picker, **THEN** the guide **SHALL** recommend `nvidia-open` / `nvidia-open-dkms` as the default.
  - **WHEN** the picker is read, **THEN** the guide **SHALL** state that Arch's `nvidia` package now ships the open kernel modules (driver 560+).
  - **WHEN** a reader on a Maxwell/Pascal/Volta GPU consults the picker, **THEN** the guide **SHALL** direct them to the proprietary `nvidia`/`nvidia-dkms` modules.
- **Notes:** update both the Step 0 table prose and the quick-reference picker table; keep kernel↔package matching rule intact.

### T-004: Mark WLR_NO_HARDWARE_CURSORS deprecated and add the Hyprland cursor config
- **Touches:** `arch-artix/README.md`, `arch-artix/artix.md`, `arch-artix/quick-reference.md`
- **Depends on:** T-003 (shares `arch-artix/README.md` and `quick-reference.md`; run after T-003 to keep those files' commits linear — addresses critique C-006)
- **Success criterion (EARS):**
  - **WHEN** a reader reads the wlroots section of `arch-artix/README.md`, **THEN** it **SHALL** contain the literal word "deprecated" adjacent to the first `WLR_NO_HARDWARE_CURSORS` mention AND show the `hyprland.conf` block `cursor { no_hardware_cursors = true }` as the current approach.
  - **WHEN** a reader uses Sway or older wlroots (incl. the Artix `/etc/environment` path), **THEN** `arch-artix/artix.md` and `arch-artix/quick-reference.md` **SHALL** retain the env var but label it deprecated-for-Hyprland-0.42+ while noting it is still valid for Sway/older.
  - **WHEN** the "Confirmed in LogOS" section references the env var, **THEN** it **SHALL** carry a flag that the LogOS installer's advice predates the Hyprland 0.42 deprecation — without rewording what LogOS actually does.
- **Notes:** Split per critique C-004/C-005. The `cursor{}` config is a `hyprland.conf` construct → README only; the env-var files get deprecation labels, not the config block. Preserve the Sway/Artix retention (research §4 risk 2). Six live `WLR_NO_HARDWARE_CURSORS` sites: README (~287, 293, 384, 393), artix.md (~70), quick-reference.md (~68, 73).

### T-005: Append ADRs to decisions.md and set the sprint summary
- **Touches:** `decisions.md`, `sprints/s0/sprint-meta.md`
- **Depends on:** T-001, T-002, T-003, T-004
- **Success criterion (EARS):**
  - **WHEN** the Loop phase reads `decisions.md`, **THEN** it **SHALL** find an ADR recording the "nvidia-open default for Turing+" stance and one recording the "surgical edits over rewrite" policy.
  - **WHEN** the Loop phase reads `sprint-meta.md`, **THEN** the Summary line **SHALL** describe the sprint goal in one line.
- **Notes:** decisions.md entries become inputs to future Research phases — keep them one-liner ADRs with date.
