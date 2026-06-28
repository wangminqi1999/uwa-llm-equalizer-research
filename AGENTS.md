# AI Collaboration Notes

This repository is a research workspace for underwater acoustic communication and LLM-based physical-layer sequence detection/equalization.

Before planning or editing research code, read these files first:

1. `PROJECT_HANDOFF.md`
2. `2026-06-08-uwa-llm-equalizer-implementation-plan.md`
3. `AI_RESEARCH_COLLABORATION_GUIDELINES.md`
4. `README.md`

Important working rules:

- Treat modulation, channel model, SNR definition, baseline, data shape, and model I/O as research decisions. Explain the basis before changing them.
- Reuse or adapt existing reference implementations where they fit the project goal.
- Keep changes traceable and beginner-readable.
- Do not commit generated caches, virtual environments, local IDE settings, training logs, or temporary checkpoints.
- Preserve papers, reference code, data files, and top-level handoff documents unless the project owner explicitly asks to remove them.
