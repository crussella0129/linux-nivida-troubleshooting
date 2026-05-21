# Plan Critique — Sprint 0

> Critic confidence: **proceed-with-caveats**. Primary-agent responses appended inline under each concern. C-001/C-002 were ground-truthed with `grep` before accepting.

## Concerns

### C-001: T-002 rests on a false premise — the top `README.md` contains no `535` claim to fix
- **Where:** build-plan.md T-002 Touches; research-report.md §2 row `README.md`; test-plan.md `test_index_version_updated`/`test_index_shows_550`.
- **Failure mode:** plan-test-mismatch
- **Why it matters:** Root `README.md` says only "Driver below the 555 Wayland floor" — no `535`, no version table. The stale string lives in `debian-ubuntu/README.md:31`.
- **Suggested response:** fix-in-plan.
- **RESPONSE (fix-in-plan, accepted):** Verified via `grep`: root `README.md` has zero `535`. Removed `README.md` and `debian-ubuntu/quick-reference.md` (also no number) from T-002 Touches. Added `test_root_readme_untouched`. The research report row was wrong; superseded by the corrected T-002.

### C-002: `debian-ubuntu/ubuntu.md` carries the stale `535 (main) / 550 (backports)` table but was in no task's Touches
- **Where:** ubuntu.md:38; test-plan.md `test_index_version_updated` (recursive grep).
- **Failure mode:** plan-test-mismatch / missing-coverage
- **Why it matters:** `535 (main)` exists in BOTH `debian-ubuntu/README.md:31` and `ubuntu.md:38`; the recursive grep can't pass unless ubuntu.md is edited.
- **Suggested response:** fix-in-plan.
- **RESPONSE (fix-in-plan, accepted):** Verified via `grep`: ubuntu.md:38 has the stale Debian column. Added `debian-ubuntu/ubuntu.md` to T-002 Touches and to `test_debian_version_consistent_across_docs`. Note: ubuntu.md's Ubuntu column (550–570+) stays; only the Debian column figure changes.

### C-003: "no remaining 535" criterion was unmeasurable
- **Where:** build-plan.md T-002 EARS clause 2; test-plan.md `test_debian_no_535_main_claim`.
- **Failure mode:** EARS-vague
- **Suggested response:** fix-in-plan.
- **RESPONSE (fix-in-plan, accepted):** Made concrete — "exactly zero `535` matches across debian-trixie.md, debian-ubuntu/README.md, ubuntu.md after edit." Confirmed no legitimate historical `535` exists in those three files (the only sites are the version claims being corrected). Renamed test to `test_debian_no_535_in_main_guide` with a `grep -c … → 0` assertion.

### C-004: T-004's Hyprland-`cursor{}` clause cannot be satisfied in the env-var files
- **Where:** build-plan.md T-004 clause 1; test-plan.md `test_hypr_cursor_config_present`.
- **Failure mode:** granularity / plan-test-mismatch
- **Suggested response:** fix-in-plan.
- **RESPONSE (fix-in-plan, accepted):** Split T-004's success criteria into three EARS clauses: (1) README gets the `hyprland.conf` cursor block + "deprecated" label; (2) artix.md/quick-reference.md retain the env var with a deprecation label; (3) the LogOS mention gets the predates-0.42 flag. Tests split accordingly (`test_hypr_cursor_config_present` README-only; `test_envvar_files_labelled` for the other two).

### C-005: `test_wlr_var_deprecated_framing` unmeasurable + risk to the Sway/Artix retention
- **Where:** test-plan.md `test_wlr_var_deprecated_framing`; research §4 risk 2.
- **Failure mode:** plan-test-mismatch
- **Suggested response:** fix-in-plan.
- **RESPONSE (fix-in-plan, accepted):** Replaced with checkable tests: `test_readme_deprecated_token` (literal "deprecated" near first mention), `test_sway_retention_preserved` (Sway note must survive — explicitly protects the retention), and `test_logos_predates_flag` (manual: LogOS behavior described accurately, only flagged as pre-0.42, NOT reworded). The "Confirmed in LogOS" line will be flagged, not rewritten.

### C-006: T-003 and T-004 share two files, both Depends-on (none)
- **Where:** build-plan.md T-003/T-004 Touches.
- **Failure mode:** hidden-dep
- **Suggested response:** defer-with-rationale or fix-in-plan.
- **RESPONSE (fix-in-plan, accepted):** Made T-004 `Depends on: T-003` so the shared `arch-artix/README.md` and `quick-reference.md` commits stay linear. The edited regions don't overlap (Step 0 picker vs wlroots section), so risk was low, but the explicit ordering removes the ambiguity the commit-granularity test depends on.

### C-007: T-005 mints a "surgical edits" ADR from a research recommendation
- **Where:** build-plan.md T-005 clause 1; research §5.
- **Failure mode:** ignored-ADR (mild)
- **Suggested response:** defer-with-rationale.
- **RESPONSE (defer-with-rationale, accepted):** `decisions.md` is empty so no prior ADR is ignored. It's correct to mint this sprint's decision now; the entry will use the literal word "surgical" and an ISO date so `test_decisions_has_adrs` and the dated-ADR convention both pass. No plan change needed.

## Confidence
proceed-with-caveats → all concerns addressed in-plan (C-001–C-006 fixed, C-007 deferred with rationale). Cleared to lock.
