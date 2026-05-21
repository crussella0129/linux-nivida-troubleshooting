# Sprint 0 Integration Tests (doc-lint)

> Cross-document consistency checks. **All PASS.**

| Test | Result |
|------|--------|
| `test_debian_version_consistent_across_docs` — debian-trixie.md, debian-ubuntu/README.md, ubuntu.md all state 550 for Debian main; none disagree | PASS |
| `test_arch_cross_doc_consistency` — arch-artix/README.md and quick-reference.md agree on open-default-for-Turing+ and WLR deprecation; no contradiction | PASS |
| `test_internal_anchors_resolve` — no heading text in edited sections was renamed; the referenced anchor `#wlroots-compositors-hyprland--sway--required-nvidia-env-vars` and others still match their headings (manual inspection) | PASS |
| `test_diff_scope` — `git status --porcelain` empty; `git log --oneline` shows distinct commits sprint-0: T-001..T-005 | PASS |

Commit boundaries: `7bb096f` (T-001), `c2b3604` (T-002), `dc37636` (T-003), `b16b0de` (T-004), `60c1521` (T-005).
