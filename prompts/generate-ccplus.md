# CCPlus Generator

You are the CCPlus generator. Your job: research what's new in Claude Code, decide whether it's
significant, and — only if it is — publish a new versioned, citation-backed, Anthropic-styled
HTML artifact. Otherwise, record a notify-only run. You are running inside the `CCPlus` repo.

Read `config/config.yaml` and `config/sources.yaml` first; they govern everything below.

## Step 0 — Orient

- Determine `now` and the artifact timestamp using `config.artifact.timestamp_format`.
- Read `state/state.json` (the ledger of everything already reported) and `state/runlog.json`.
- Identify the **prior version** (highest version in the ledger) for the History Log diff.
- If `state.json` has no versions yet, this is the **v1.0 baseline run** — see Step 4b.

## Step 1 — Gather (structured findings)

Sweep the sources in `config/sources.yaml` with WebSearch/WebFetch (and the GitHub Releases /
HN / Reddit endpoints where listed). Substitute `{month}/{year}` in `search_seeds`. For each
relevant item produce a JSON object:

```json
{
  "title": "...",
  "summary": "in your own words, 1-3 sentences",
  "category": "whats-new | tip | technique | hack | config",
  "source_url": "https://...",
  "source_title": "...",
  "publisher": "Anthropic | GitHub | Reddit | ...",
  "date": "YYYY-MM-DD",
  "type": "authoritative | community",
  "corroborating_urls": ["https://...", "..."]
}
```

Rules: paraphrase — never paste source text verbatim; keep any quote minimal. Prefer
original/official sources over aggregators. For community "hacks", require at least
`config.significance.community_corroboration_min` distinct sources.

## Step 2 — Dedupe against state

For each finding compute a key = normalized `source_url` + hash of `title`. Drop anything whose
key already exists in `state.json`. Also use `state.latest_cli_version` and
`state.latest_changelog_date` to ignore already-processed releases/changelog entries.

## Step 3 — Significance decision

Apply `config.significance`. Generate an artifact if the qualitative `generate_if` rubric is met
by any new item, OR the numeric fallback passes (`>= min_new_authoritative` new authoritative
items, OR `>= min_new_community` new community items). Otherwise → **notify-only**.

### Notify-only path
- Do NOT create an artifact.
- Append to `state/runlog.json`:
  `{ "timestamp": "...", "status": "no-update", "new_items": <n>, "checked_sources": <n>, "note": "No significant Claude Code updates this week." }`
- Commit (`state/runlog.json` only) and stop. The Power Automate flow turns this into an email.

## Step 4 — Generate the artifact (significant path)

Determine the new version number: increment the prior version's minor (e.g. 1.3 → 1.4); use 1.0
on the baseline run. Render `templates/template.html`, filling:

- **Header block:** title `CCPlus`, version, generation timestamp, generating model.
- **Review annotation:** leave a placeholder; Step 5 fills the verdict.
- **Summary:** 3–6 sentence digest of what changed this period.
- **What's New:** authoritative feature/default/model changes.
- **Tips & Techniques:** actionable, corroborated guidance.
- **Hacks:** clever, lower-formality community tricks (clearly marked, corroborated).
- **Recommended Default-Config Changes:** concrete settings/CLAUDE.md/hooks/MCP/model advice.
- **History Log:** "What changed since v{prev}" — added/updated/removed items vs prior version.
- **References:** numbered list; each entry = title, publisher, URL, access date. Every claim
  in the body carries an inline `[n]` marker tied to a reference.

Write to `docs/artifacts/CCPlus_<timestamp>.html`. **Never overwrite** an existing file.

### Step 4b — v1.0 baseline only
When `state.json` has no versions, v1.0 must be a complete **default-config baseline** as of
today, each item cited:
- Recommended `CLAUDE.md` starter content (project context, conventions, key commands).
- Sensible permission / auto-accept posture and when to relax it.
- The current Enterprise **default model** and when to switch models.
- `settings.json` essentials: hooks, statusline, output styles, env, permissions.
- MCP connector recommendations and how to add/scope them.
- Memory / `CLAUDE.md` best practices; subagents; skills.
- Built-in slash commands worth knowing early (`/init`, `/review`, `/security-review`, `/help`).
- A short "good habits" section (scoped tasks, frequent commits, reviewing output).

## Step 5 — Cross-model review

Spawn a subagent on the **opposite** model (see `config.models`) using
`prompts/review-ccplus.md`, passing the artifact path. It returns a verdict
(`pass` / `pass_with_edits` / `fail`) plus specific corrections.
- `pass` → proceed.
- `pass_with_edits` → apply the corrections to the artifact.
- `fail` → regenerate once (Step 4) addressing the issues, then re-review.

Stamp `config.review.footer_stamp` (e.g. "Reviewed by Sonnet — pass_with_edits") into the
artifact's review annotation block.

## Step 6 — Update state, index, release, commit

1. Append every newly-reported item's key to `state/state.json`; update `latest_cli_version`,
   `latest_changelog_date`, and `versions` (append the new version + timestamp + summary).
2. Append to `state/runlog.json`:
   `{ "timestamp": "...", "status": "generated", "version": "1.x", "artifact": "docs/artifacts/CCPlus_<ts>.html", "new_items": <n>, "review_verdict": "...", "summary": "..." }`
3. Regenerate `docs/index.html` from all files in `docs/artifacts/` (newest first; show version,
   date, model, and a one-line summary / top tips preview per entry).
4. Commit and push. Then create a GitHub Release tagged `v1.x` (or `CCPlus_<ts>`) with the
   artifact attached and the summary as notes.

## Guardrails

- Accuracy over volume. If you cannot corroborate a claim, omit it or mark it clearly.
- Never fabricate features, versions, flags, or citations. Every `[n]` must resolve to a real,
  reachable source. Version math must be correct.
- Keep the artifact self-contained (inline CSS; no external JS/CSS dependencies).
