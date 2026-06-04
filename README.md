# CCPlus — Claude Code Plus

An automated pipeline that tracks and surfaces **what's new** in Claude Code — updates, tips,
tricks, techniques, and "hacks" — from Anthropic, the Claude community, and AI experts, and
publishes it as a versioned, citation-backed, Anthropic-styled HTML document.

**Live site:** https://fredman08.github.io/CCPlus/ &nbsp;|&nbsp; **Versions:** see
[Releases](https://github.com/fredman08/CCPlus/releases)

---

## What it does

Every week (Mondays 1:00 PM, or on demand) the pipeline:

1. **Researches** the sources in [`config/sources.yaml`](config/sources.yaml).
2. **Detects significance** — compares findings against [`state/state.json`](state/state.json)
   using the rubric in [`config/config.yaml`](config/config.yaml).
3. If nothing significant is new → **notifies only** (no artifact), logging `no-update` to
   [`state/runlog.json`](state/runlog.json).
4. If there are significant updates → **generates** a new
   `docs/artifacts/CCPlus_<YYYY-MM-DD_HHmm>.html`, **cross-model reviews** it (Opus ⇄ Sonnet),
   updates the history log, refreshes [`docs/index.html`](docs/index.html), creates a GitHub
   Release, and commits.

Prior versions are **never overwritten** — each run produces a new timestamped file.

## How it runs

- **Weekly / scheduled & on-demand "run now":** a Claude Code **Routine** at
  [claude.ai/code/routines](https://claude.ai/code/routines), pointed at
  [`prompts/generate-ccplus.md`](prompts/generate-ccplus.md). See [`SETUP.md`](SETUP.md).
- **On demand in any local session:** the `/ccplus` skill
  ([`.claude/skills/ccplus/SKILL.md`](.claude/skills/ccplus/SKILL.md)).
- **Notifications:** a Power Automate flow watches this repo and emails the run summary
  (including the "no significant updates" case). See [`SETUP.md`](SETUP.md).

## Repository layout

```
CCPlus/
  config/sources.yaml          # tracked sources (authoritative vs community)
  config/config.yaml           # significance rubric + model choices
  prompts/generate-ccplus.md   # the generator the routine runs
  prompts/review-ccplus.md     # cross-model reviewer instructions
  templates/template.html      # Anthropic-styled HTML shell
  state/state.json             # dedup ledger + version/changelog/hash tracking
  state/runlog.json            # per-run status log
  docs/index.html              # GitHub Pages landing page (all versions, newest first)
  docs/artifacts/CCPlus_*.html # immutable artifact per version
  .claude/skills/ccplus/       # /ccplus on-demand trigger
  SETUP.md                     # Routine + Power Automate setup steps
```

## Status legend (runlog)

| status | meaning |
|--------|---------|
| `generated` | a new artifact was produced and published |
| `no-update` | nothing significant; notification only |
| `error`     | run failed; see message |

---

*CCPlus is generated with Claude Code. Content is paraphrased from cited public sources; see
each artifact's References section.*
