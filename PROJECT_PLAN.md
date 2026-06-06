# CCPlus — Project Plan

*Automated pipeline that tracks and surfaces what's new in Claude Code — updates, tips,
techniques, and "hacks" — and publishes it as a versioned, cited, Anthropic-styled artifact.*

- **Repo:** https://github.com/fredman08/CCPlus
- **Live site:** https://fredman08.github.io/CCPlus/
- **Started:** June 4, 2026 · **Status:** v1.0 live

---

## 1. Context

The owner (a Claude Code beginner on an enterprise plan) plans to rely heavily on
Claude Code for project delivery. Because the AI space changes weekly, they want an automated
pipeline that researches and surfaces what's new from Anthropic, the Claude community, and AI
experts, and publishes it as a versioned, citation-backed, shareable document — generated weekly
or on demand, and skipped (notification only) when nothing significant has changed.

## 2. Requirements

| # | Requirement | Status |
|---|-------------|--------|
| 1 | Artifact named `CCPlus_<YYYY-MM-DD_HHmm>` | ✅ |
| 2 | Anthropic/Claude documentation look & feel | ✅ |
| 3 | History log documenting updates from the prior version | ✅ |
| 4 | Proper citations + a References section at the bottom | ✅ |
| 5 | v1.0 covers all recommended default-config improvements to date | ✅ |
| 6 | Never overwrite — a new artifact per version | ✅ (timestamped files + Releases) |
| 7 | Accessible / shareable to others | ✅ (public GitHub Pages + Releases) |
| 8 | Weekly schedule **or** by request/trigger | ⏳ Routine pending; `/ccplus` ready |
| 9 | No artifact if nothing significant — notify instead | ✅ logic; ⏳ email flow pending |
| 10 | Cross-model review (Opus ⇄ Sonnet) for accuracy | ✅ |

## 3. Chosen approach

- **Engine:** native Claude Code **Routine** (cloud, weekly cron), running
  `prompts/generate-ccplus.md`.
- **Schedule:** Mondays 1:00 PM Asia/Manila (cron `0 13 * * 1`, = `0 5 * * 1` UTC); on-demand via
  the `/ccplus` skill or the routine's "Run now".
- **Format:** self-contained Anthropic-styled HTML.
- **Hosting / sharing:** public GitHub Pages from `docs/`, plus a GitHub Release per version.
- **Notifications:** Outlook email via a Power Automate cloud flow that watches
  `state/runlog.json` (covers both new-artifact and "no significant updates" cases).
- **Cross-model review:** routine runs on Opus; spawns a Sonnet reviewer subagent to fact-check
  every claim against its citation before publish (roles swap if the base model is Sonnet).

## 4. Architecture

```
Weekly cron (Mon 13:00)  ─┐
/ccplus (on demand)       ├─►  generate-ccplus.md (Opus)
Routine "Run now"        ─┘        │
                                   ▼
        1. Gather   — WebSearch/WebFetch over config/sources.yaml → structured JSON
        2. Dedupe   — drop items already in state/state.json
        3. Significance — config/config.yaml rubric → generate OR notify-only
              │ notify-only → append runlog (no-update) → commit → STOP
              ▼ significant
        4. Generate — fill templates/template.html → docs/artifacts/CCPlus_<ts>.html
        5. Review   — Sonnet subagent (review-ccplus.md): pass / pass_with_edits / fail
        6. Publish  — update state + runlog, regenerate docs/index.html, commit & push,
                      create GitHub Release
                                   ▼
        Power Automate watches runlog.json → Outlook email (new version OR "no updates")
```

## 5. Repository layout

```
CCPlus/
  config/sources.yaml          # tracked sources (authoritative vs community)
  config/config.yaml           # significance rubric + model choices
  prompts/generate-ccplus.md   # the generator the routine runs
  prompts/review-ccplus.md     # cross-model reviewer instructions
  templates/template.html      # Anthropic-styled HTML shell ({{tokens}})
  state/state.json             # dedup ledger + version/changelog/hash tracking
  state/runlog.json            # per-run status log (drives notifications)
  docs/index.html              # GitHub Pages landing page (all versions, newest first)
  docs/artifacts/CCPlus_*.html # immutable artifact per version
  .claude/skills/ccplus/       # /ccplus on-demand trigger
  SETUP.md                     # Routine + Power Automate setup steps
  PROJECT_PLAN.md              # this file
```

## 6. Significance rubric (config/config.yaml)

**Generate** if any new item is: a Claude Code release changing features/defaults/flags/config
(not pure bugfix); new official Anthropic guidance on optimizing the default config; a new
default model or meaningful capability/limit change; or a community technique corroborated by
≥2 reputable sources that materially improves the default workflow.
**Notify-only** if items are merely cosmetic/typo/minor, single-source/unverified hype, or
already in the state ledger.

## 7. Build status

**Done**
- Repo scaffolded, committed, pushed; GitHub Pages enabled and serving (HTTP 200).
- v1.0 artifact generated (Opus 4.8), cross-model reviewed (Sonnet 4.6 → `pass_with_edits`,
  two corrections applied), published to Pages and Release `v1.0`.
- `index.html`, `state.json`, `runlog.json` initialized; `/ccplus` skill and `SETUP.md` in place.

**Engine note (June 2026):** Claude Code on the web / Routines are **disabled** for this account,
so the active engine is **GitHub Actions** (`.github/workflows/ccplus.yml`), committed but dormant
until a credential secret is added. `/ccplus` on demand works today.

**Remaining (owner/admin actions — not automatable)**
1. **Activate the engine** — add one secret to enable GitHub Actions: `ANTHROPIC_API_KEY` (PAYG),
   `CLAUDE_CODE_OAUTH_TOKEN` (subscription), or wire Bedrock/Vertex creds. See `SETUP.md` §A2.
   *(If web access is later enabled, switch to the no-key Routine instead — `SETUP.md` §A.)*
2. **Build the Power Automate Outlook flow** watching `state/runlog.json`. See `SETUP.md` §B.
3. *(Interim)* run `/ccplus` weekly on demand until automation is activated.

## 8. Fallbacks (if Claude Code on the web can't be enabled)

- **Local headless** — Task Scheduler / Power Automate Desktop runs `claude -p ...` weekly using
  the existing CLI login. No web entitlement, no API key; machine must be on at run time.
- **GitHub Actions** — fully unattended cloud runs; requires a Console **pay-as-you-go**
  `ANTHROPIC_API_KEY` (not subscription credentials) and incurs token billing.

## 9. Verification

- v1.0 renders with Anthropic styling; citations/References/History Log present; no stray
  template tokens; public Pages URL opens (verified HTTP 200).
- No-overwrite: each run writes a new timestamped file; `index.html` lists all; Releases archive
  each version.
- Notify path: a run with the state ledger already current yields `no-update` in `runlog.json`
  and an email instead of an artifact.
- Cross-model review: reviewer runs on the opposite model; verdict recorded in the artifact and
  `runlog.json`.

## 10. Maintenance

Tune `config/config.yaml` significance thresholds after the first runs; add sources in
`config/sources.yaml`; adjust shared styling in `templates/template.html`.
