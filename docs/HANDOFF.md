# Handoff

State: planning only. Branch `spec` holds `CLAUDE.md`, `docs/SPEC.md`, `docs/DECISIONS.md` and `.claude/`; its PR is open, not merged. No code exists.
Next task: T01 Scaffold (SPEC §12), once the spec PR is merged and Q1 is answered.
Open questions for the owner (recommendations in SPEC §13):
- Q1 maturin from T01, or pure Python until 0.2.0? Blocks T01.
- Q2 Anthropic SDK as an optional extra? Blocks T11. Q3 default model for CLI and examples? Blocks T13.
- Q4 vendor the GPT-2 files? Blocks T21. Q5 benchmark corpus titles? Blocks T26.
- Q6 baseline shape, checked with a real key in T18. Q7 is the name `promptshape` free on PyPI and TestPyPI? Part of T17.
Not available yet: an API key (needed only for T18); GitHub environments and PyPI publishers (T17).
