# CCPlus — Setup & Operations

This guide finishes wiring CCPlus after the repo is created and v1.0 is published. There are
three things to set up: **(A)** the weekly Routine, **(B)** the Outlook-email notification flow
in Power Automate, and **(C)** the on-demand trigger. A **fallback** section covers what to do if
Claude Code on the web can't be enabled.

> **Prerequisite — Claude Code on the web (hard gate for the Routine).** Routines require
> "Claude Code on the web" to be enabled for your account. This is an Anthropic org-side setting
> that an admin requests from your Anthropic account team — it can't be toggled by a user or
> automated. Check at <https://claude.ai/code>; if it says *"Not available for the selected
> organization,"* it's disabled — request enablement, or use the **Fallback** below.

---

## A. Weekly Routine (cloud, native)

1. Go to <https://claude.ai/code/routines> and create a new routine.
2. **Connect this repo** (`fredman08/CCPlus`) via the GitHub connector (authorize once).
3. **Schedule:** weekly, cron `0 13 * * 1` (Mondays 13:00). Set the timezone to **Asia/Manila**
   (or your local zone). If the UI only accepts UTC, use `0 5 * * 1` for Mon 13:00 PHT.
4. **Model:** Opus. The reviewer subagent runs on Sonnet automatically (see
   `prompts/generate-ccplus.md` → Step 5).
5. **Network access:** Trusted (or allow the source domains in `config/sources.yaml`).
6. **Branch pushes:** enable **"Allow unrestricted branch pushes"** so the routine can publish to
   `main` (otherwise it pushes to a `claude/` branch and you'll merge each run).
7. **Prompt / instructions:** point the routine at the repo and use:
   > *Follow `prompts/generate-ccplus.md` exactly, using `config/config.yaml` and
   > `config/sources.yaml`. Generate a new CCPlus version only if the significance rubric is met;
   > otherwise record a `no-update` run. Commit and push, and create a GitHub Release on generate.*
8. Click **Run once now** to confirm an end-to-end run works.

The routine draws on your Enterprise **subscription usage** (no API key), subject to a daily
routine-run cap. View run history on the routines page.

---

## B. Outlook email notifications (Power Automate webhook)

The GitHub workflow POSTs each run's summary to a Power Automate flow, which emails you. This
uses only **standard** connectors (works on a free Power Automate license), fires for both
scheduled and on-demand runs, and needs just two actions. The workflow step is already in
`.github/workflows/ccplus.yml` — it stays dormant until the `PA_WEBHOOK_URL` secret is set.

### B1. Build the flow (in Power Automate)

At <https://make.powerautomate.com> → **+ Create → Instant cloud flow** →
**Skip** (choose trigger manually):

1. **Trigger:** search **"When a HTTP request is received"** (Request). Open it and paste this
   into **Request Body JSON Schema**:
   ```json
   {
     "type": "object",
     "properties": {
       "status":    { "type": "string" },
       "version":   { "type": "string" },
       "subject":   { "type": "string" },
       "summary":   { "type": "string" },
       "url":       { "type": "string" },
       "timestamp": { "type": "string" }
     }
   }
   ```
2. **Action:** **+ New step** → **Office 365 Outlook** → **Send an email (V2)**:
   - **To:** `fredman08@users.noreply.github.com`
   - **Subject:** insert the `subject` dynamic field.
   - **Body:** type a line then insert dynamic fields, e.g.
     `{summary}` … `View: {url}` … `(status: {status}, run {timestamp})`.
3. **Save.** Reopen the trigger and copy the generated **HTTP POST URL** (a long
   `https://prod-…logic.azure.com/…?sig=…` link). Treat it like a secret.

### B2. Give the workflow the URL

Add it as a repo secret (from a terminal, hidden input):
```powershell
gh secret set PA_WEBHOOK_URL --repo fredman08/CCPlus
```
(or via the GitHub web UI → Settings → Secrets → Actions). Done — the next run emails you.

### B3. Test

Actions tab → **CCPlus weekly** → **Run workflow**. You should receive an email within a minute
("no significant updates" if nothing changed). To test the "new version" email, run it with the
**force** input checked.

> Notes: the email fires whether or not a new artifact was produced. If the **Request** trigger
> isn't available on your license, the simplest alternative is to skip Power Automate and let
> **GitHub email you** — Watch the repo (top-right **Watch → All Activity**) so you're notified on
> each `ccplus-bot` commit and new Release.

---

## C. On-demand ("by request")

- **In any Claude Code session** (VSCode or CLI) opened on this repo: run **`/ccplus`** (force a
  build with `/ccplus force`). See `.claude/skills/ccplus/SKILL.md`.
- **GitHub Actions:** Actions tab → **CCPlus weekly** → **Run workflow** (check **force** to build
  even with no significant updates).

---

> **Note:** As of June 2026, Claude Code on the web is **disabled** for this account, so the
> Routine (section A) is not available yet. The active engine is **GitHub Actions** below. If
> web access is later enabled, switch to the Routine (it needs no API key).

---

## A2. GitHub Actions (active engine while Routines are unavailable)

The workflow is committed at `.github/workflows/ccplus.yml`. It runs weekly (Mon 05:00 UTC =
13:00 Asia/Manila) and via the **Run workflow** button (on-demand). It is **dormant until a
credential secret is added**, so scheduled runs no-op cleanly until you activate it.

### Activating

Add **one** credential as a repo secret at
`https://github.com/fredman08/CCPlus/settings/secrets/actions`:

| Option | Secret | Notes |
|--------|--------|-------|
| **A. PAYG key** | `ANTHROPIC_API_KEY` | A Console pay-as-you-go key from console.anthropic.com. **Not** subscription credentials. ~$0.50–$2 per run. |
| **B. OAuth token** | `CLAUDE_CODE_OAUTH_TOKEN` | Generated via `claude setup-token`; uses a Claude subscription instead of PAYG. Availability depends on your Enterprise plan — confirm with your admin. |
| **C. Bedrock/Vertex** | (cloud creds) | If your organization already has Claude via AWS Bedrock / Google Vertex, authenticate the action with cloud credentials instead of an Anthropic key. Requires editing the workflow per the action's provider docs. |

Then: **Actions tab → CCPlus weekly → Run workflow** to test. New artifacts publish to GitHub
Pages automatically (push to `main` rebuilds Pages) and a GitHub Release is created.

> Verify the `anthropics/claude-code-action` version and input names against its current docs
> before enabling — the action evolves frequently.

---

## Other fallbacks

**Local headless** (no API key — uses your CLI login; PC must be on at run time):

1. Install the standalone CLI: `npm install -g @anthropic-ai/claude-code` (you currently have
   only the VS Code extension), then sign in once with `claude` and confirm `claude --version`.
2. Schedule weekly (Mon 13:00) via **Task Scheduler** / **Power Automate Desktop**, running from
   the repo directory:
   ```powershell
   claude -p "Follow prompts/generate-ccplus.md ..." --permission-mode acceptEdits
   ```
3. Keep the same repo, Pages, and Power Automate email flow.

**Manual on-demand only** (zero infrastructure): run **`/ccplus`** in VS Code whenever you want
(e.g. a weekly calendar reminder). No CLI install, no key, no machine-on requirement. This works
right now and is the recommended interim path until an Actions credential is sorted.

---

## Maintenance

- **Tune significance** in `config/config.yaml` after the first few runs (too noisy → raise
  thresholds; missing things → lower them).
- **Add sources** in `config/sources.yaml`.
- **Adjust styling** in `templates/template.html` (used for every new version).
