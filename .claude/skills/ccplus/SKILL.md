---
name: ccplus
description: Generate a CCPlus "what's new in Claude Code" artifact on demand. Researches Anthropic + community sources, decides significance, and (if significant) publishes a new versioned, Anthropic-styled HTML artifact with cross-model review. Use when the user types /ccplus or asks to run/refresh the CCPlus digest now.
---

# /ccplus — run the CCPlus pipeline on demand

Run the CCPlus generator immediately in the current session. This is the same logic the weekly
Routine runs; use it for an ad-hoc refresh or a forced build.

## Steps

1. From the repo root, read and follow `prompts/generate-ccplus.md` end to end, using
   `config/config.yaml` and `config/sources.yaml`.
2. Honor any argument:
   - `force` → generate even if the significance rubric says notify-only (still run the
     cross-model review and version bump).
   - default (no arg) → respect the significance rubric (may end as notify-only).
3. Run the cross-model review via `prompts/review-ccplus.md` on the opposite model.
4. Update `state/state.json`, `state/runlog.json`, regenerate `docs/index.html`, commit, push,
   and create the GitHub Release (skip release if not authenticated).
5. Report back: the version produced (or `no-update`), the artifact path, the review verdict,
   and the published URL.

## Notes
- Never overwrite an existing artifact file.
- If web access is unavailable, stop and report rather than guessing — accuracy over volume.
