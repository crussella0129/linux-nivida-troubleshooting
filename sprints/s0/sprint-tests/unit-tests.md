# Sprint 0 Unit Tests (doc-lint)

> These are documentation-lint `grep` assertions, not software tests (this repo is a doc repo). Run via the batch verification script in the Test phase. **17/17 PASS, 0 FAIL.** Hardware behavior is NOT tested here — deferred to other machines (see e2e-tests.md).

| Test (EARS-derived) | Task | Result |
|---------------------|------|--------|
| `test_debian_main_version_is_550` | T-001 | PASS (3 matches of `550.163.01`) |
| `test_debian_no_535_in_main_guide` | T-001 | PASS (0 matches of `535`) |
| `test_debian_kernel_gotcha_present` | T-001 | PASS (`6.12`/`6.19` note present) |
| `test_debian_x11_conclusion_intact` | T-001 | PASS (`555` floor reasoning present) |
| `test_index_version_updated` | T-002 | PASS (0 `535` in README.md + ubuntu.md) |
| `test_root_readme_untouched` | T-002 | PASS (root README has no `535`) |
| `test_index_shows_550` | T-002 | PASS (family README shows 550) |
| `test_arch_open_default_turing` | T-003 | PASS (`nvidia-open` in README + quick-ref) |
| `test_arch_open_modules_note` | T-003 | PASS (`560`/open-modules note present) |
| `test_arch_proprietary_legacy` | T-003 | PASS (Maxwell/Pascal/Volta mentioned) |
| `test_readme_deprecated_token` | T-004 | PASS (`deprecated` near WLR mention) |
| `test_hypr_cursor_config_present` | T-004 | PASS (`no_hardware_cursors = true` block) |
| `test_envvar_files_labelled` | T-004 | PASS (artix.md + quick-ref labelled) |
| `test_sway_retention_preserved` | T-004 | PASS (Sway note retained) |
| `test_logos_predates_flag` | T-004 | PASS (manual: LogOS flagged as pre-0.42, behavior unchanged) |
| `test_decisions_has_adrs` | T-005 | PASS (both ADRs with dates) |
| `test_sprint_summary_set` | T-005 | PASS (summary line set) |

All EARS clauses in `build-plan.md` have a corresponding passing check.
