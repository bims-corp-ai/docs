# Onboarding QA + PM to the BIMS-Corp AI Bug-Hunting Environment

**Audience:** QA Engineer and Project Manager / QA (NY office, Windows 11)
**Time to first bug:** ~60 minutes for first-time install; ~30 seconds per bug after that
**What this gets you:** an AI pair-programmer that already knows all 8 BIMS-Corp codebases (LSP-Web, LIP-Web, PO-Web, SA-Web, and the 4 mobile apps) — you ask questions in plain English, it answers with `file:line` citations, helps you confirm bugs, and drafts complete Jira tickets you paste into the queue.

---

## TL;DR — what you're about to set up

You're installing one CLI tool (Claude Code), one Linux compatibility layer (WSL2 — Microsoft's first-party feature, free, built into Windows 11), and cloning two folders from GitHub. Total disk: ~3 GB. The AI talks to you through your terminal; it has read-only access to all the BIMS-Corp source code and can already navigate it with `file:line` precision because we pre-built knowledge graphs for every repo.

You don't write code. You don't push anything to production. Your loop is:

1. You: "I think there's a bug in the invoice email link expiration"
2. AI: "Reading `app/Repositories/InvoiceRepository.php:1495`. Found one — payment-link URLs use a custom `encrypt_decrypt()` helper at `GeneralHelper.php:370-386` that has a hardcoded key. Concrete attack: ..."
3. You: "Confirm it's a real bug and write me a Jira ticket"
4. AI: writes `tickets/draft/BIMS-F-2026-05-27-ddca6b27.md` with the full Implementation Note
5. You: paste it into Jira yourself (until the Jira API is wired)

That's the whole loop.

---

## Part 1 — One-time install (Windows 11)

### 1.1 Enable WSL2 (Windows Subsystem for Linux)

WSL2 lets you run Linux commands inside Windows. Everything in this environment is built for Linux/Mac; WSL2 is the simplest way to make Windows behave.

Open **PowerShell as Administrator** (right-click Start menu → "Terminal (Admin)" or "Windows PowerShell (Admin)") and run:

```powershell
wsl --install
```

This installs WSL2 with Ubuntu by default. **Reboot** when it tells you to. After reboot, Ubuntu opens automatically and asks for a username + password — pick anything (e.g. `qa` / a memorable password). Write it down.

If `wsl --install` says "WSL is already installed", run:
```powershell
wsl --install -d Ubuntu
wsl --set-default Ubuntu
```

