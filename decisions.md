# Architectural Decisions

## 2026-05-20 — Recommend NVIDIA open kernel modules by default for Turing+
On Arch (and reflected across the guides), the **open** kernel modules (`nvidia-open`/`nvidia-open-dkms`) are the default recommendation for Turing-architecture GPUs (RTX 20-series) and newer, because NVIDIA made open the default from the 560 driver series and Arch's `nvidia` package now ships open modules. Proprietary `nvidia`/`nvidia-dkms` is reserved for legacy Maxwell/Pascal/Volta GPUs the open modules don't support. (Sprint 0)

## 2026-05-20 — Maintain guides via surgical edits, not rewrites
When correcting verified factual drift, prefer **surgical** in-place edits to the affected sentences/tables over wholesale rewrites. Rationale: lower risk of introducing new errors, smaller review surface, and it preserves the hard-won prose and the "X11-first on Optimus" thesis. Full rewrites are reserved for structural reorganizations. (Sprint 0)
