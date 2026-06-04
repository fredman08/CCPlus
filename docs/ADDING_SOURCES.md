# Adding Sources for Claude Code Updates — CCPlus

Adding a source is just editing one file — `config/sources.yaml` — and pushing it so the weekly
cloud workflow picks it up. Follow these five steps.

---

## Step 1 — Open the file and choose a section

Edit [`config/sources.yaml`](../config/sources.yaml). It has two lists plus search phrases:

- **`authoritative:`** — official / primary sources (Anthropic docs, changelog, GitHub
  releases). High trust; a single new item here can trigger a new artifact version.
- **`community:`** — Reddit, Hacker News, blogs, experts. Needs corroboration (≥2 sources)
  before it counts as significant.
- **`search_seeds:`** — phrases the generator can *search* for, rather than fixed pages.

Add your source under whichever list fits.

## Step 2 — Add an entry

Each source uses these fields:

| Field    | Meaning                                                                 |
| -------- | ----------------------------------------------------------------------- |
| `name`   | Human label shown in the run's reasoning                                |
| `url`    | The page / feed / API to read                                           |
| `type`   | `authoritative` or `community`                                          |
| `weight` | 1 (low) – 5 (high); higher = more influence on the significance decision |
| `method` | How to fetch: `webfetch`, `websearch`, `github_api`, `rest`, or `cli`   |
| `notes`  | Guidance for the generator (what to look for, caveats)                  |

**Example — add two community sources:**

```yaml
community:
  # ... existing entries ...

  - name: Simon Willison's blog (Claude tag)
    url: https://simonwillison.net/tags/claude/
    type: community
    weight: 3
    method: webfetch
    notes: Reputable practitioner; often surfaces real techniques. Corroborate before "hack".

  - name: Anthropic YouTube — Claude Code
    url: https://www.youtube.com/@anthropic-ai/videos
    type: community
    weight: 2
    method: websearch
    notes: Scan titles for Claude Code feature walkthroughs; transcript optional.
```

For an **authoritative** feed (e.g. a new official changelog), add it under `authoritative:`
with `weight: 4–5` and `method: webfetch`.

## Step 3 — (Optional) add a discovery search phrase

To have the generator *search* for a topic instead of reading a fixed page, add to
`search_seeds:` (`{month}` / `{year}` are auto-filled):

```yaml
search_seeds:
  - "Claude Code MCP servers {year}"
  - "Claude Code subagents workflow {month} {year}"
```

## Step 4 — Push it (required — the engine runs in the cloud)

The weekly run executes on GitHub's runner, so local edits only take effect after you push:

```powershell
git -C C:\Users\user\CCPlus add config/sources.yaml
git -C C:\Users\user\CCPlus commit -m "Add sources: Simon Willison, Anthropic YouTube"
git -C C:\Users\user\CCPlus pull --rebase
git -C C:\Users\user\CCPlus push
```

## Step 5 — (Optional) test immediately

Actions tab → **CCPlus weekly** → **Run workflow** with **force** checked, so it actually
researches the new sources and builds a version even if the rubric isn't strictly met. Then
check the artifact's **References** section to confirm your source was used.

---

## Tips

- **`method` matters in CI:** WebSearch may be unavailable on the GitHub runner, so `webfetch`
  (a fixed URL) and `github_api` / `rest` (JSON endpoints) are the most reliable there. Prefer a
  direct URL over a search phrase when you can.
- **Tuning trust:** the corroboration threshold for community "hacks" lives in
  [`config/config.yaml`](../config/config.yaml) as `community_corroboration_min` (default 2) —
  raise it to be stricter, lower it to surface more.
- **Authoritative ≠ unlimited:** adding many noisy authoritative sources makes almost every week
  "significant." Reserve `authoritative` for genuinely official feeds.