After this point, every command in this guide runs inside the **Ubuntu terminal**, NOT PowerShell. You open Ubuntu from the Start menu (it's a separate app).

> **Troubleshooting:** if your machine doesn't support virtualization, `wsl --install` will tell you. Reboot into BIOS and enable "Intel VT-x" or "AMD-V" — your IT person can help. This is a one-time BIOS toggle.

### 1.2 Install the prerequisites inside Ubuntu

Open the Ubuntu app from the Start menu. You're now at a Linux prompt. Paste this **single line** and press Enter:

```bash
sudo apt update && sudo apt install -y git curl unzip jq sqlite3 python3 python3-pip build-essential
```

Type your Ubuntu password when asked. This takes ~3 minutes.

Then install `uv` (the Python package manager this project uses):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
```

And install the GitHub CLI:

```bash
(type -p wget >/dev/null || sudo apt-get install wget -y) \
&& sudo mkdir -p -m 755 /etc/apt/keyrings \
&& wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
&& sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
&& sudo apt update \
&& sudo apt install gh -y
```

Then authenticate with GitHub:

```bash
gh auth login
```

Pick **GitHub.com → HTTPS → Authenticate via browser**. It'll print a one-time code; copy it, then your browser opens to a GitHub page where you paste the code. After "✓ Authentication complete", you're done.

> **Why GitHub auth?** You need read access to the BIMS-Corp legacy repos (private). Max will add your GitHub username to the `bims-corp-ai` organization with read-only permissions on the legacy repos before the meeting. If you get "permission denied" later, tell Max your GitHub username.

### 1.3 Install Node.js (for Claude Code)

Claude Code is distributed as an npm package, so we need Node:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

### 1.4 Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:
```bash
claude --version
```

You should see something like `Claude Code 1.x.x`. If you get "command not found", run `source ~/.bashrc` and try again.

### 1.5 Log into Claude

```bash
claude
```

On first run, it'll prompt you to authenticate. Two options:

- **Option A (recommended for now):** Log in with your **claude.ai Pro subscription** ($20/mo per user; each of you will have your own). Pick "Anthropic Console" or "Claude Pro" — it opens your browser for OAuth. Max may already have a team subscription that includes you; check with him.
- **Option B:** Use an Anthropic API key. Max will provide one if Option A doesn't fit your billing setup. Paste it when prompted.

After login, you'll see the Claude Code prompt. Type `/help` to see commands. Type `/exit` for now — we'll come back.

---

## Part 2 — Clone the BIMS-Corp environment

You're cloning **two** folders side-by-side:

1. **`bims-corp-ops`** — the workbench. This is where the AI lives — it has the skills, the knowledge graphs, the finding-tracking database, the ticket drafts. ~250 MB.
2. **`BIMS-Corp-LEGACY/`** — a folder containing the 8 production code repos. The AI reads these read-only; you NEVER modify them. ~1.5 GB.

Inside Ubuntu, pick a working directory. I'll use `~/Developer` for the example — change it if you want:

```bash
mkdir -p ~/Developer
cd ~/Developer
```

### 2.1 Clone the workbench

```bash
git clone https://github.com/bims-corp-ai/bims-corp-ops.git
cd bims-corp-ops
```

### 2.2 Clone the 8 legacy repos

The workbench has a helper that does all 8 in one shot:

```bash
mkdir -p ~/Developer/BIMS-Corp-LEGACY
./bin/clone-legacy
```

This takes ~10-15 minutes depending on your connection. It clones LinkServicePro-Web, LinkInspectPro-Web, PropertyOrganizer-Web, SuperAdmin-Web, and the 4 mobile repos.

> **If clone-legacy fails on one repo:** that means your GitHub account doesn't have access yet. Ping Max with your GitHub username; he'll add you. After he confirms, re-run `./bin/clone-legacy` (it skips already-cloned repos).

### 2.3 Configure your local environment

The workbench needs a small per-machine config file. Copy the example and edit it:

```bash
cp etc/local-config.example.json state/local-config.json
nano state/local-config.json
```

Fill in these fields (the rest can stay as-is):

- `"machine_id"`: your name + machine, e.g. `"alex-qa-laptop"` or `"sam-pm-desktop"`
- `"operator"`: your email
- `"BIMS_OPS_ROOT"`: `/home/<your-ubuntu-username>/Developer/bims-corp-ops` (replace `<your-ubuntu-username>` with the username you picked in step 1.1; you can check with `whoami`)
- `"BIMS_LEGACY_ROOT"`: `/home/<your-ubuntu-username>/Developer/BIMS-Corp-LEGACY`
- `"BIMS_STORE_URL"`: for day 1, use `"file:///home/<your-ubuntu-username>/Developer/bims-corp-ops/state/bims-store.db"` (local sqlite — your findings stay on your machine for now; Max will migrate the team to a shared store later)
- `"mode"`: `"secondary"` (since you're not the primary indexing machine — Max is)
- `"sync_enabled"`: `false` (Max's machine does the legacy-repo syncing)

Delete the `BIMS_STORE_URL_BASE` and `BIMS_STORE_TOKEN_FILE` lines (those are for the shared Turso pattern; you're not using it yet).

Save (Ctrl+O, Enter, Ctrl+X in nano).

### 2.4 Bootstrap

This is the one-command setup that wires everything together:

```bash
./bin/setup --mode secondary --no-launchd --skip-indexes
```

The flags:
- `--mode secondary` matches your config
- `--no-launchd` skips Mac-only stuff
- `--skip-indexes` skips graph-building (Max has already built and committed them; you get them for free)

The bootstrap runs 11 numbered steps. Each step prints `N/11`. If a step says PASS, move on. If anything FAILs, paste the error to Max — DON'T try to fix it yourself the first time.

Expected end-state:
- `pytest -q` says ~882 tests pass
- `bin/doctor` shows about 68 PASS / 1 WARN / 7 FAIL (the 7 FAILs are EXPECTED — they're placeholders for engineering work that's still in progress; they don't affect your QA workflow)

If `bin/doctor` shows any FAILs you don't recognize, ping Max.

---

## Part 3 — First conversation with Claude

You're now ready to talk to the AI. From inside the `bims-corp-ops` directory:

```bash
claude
```

Wait a few seconds. You'll see the Claude prompt and a list of available skills (`/bims-ask`, `/bims-diagnose`, `/bims-ticket`, `/bims-promote`, and others).

Type this exact message:

```
who are you, and what can you help me with?
```

Claude will respond explaining the BIMS-Corp environment, the 8 repos it knows about, your role (advisory-tier — finding bugs, not fixing them), and the skills it can use. Read the response.

Now try a real question. Type:

```
/bims-ask how does invoice payment work in LinkServicePro-Web?
```

Claude will:
1. Look up the question in the LSP-Web knowledge graph (the `mcp__graphify-bims-lsp-web` MCP)
2. Read the precise file:line ranges that match
3. Give you a cited explanation: every claim ends with `LinkServicePro-Web/app/Repositories/InvoiceRepository.php:1442` so you can verify

That's the basic `/bims-ask` loop. Use it any time you want to understand how something works in any of the 8 repos.

---

## Part 4 — Hunt your first bug

This is the actual workflow you'll use every day. We'll walk through it once together.

### Step 4.1 — Pick a target

Bug-hunting works best when you have a TARGET. Examples:

- "Customer says the invoice payment link sometimes shows the wrong invoice"
- "Mobile app crashes when uploading a photo larger than 10MB"
- "QA found that the customer search returns deleted customers"
- Just "scan the password reset flow in PO-Web for security issues"

You don't need to know where the code lives — that's Claude's job. You provide the user-facing symptom or the area of concern.

### Step 4.2 — Ask Claude to investigate

```
/bims-ask the QA team got a customer complaint that they could see someone else's invoice by tweaking the URL. Is there anything in LinkServicePro-Web's invoice flow that would let that happen?
```

Claude will:
- Trace the invoice-URL flow through the graph
- Identify the security mechanism (in this case, `encrypt_decrypt()`)
- Read the actual implementation
- Tell you what it found, with `file:line` citations

If Claude finds nothing concerning, it'll say so explicitly. Don't push it to find a bug if there isn't one — false-positive findings waste the dev team's time.

### Step 4.3 — Record the finding

When Claude DOES find something real, ask:

```
record this as a finding
```

Claude calls `mcp__bims-store__record_finding` with the right severity rubric. It'll print the finding ID (e.g. `BIMS-F-2026-05-27-ddca6b27`) and the auto-computed severity (P0/P1/P2/P3 + CRITICAL/IMPORTANT/MINOR).

### Step 4.4 — Diagnose (optional but recommended for P0/P1)

For high-severity findings, ask for a structured diagnosis:

```
/bims-diagnose BIMS-F-2026-05-27-ddca6b27
```

This produces 3 competing hypotheses about the root cause (with falsifiability criteria for each) plus a two-track fix proposal:

- **Track A** = minimum-viable fix that ships on the current framework version
- **Track B** = the ideal fix on a modern codebase

The diagnosis lives at `diagnoses/<finding_id>/diagnosis.json`. Don't paste this whole thing to TatvaSoft — it's an internal artifact. It feeds the next step:

### Step 4.5 — Draft a Jira ticket

```
/bims-ticket BIMS-F-2026-05-27-ddca6b27
```

Claude generates `tickets/draft/<finding_id>.md` — a complete Jira-shaped ticket with:

- Title, Severity, Repo, Labels
- "Why this matters" (plain-English impact, no jargon)
- "Current behavior" / "Expected behavior"
- "Root cause" (from the diagnosis)
- "Authorized track" (you / Max marks A or B before promoting)
- **Implementation Note** — files to touch, patterns to follow, patterns to avoid, test to write, Definition of Done, complexity tier

Open the file and read it: `nano tickets/draft/BIMS-F-2026-05-27-ddca6b27.md`

If anything is wrong (Claude misunderstood the impact, the suggested files are off, etc.), tell Claude:

```
the Implementation Note has the wrong file — the actual entry point is StripeController.php not InvoiceController.php. update the draft.
```

Claude will edit the file. Iterate until you're happy.

### Step 4.6 — Promote to Jira

```
/bims-promote BIMS-F-2026-05-27-ddca6b27
```

In the current setup (no Jira API token wired yet), this prints a copy-paste-ready block:

```
==== COPY THIS INTO JIRA ====
Project: BIMS
Issue type: Bug
Summary: [Security/Crypto] encrypt_decrypt() helper uses hardcoded AES key...
Labels: ai-drafted, security, crypto, complexity-M

Description:
---
<full ticket body>
---
```

You open Jira in your browser, "Create Issue", paste the Summary into the summary field, paste the Description into the description field, set the labels, and submit. Copy the resulting `BIMS-NNNN` ID.

Then tell Claude:

```
the Jira ID for that ticket is BIMS-5402
```

Claude records it back to the finding so the audit trail is complete.

> **When the Jira API gets wired** (Max is working on it), `/bims-promote` will create the issue automatically and you'll skip the manual paste.

### Step 4.7 — Done

That's one complete bug-hunt-to-Jira cycle. The whole loop typically takes 5-15 minutes per bug after the first one.

---

## Part 5 — Daily workflow

**Start of day:**

```bash
cd ~/Developer/bims-corp-ops
git pull origin main          # get latest skills + graphs from Max
claude                        # open Claude
```

In Claude:

```
/bims-orient
```

This prints the day's project state: open findings count, recent decisions, sync status. ~3 seconds. Always start here.

**For each bug report / area you're investigating:**

1. Use `/bims-ask` to investigate
2. If real, record finding + diagnose + ticket + promote
3. If not real, just move on (no finding recorded)

**Mid-day check-in:**

```
/bims-ask show me my findings from today
```

or

```
which P0 / P1 findings are still open across all repos?
```

**End of day (if working from your own clone for now):**

If you want Max to see your tickets/draft/ files, commit and push:

```bash
git checkout -b qa/$(date +%Y%m%d)-<your-name>-tickets
git add tickets/draft/
git commit -m "QA tickets drafted $(date +%Y-%m-%d)"
git push -u origin qa/$(date +%Y%m%d)-<your-name>-tickets
```

Then create a PR on GitHub (`gh pr create`) and ping Max for review. He'll merge so the tickets are visible to everyone.

> **Future state:** once Max migrates the team to the shared Turso store, your findings appear in his `/bims-orient` automatically, no PR needed. We'll cover that when it lands.

---

## Part 6 — What to do when things go wrong

### "Claude says my finding's severity is P3 but I think it's P0"

Tell Claude. The severity rubric is computed from four facts (worst_case, trigger_likelihood, population_affected, defenses_remaining). Push back if you think Claude classified one of these wrong:

```
I think the worst_case here is integrity-bypass not silent-degrade because the bug lets attackers forge access tokens, not just see misleading info. re-rate it.
```

Claude will re-compute and explain. If you and Claude still disagree, escalate to Max — he owns severity calls.

### "Claude is making stuff up about a file"

Stop, and ask Claude to **cite file:line**. Every claim about code should end with `path/to/file:lineNumber`. If Claude can't cite, the claim is invented — don't trust it. Open the file yourself (`cat path/to/file` or `code path/to/file` if you installed VS Code in WSL) and verify.

This is rare but important: if Claude pattern-matches an answer instead of reading code, you'll get plausible-but-wrong findings. The rule is **"cite or it doesn't count"**.

### "Bin/doctor shows a new FAIL I don't recognize"

```bash
./bin/doctor 2>&1 | grep FAIL
```

Send the output to Max. Don't try to fix harness issues yourself — those are engineering, not QA.

### "I broke my local config"

Reset:

```bash
rm state/local-config.json
cp etc/local-config.example.json state/local-config.json
# re-edit per step 2.3
```

### "I want to stop for the day mid-bug-hunt"

Just exit Claude (`/exit` or Ctrl+D). Your findings + tickets are saved to disk. Tomorrow:

```bash
cd ~/Developer/bims-corp-ops
claude
/bims-orient
/bims-ask remind me where I left off — which finding was I working on yesterday?
```

Claude reads the audit log and tells you.

### "Something REALLY weird happened"

Send Max:
- The exact command you ran
- The exact output (paste it; don't paraphrase)
- What you expected to happen
- What actually happened

Two-option rule: don't keep poking. Stop, ask Max, wait for direction.

---

## Part 7 — What you can and can't do (under current operating mode)

### Things you CAN do (always)

- Read any of the 8 legacy repos' source code (you have read-only GitHub access)
- Ask Claude any question about the code, architecture, or behavior
- Record findings via `/bims-ask` → `record this as a finding`
- Run `/bims-diagnose` on any finding
- Draft Jira tickets via `/bims-ticket`
- Promote tickets to Jira manually via `/bims-promote` (paste mode for now)
- Mark findings as `triaged` (Claude does this automatically when you `/bims-ticket`)
- Push your `tickets/draft/` work to a branch and PR it to Max

### Things you CANNOT do (under current advisory-only mode)

- **Edit any file under `BIMS-Corp-LEGACY/`** — this is enforced by hooks; Claude refuses. We're in advisory-only mode while AI capability is being assessed. Don't try to work around it.
- **Push, commit, or merge anything to the legacy repos** — same reason.
- **Promote tickets to PAS-*** in Jira — AI only writes to `BIMS-*` with `label=ai-drafted` (per trap BIMS-T-008). If you need a PAS-* ticket, that goes through TatvaSoft directly.
- **Echo secret values into prompts or tickets** — there are committed secrets in some repos (Max knows which); Claude knows the deny-list and won't paste contents into prompts. If you stumble on a secret, REFERENCE the file:line in the ticket, never PASTE the value.
- **Make business decisions** — if a bug-vs-feature judgment call is needed, escalate to Max.

### Things you SHOULD do (best practice)

- Start every session with `/bims-orient`
- Cite file:line in any QA communication that references code
- One finding per bug — don't bundle "5 things broken in the customer module" into one ticket; split them
- Re-read Claude's draft Jira ticket before promoting — it's an AI draft, not a finished product
- Ping Max if a finding feels >P2 and you're not sure

---

## Part 8 — Cheat sheet (print this and keep it next to your monitor)

| Command | What it does |
|---|---|
| `claude` | Open Claude Code (run from `~/Developer/bims-corp-ops`) |
| `/bims-orient` | Print current project state (open findings, recent decisions, traps) |
| `/bims-ask <question>` | Ask Claude anything about the 8 BIMS-Corp codebases — cited answer |
| `/bims-diagnose <finding-id>` | Run the 5-step diagnostic protocol on a finding (3 hypotheses, 2-track fix) |
| `/bims-ticket <finding-id>` | Draft a Jira ticket from a finding (writes to `tickets/draft/`) |
| `/bims-promote <finding-id>` | Promote a drafted ticket to Jira (paste-ready output until API wired) |
| `/help` | Claude's built-in help |
| `/exit` or Ctrl+D | Close Claude (your work is saved) |

| Where things live | Path |
|---|---|
| Workbench | `~/Developer/bims-corp-ops/` |
| The 8 legacy repos | `~/Developer/BIMS-Corp-LEGACY/` |
| Your findings DB | `~/Developer/bims-corp-ops/state/bims-store.db` (day 1; shared Turso later) |
| Drafted tickets | `~/Developer/bims-corp-ops/tickets/draft/*.md` |
| Promoted tickets | `~/Developer/bims-corp-ops/tickets/promoted/*.md` |
| Diagnoses | `~/Developer/bims-corp-ops/diagnoses/<finding-id>/diagnosis.json` |

| Repo nicknames | Full name |
|---|---|
| LSP-Web | LinkServicePro-Web (Laravel 8 PHP) |
| LIP-Web | LinkInspectPro-Web (CodeIgniter 2.1.4 PHP — legacy) |
| PO-Web | PropertyOrganizer-Web (CodeIgniter 2.1.4 PHP — legacy) |
| SA-Web | SuperAdmin-Web (CodeIgniter 2.1.4 PHP — legacy) |
| LSP-iOS / LIP-iOS | LinkServicePro / LinkInspectPro for iOS (Swift) |
| LSP-Android / LIP-Android | LinkServicePro / LinkInspectPro for Android (Kotlin) |

---

## Part 9 — What's coming (don't worry about yet)

These are on Max's roadmap; you don't need to act on any of them today. They're listed so you know what's possible later:

- **Shared Turso findings store** — your findings instantly visible to the whole team
- **Atlassian MCP wired** — `/bims-promote` posts to Jira automatically (no paste)
- **AI moves from advisory to limited workbench mode** — for now, only Max's machine does engineering; eventually the system will be capable enough that AI can also write fixes (under careful guardrails). You don't need to change anything about how you work when that happens.
- **Mobile QA flow with Playwright + iOS/Android simulators** — UI bug-hunting in mobile apps gets the same treatment as web bugs.

---

## Part 10 — Who to ping for what

| Question | Who | How |
|---|---|---|
| "Install broke / config broken / WSL crashed" | Max | Slack DM with screenshot + exact error text |
| "Claude says X but I disagree" | Max | Slack DM — paste the Claude output verbatim |
| "Is this finding really a P0 or P1?" | Max | Slack DM — paste the finding-id + your reasoning |
| "Can I file the ticket under PAS-* instead of BIMS-*?" | Max | Don't — but Slack him to discuss why you wanted to |
| "TatvaSoft pushed back on a ticket I drafted" | Max | Slack — he handles vendor-side disputes |
| "I want to do something the AI says it can't do" | Max | Slack — there's a reason it refuses; he'll explain or update the rules |
| "I found a real prod bug at 3am and Max is asleep" | Max + on-call channel | Record finding immediately, draft ticket, ping the on-call channel; Max picks it up in the morning |

---

## Appendix A — Example: the real bug from the demo

This bug is REAL and was discovered by Claude during the demo session that produced this guide. Use it as a reference for what a finished finding/diagnosis/ticket looks like:

- **Finding:** `BIMS-F-2026-05-27-ddca6b27` (P0, CRITICAL, LinkServicePro-Web)
- **Title:** `encrypt_decrypt()` helper has hardcoded AES key + static IV + no HMAC — forge-any-token across Stripe, invoices, customer flows, public forms
- **Diagnosis:** `diagnoses/BIMS-F-2026-05-27-ddca6b27/diagnosis.json`
- **Ticket draft:** `tickets/draft/BIMS-F-2026-05-27-ddca6b27.md`
- **Ticket promoted:** `tickets/promoted/BIMS-F-2026-05-27-ddca6b27.md`

In the meeting, Max will walk through this one end-to-end so you see what a complete finding looks like before you start hunting yourself.

---

## Appendix B — Glossary (5 minutes, read once)

- **Advisory-tier** — the current operating mode where AI finds bugs and drafts tickets but never modifies code. Your work mode.
- **Workbench-tier** — Max's mode; the AI can write/commit/push code under engineering guardrails. Not relevant to you.
- **Finding** — an entry in `bims-store.findings`. One row per bug. Has a stable ID like `BIMS-F-YYYY-MM-DD-XXXXXX`.
- **Diagnosis** — a structured analysis of a finding (3 hypotheses + 2 fix tracks). Lives at `diagnoses/<finding-id>/diagnosis.json`.
- **Implementation Note** — the section of a ticket that tells TatvaSoft EXACTLY what files to touch and what tests to write. Required for promotion.
- **Trap** — a known constraint about the codebase that the AI must respect (e.g. "CI 2.1.4 doesn't support PHP 8.1 idioms"). Surfaced by `/bims-orient`.
- **graphify** — the system that builds knowledge graphs of each codebase. Each repo has one MCP server (e.g. `mcp__graphify-bims-lsp-web`) that Claude uses to navigate.
- **MCP** — Model Context Protocol; how Claude talks to specialized servers (graphify, bims-store, atlassian, etc.). You don't interact with MCPs directly; Claude does.
- **rubric** — the deterministic algorithm that converts 4 facts about a finding into a severity label. Lives at `docs/severity-rubric.md`.
- **TatvaSoft** — the subcontractor team that writes the actual fixes in the legacy repos. Your tickets go to them via Jira.
