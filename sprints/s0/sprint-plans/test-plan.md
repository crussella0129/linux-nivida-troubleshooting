Finalized - DO NOT EDIT

# Sprint 0 Test Plan

> These are doc-lint verification checks, not software tests. Hardware behavior is verified on other machines in a future "real-install" sprint (out of scope per the sprint goal). Each check maps to a build-plan EARS clause and is a `grep`/inspection assertion runnable in this repo.

## Unit Tests

### T-001 unit checks
- `test_debian_main_version_is_550`: `grep -n "550.163.01" debian-ubuntu/debian-trixie.md` → matches in *Why this guide exists* and *Wayland* sections.
- `test_debian_no_535_in_main_guide`: `grep -cn "535" debian-ubuntu/debian-trixie.md` → exactly `0` (both pre-existing `535` sites are version claims being corrected; no historical 535 remains in this file). Concrete per critique C-003.
- `test_debian_kernel_gotcha_present`: `grep -niE "6\.12|6\.19" debian-ubuntu/debian-trixie.md` → matches the new kernel-compat note.
- `test_debian_x11_conclusion_intact`: `grep -ni "555" debian-ubuntu/debian-trixie.md` → the 555-floor reasoning still present.

### T-002 unit checks
- `test_index_version_updated`: `grep -rn "535" debian-ubuntu/README.md debian-ubuntu/ubuntu.md` → no match (both the family README and ubuntu.md tables fixed). Per critique C-001/C-002.
- `test_root_readme_untouched`: `grep -n "535" README.md` → no match expected and none introduced (root README carries no version table; only the "555 floor" line, which stays).
- `test_index_shows_550`: `grep -n "550" debian-ubuntu/README.md` → Debian column shows 550.

### T-003 unit checks
- `test_arch_open_default_turing`: `grep -niE "nvidia-open" arch-artix/README.md arch-artix/quick-reference.md` → present and framed as default/recommended for Turing+.
- `test_arch_open_modules_note`: `grep -niE "open kernel module|560" arch-artix/README.md` → states `nvidia` package ships open modules.
- `test_arch_proprietary_legacy`: `grep -niE "maxwell|pascal|volta" arch-artix/README.md` → proprietary path restricted to legacy GPUs.

### T-004 unit checks
- `test_readme_deprecated_token`: `grep -ni "deprecat" arch-artix/README.md` → the literal "deprecated" appears in the wlroots section near the first `WLR_NO_HARDWARE_CURSORS` mention. Concrete per critique C-005.
- `test_hypr_cursor_config_present`: `grep -n "no_hardware_cursors = true" arch-artix/README.md` → the `cursor { no_hardware_cursors = true }` hyprland.conf block shown (README only — the config is a hyprland.conf construct, per C-004).
- `test_envvar_files_labelled`: `grep -ni "deprecat" arch-artix/artix.md arch-artix/quick-reference.md` → each env-var block carries a deprecated-for-Hyprland-0.42 label while retaining the var.
- `test_sway_retention_preserved`: `grep -ni "sway" arch-artix/README.md` → env var still noted as valid for Sway/older wlroots (research §4 risk 2 — must NOT be deleted).
- `test_logos_predates_flag`: manual inspection — the "Confirmed in LogOS" mention is flagged as predating Hyprland 0.42, with LogOS's actual behavior left accurately described (not reworded).

### T-005 unit checks
- `test_decisions_has_adrs`: `grep -niE "nvidia-open|surgical" decisions.md` → both ADRs present with dates.
- `test_sprint_summary_set`: `grep -n "Summary:" sprints/s0/sprint-meta.md` → no longer the placeholder.

## Integration Tests
### Component A integration (Debian version)
- `test_debian_version_consistent_across_docs`: across `debian-ubuntu/debian-trixie.md`, `debian-ubuntu/README.md`, and `debian-ubuntu/ubuntu.md` (the three docs that state a Debian driver figure), the main driver is 550 everywhere — no doc disagrees. (Root `README.md` carries no figure; excluded by design per C-001.)

### Component B integration (Arch currency)
- `test_arch_cross_doc_consistency`: `arch-artix/README.md` and `arch-artix/quick-reference.md` agree on (a) open-default for Turing+ and (b) WLR var deprecation; no internal contradiction.

### Cross-cutting integration
- `test_internal_anchors_resolve`: every `](#...)` and `](recovery.md#...)`/`(README.md#...)` link target still corresponds to an existing heading slug after edits (manual inspection of changed headings; no heading text in edited sections was renamed).
- `test_diff_scope`: `git diff --stat` lists only the 8 planned files; each task is a separate commit (`git log --oneline` shows T-001..T-005).

## End-to-End Tests
- **Status:** not-yet-possible
- Real installs (boot to desktop, `nvidia-smi`, X11 vs Wayland session behavior, akmod/DKMS build, MOK enrollment) require the actual Debian/Arch/Fedora machines.
- Unlocked by: a future "real-install validation" sprint run on the target hardware, which will annotate guides with `confirmed on <machine> <date>` notes — explicitly deferred this sprint per the user's instruction.
