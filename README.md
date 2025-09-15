# Precis: Hierarchical Repository Summarizer

## 1. Background Concepts

### Repository Hierarchies

Most Git repositories already contain a mix of:

* **Markdown files** (README.md, docs/)
* **Code files** (src/, lib/)
* **Image assets** (assets/, static/)
* **Other structured data** (CSV, JSON, YAML)

These form a **hierarchy of directories** that describe projects in natural layers. Git tracks file changes, but it does not provide human/LLM-readable summaries of the repository contents. That’s where **Precis** comes in.

### Obsidian Vaults

* **Vaults** are plain directories with Markdown notes.
* Typical contents include **personal journal entries**, **project notes**, **timelines**, and attachments.
* Subfolders and attachments mirror the filesystem.
* Precis treats any vault like a normal repository tree and can produce meaningful summaries for these personal knowledge structures.

### gitingest

* A tool that ingests entire Git repos into LLMs.
* Produces a flattened representation of files.
* Good for one-time ingestion but not hierarchical roll-up.

Precis combines these ideas into a **stream processing system** for continuous, hierarchical summarization.

---

## 2. Precis Overview

**Precis** is a tool that:

* Walks the repository tree.
* Creates structured summaries of each file.
* Summarizes directories based on their children.
* Recursively rolls up summaries until the root.
* Stores results in a `.precis` directory, alongside `.git`.

This makes a repo into a **self-explaining knowledge structure**, updated automatically whenever files change.

---

## 3. The `.precis` Directory Structure

Precis generates a parallel hierarchy inside `.precis/`:

```
my-repo/
  src/
    main.py
    utils.py
  docs/
    README.md
  assets/
    logo.png
  journal/
    2025-09-15.md

  .git/
  .precis/
    src/
      main.py.summary.md
      utils.py.summary.md
      __dir__.summary.md
      main.py.strategy.md
      main.py.review.md
    docs/
      README.md.summary.md
      __dir__.summary.md
      README.md.strategy.md
    assets/
      logo.png.summary.md
      __dir__.summary.md
    journal/
      2025-09-15.md.summary.md
      __dir__.summary.md
      2025-09-15.md.strategy.md
    __root__.summary.md
    index.md
    graph.md
    _meta/
      policies.yaml        # project-level defaults (limits, adapters, rubrics)
      adapters.yaml        # project-level adapter registry/overrides
      rubrics.yaml         # project-level evaluation weights/criteria
```

### Rules

* **Every file** has a summary in `.precis` (`<filename>.summary.md`).
* **Every directory** has a summary (`__dir__.summary.md`).
* The **root directory** has a top-level summary (`__root__.summary.md`).
* **Agentic artifacts** (per-file) are recorded alongside summaries:

  * `*.strategy.md` — chosen adapter + prompt reference + parameters (no content).
  * `*.review.md` — critique notes, rubric scores, and decision to accept / re-summarize.
* `index.md` collects all summaries; `graph.md` describes cross-links between files and folders.
* Global, reusable prompts/adapters live in the **Precis System Home** (Section 11). Project-level overrides live in `.precis/_meta`.

### File & Artifact Formats

All human- and LLM-facing artifacts are **Markdown with YAML front matter**. Example:

**`src/main.py.summary.md`**

```markdown
---
path: src/main.py
bytes: 1234
updated: 2025-09-15
source_hash: 8b3c…
children: []
---

# Summary
Implements the program entry point and HTTP server bootstrap.

- Functions: `init_app`, `run_server`
- Dependencies: `flask`, `sqlmodel`
- Key responsibilities: configuration load, route registration, server start
```

**`src/main.py.strategy.md`**

```markdown
---
adapter: code
prompt_id: adapter/code_summarize_v2
params:
  max_tokens: 160
  temperature: 0
signals: [".py extension", "import statements", "def "]
---

# Strategy
Use the **code adapter** to extract symbols (functions/classes), imports, and module role. Emphasize entry points and public API.
```

**`src/main.py.review.md`**

```markdown
---
rubric: code_v1
scores:
  coverage: 0.92
  correctness: 0.95
  concision: 0.88
decision: accept
iteration: 1
---

# Review Notes
- Captures all public functions and imports.
- Concision acceptable; no re-run required.
```

---

## 4. Agentic Summarization & Review

Precis uses an **agentic pipeline** to decide *how* to summarize each item and to self-check quality:

1. **First-look classification** (detector):

   * Heuristics + lightweight models read a small slice of content to infer type: `journal`, `project_note`, `timeline`, `code`, `markdown_doc`, `csv`, `image`, etc.
   * Signals include filename patterns (e.g., `YYYY-MM-DD.md`), front matter tags, headings ("Journal", "To‑Do"), code tokens, MIME, and folder context.
   * Decision recorded in `*.strategy.md` so future runs **skip re-deciding** unless the source or context changes.

2. **Adapter selection**:

   * A **type-specific adapter** is chosen from the **Precis System Home** registry (Section 11) with a base prompt template and extraction plan.
   * Project-level overrides can swap adapters via `.precis/_meta/adapters.yaml`.
   * Examples:

     * **Journal adapter**: extract date, high-level themes, key events, follow‑ups/tasks (redact PII if enabled).
     * **Project note adapter**: goals, decisions, owners, status, blockers, deadlines, next steps.
     * **Timeline adapter**: normalized chronology (date → event → impact), gaps/questions.
     * **Code adapter**: symbols, imports, responsibilities, entry points, integration points.
     * **CSV adapter**: columns, types, counts, missingness, intended use.

3. **Custom prompt synthesis**:

   * The adapter **references a general prompt** (no personal or content-specific details) and sets caps aligned to limits (Section 6).
   * Prompt reference stored in `*.strategy.md` as `prompt_id`.

4. **Summarize → Review loop**:

   * Initial summary produced with **temperature 0** (deterministic for given content/config).
   * **Critic pass** applies rubric from the registry (coverage, correctness, concision, safety/redaction).
   * If below threshold, generate **targeted feedback** and re-summarize (bounded iterations), storing notes in `*.review.md`.

5. **Acceptance & roll‑up**:

   * On acceptance, the file summary becomes input to its directory’s roll‑up.
   * Directory summaries are then re-evaluated if any child changed.

### Skipping repeated work

* The **strategy cache** (`*.strategy.md`) pins the chosen adapter and prompt reference so future runs don’t revisit classification/selection.
* Re-classification triggers: file moved across context boundaries, front‑matter tag changes, **adapter/prompt version bump** in the registry or project overrides.

### Privacy & redaction (Obsidian-friendly)

* Optional **PII/memoir redaction** for journals: names, emails, phone numbers hashed or replaced by stable pseudonyms.
* A safe-mode policy can suppress mood metadata or sensitive topics.

---

## 5. Roll-Up Process

### Stream Processing

* Precis runs incrementally, summarizing only changed files.
* Directory summaries are recomputed when any child summary changes.
* The process bubbles upward until the root summary is refreshed.

### Hierarchical Summaries

1. **File-level summaries** (bounded): adapter-specific abstracts + key points.
2. **Directory-level summaries** (bounded): thematic overview, key assets, risks, open questions, notable changes.
3. **Root summary** (bounded): global overview of components and purposes.

### Context Windows

* Summaries are **hard-capped** in size.
* **Default caps** (tunable): File ≤120 tokens; Directory ≤200 tokens; Root ≤500 tokens.
* Roll-ups are built only from summaries, never raw content, ensuring the root remains within normal context windows.

---

## 6. Default Parameters

Precis uses default parameters to manage consistency and resource efficiency:

* **Limits**

  * File summary limit: **120 tokens**
  * Directory summary limit: **200 tokens**
  * Root summary limit: **500 tokens**
  * Context reserve: **\~20%** of window left free for instructions
  * Review: **max 2 iterations**, acceptance threshold **≥0.85** rubric score
* **Execution / Models**

  * Default local model: **Qwen3‑4B‑Instruct‑2507 Q8**

    * Fallback to **Q4** if **<4 GB** system RAM
  * Local execution: **CPU or GPU selectable**
  * Remote execution (optional): **Ollama** or corporate API endpoint
* **Adapters**

  * Markdown/Docs, Code, Journal, Project Note, Timeline, CSV/TSV, JSON/YAML, Image, PDF
  * Redaction: **off by default**, configurable per adapter

---

## 7. Design Principles

* **Readable Markdown**: Summaries and agentic artifacts are Markdown with YAML front matter.
* **No random seed metadata**: Determinism comes from content + configuration; seeds are not stored in summaries.
* **Hierarchical roll-up**: bottom-up summarization guarantees scalability.
* **Compact summaries**: ensures roll-ups remain usable within standard context windows.
* **Mirrored structure**: `.precis/` mirrors the main repo, co-locating strategies and reviews for transparency.
* **Deterministic and auditable**: summaries reference source hashes and child summaries; reviews record rubric scores and decisions.

---

## 8. Example Workflow

* You write a journal entry `journal/2025-09-15.md`.
* Precis detects `journal` type → selects **Journal adapter** from the registry.
* Creates `journal/2025-09-15.md.summary.md` + `journal/2025-09-15.md.strategy.md`.
* Critic pass accepts; updates `journal/__dir__.summary.md` and then `.precis/__root__.summary.md`.

---

## 9. Benefits

* **Self-explaining repos & vaults**: Every file and folder has a readable, bounded summary.
* **Continuous documentation**: Docs stay in sync with changes.
* **Agentic quality**: Strategy selection, critique, and re-run ensure useful outputs.
* **Scalable**: Works for tiny vaults and massive repos.
* **Portable**: All artifacts are plain text in `.precis/`, versionable with Git.

---

## 10. Precis Mental Models: A public Precis repo which is used by the precis tool to decide how to summarize content

The shared public repository is **itself a Precis repository** that contains a curated set of **general, content‑agnostic mental models** for prompting and reviewing. It also includes things like AGENTS.md and other style and process information to help the planning agents decide how to prompt agents for summarization, review, etc.

### Goals

* **General, content‑agnostic**: captures *how* to think (metacognitive procedures), never *what* to say about specific content.
* **Edge‑capable**: navigable with a small local model on low compute (CPU or modest GPU), offline‑first.
* **Composable**: mental models can be chained (detect → plan → summarize → critique, revise, etc) and specialized by adapter.
* **Versioned & shareable**: distributed as a Git repo; local clones can contribute improvements upstream.

### Location (search order)

1. `$PRECIS_HOME` (if set)
2. `~/.precis/` (user scope)
3. `/etc/precis/` (system scope, read‑mostly)

Each location is a **Git repository** with its \*\*own \*\***`.precis/`** tree so it is self‑documenting and navigable by Precis.

### Layout (all Markdown with YAML front matter)

```
~/.precis/
  mental_models/
    detect/file_type.md          # how to classify files quickly
    plan/prompt_planner.md       # how to assemble prompt plans from models
    summarize/code.md            # adapter‑independent summarization mental model
    summarize/journal.md         # summarization for journals/notes
    summarize/csv.md             # summarization from schema/stats
    critic/generic.md            # generic rubric‑driven critique
    critic/code.md               # code‑aware critique
  prompts/                       # reusable prompt templates (general only)
    adapter/code_summarize_v2.md
    adapter/journal_summarize_v1.md
    adapter/csv_summarize_v1.md
    critic/generic_critic_v1.md
  adapters.yaml                  # maps detected types → (mental_models, prompt_ids)
  rubrics.yaml                   # named rubrics with weights/thresholds
  policies.yaml                  # global limits, context caps, safety/redaction
  CONTRIBUTING.md                # contribution process & safety rules
  OUTBOX/                        # queued PRs when offline
```

### Mental‑model format (example)

**`mental_models/summarize/journal.md`**

```markdown
---
id: mm/summarize/journal
version: 1
intent: produce bounded, privacy‑aware summaries of personal journal‑style text
applies_to: [journal, diary, daily_note]
inputs: [title?, date?, headings?, paragraph_slices]
outputs: [abstract, key_events, follow_ups]
compute_cost: low
safety:
  redact_pii: true
  forbid_content_leak: true
limits:
  file_tokens_max: 120
---

# Procedure
1) Prefer events, decisions, and follow‑ups over mood details.
2) Normalize dates; avoid quoting long passages.
3) If no clear events, summarize themes and questions.
4) Keep ≤120 tokens; use bullets sparingly (≤5).
```

> Mental models **never** contain personal examples or repository content; they only describe *procedures* and *constraints*.

### Precedence & versioning

1. Project overrides in `.precis/_meta/` (optional)
2. User‑custom models in `$PRECIS_HOME`
3. Upstream defaults in `$PRECIS_HOME` (pulled from public repo)

Each model/prompt has `id`, `version`, `intent`, `inputs`, `outputs`, and `limits`. Version bumps can trigger controlled re‑summarization.

### Impact on project summaries

* Project `.precis/` stores only **references** to mental models & prompts (IDs/versions) inside `*.strategy.md` plus rubric results in `*.review.md`.
* Example strategy metadata extension:

  ```yaml
  mental_models: ["mm/detect/file_type@1", "mm/summarize/journal@1"]
  prompt_plan: adapter/journal_summarize_v1@1
  router_evidence: ["YYYY‑MM‑DD filename", "frontmatter: journal: true"]
  cost_profile: { tokens: 150, passes: 1 }
  ```

---

## 11. Security & Privacy Guardrails

* Mental models and prompts in System Home remain **general**; no personal or repository content is stored there.
* Strategy and review files store **references** (IDs/versions) and **procedural evidence**, not raw excerpts.
* Optional redaction for journals/notes; configurable in `policies.yaml`.

---

---

## 12. Operation Modes & Integrations

Precis can run standalone (like `git`/`make`), as a Git-integrated hook, or via schedulers/watchers.

### 12.1 Standalone CLI (git/make‑style)

```
precis init                 # scaffold .precis/_meta with defaults and guidance
precis                      # incremental run in the current repo (alias: precis run .)
precis run [PATH]           # explicit path
precis status               # show pending/changed items and last run stats
precis verify               # recompute spot checks to ensure determinism
precis watch                # optional foreground watcher (fs events) that updates summaries
```

* **Incremental by default**: only changed files/dirs are reprocessed, with roll‑ups bubbling upward.
* **Resource guards** (set in `policies.yaml` or flags):

  * `--max-workers N`, `--cpu|--gpu`, `--window-tokens 131072`, `--max-tokens-per-call 1024`.
  * Model selection respects defaults in Section 6 (Qwen3‑4B‑Instruct‑2507 Q8; fallback Q4 if <4 GB RAM).

### 12.2 Git‑integrated (hooks)

Precis can integrate with Git hooks. In addition to post‑commit/post‑merge, you can run **before the commit** and optionally **prompt the user (y/N)** to update `.precis/`.

Install hooks:

```
precis hook install --events pre-commit,post-commit,post-merge,post-rewrite
# add --global to install into your global template
```

#### Interactive **pre‑commit** (prompt y/N)

**`.git/hooks/pre-commit`**

```bash
#!/usr/bin/env bash
set -euo pipefail

# Skip in CI or when bypassing hooks
if [ -n "${CI:-}" ]; then exit 0; fi

# Determine staged paths only (respect partial commits)
mapfile -t STAGED < <(git diff --cached --name-only --diff-filter=ACMRDTUXB)
[ "${#STAGED[@]}" -eq 0 ] && exit 0

# Decide whether to run (TTY prompt or env override)
RUN=no
if [ -t 0 ] && [ -t 1 ]; then
  read -r -p "Update .precis summaries before committing? [y/N] " ans || true
  [[ "$ans" =~ ^[Yy]$ ]] && RUN=yes
elif [ "${PRECIS_AUTO:-}" = "1" ]; then
  RUN=yes
fi

if [ "$RUN" = "yes" ]; then
  # Update summaries for exactly what is staged
  precis --changed "${STAGED[@]}" || {
    echo "precis failed; aborting commit" >&2
    exit 1
  }
  # Stage .precis updates into this same commit
  git add -A .precis
fi

exit 0
```

Notes:

* Uses **staged** paths so the summaries match *this* commit (respects partial staging).
* Prompts only if attached to a TTY; otherwise honors `PRECIS_AUTO=1` for auto‑run.
* Bypass anytime with `git commit --no-verify`.

#### Non‑interactive **post‑commit** (fire‑and‑forget)

```bash
#!/usr/bin/env bash
set -euo pipefail
if git rev-parse --verify HEAD~1 >/dev/null 2>&1; then
  mapfile -t files < <(git diff --name-only --diff-filter=ACMR HEAD~1 HEAD)
else
  mapfile -t files < <(git ls-files)
fi
[ "${#files[@]}" -gt 0 ] && precis --changed "${files[@]}" || true
```

#### Windows (PowerShell) pre‑commit prompt (minimal)

Create `.git/hooks/pre-commit` (no extension) and make it executable or ensure Git uses PowerShell hooks:

```powershell
$ErrorActionPreference = 'Stop'
if ($env:CI) { exit 0 }
$staged = git diff --cached --name-only --diff-filter=ACMRDTUXB | Where-Object { $_ }
if (-not $staged) { exit 0 }
$run = $false
if ($Host.UI.RawUI.KeyAvailable) {
  $ans = Read-Host 'Update .precis summaries before committing? [y/N]'
  if ($ans -match '^[Yy]$') { $run = $true }
} elseif ($env:PRECIS_AUTO -eq '1') { $run = $true }
if ($run) {
  & precis --changed $staged
  if ($LASTEXITCODE -ne 0) { Write-Error 'precis failed'; exit 1 }
  git add -A .precis
}
exit 0
```

#### Versioning `.precis/`

* **Share summaries**: commit `.precis/` (add `.precis/** linguist-generated=true` to `.gitattributes`).
* **Local only**: add `.precis/` to `.gitignore`.

### 12.3 Schedulers & File‑watchers

Run Precis on a cadence or on filesystem events.

**cron** (every 15 minutes):

```
*/15 * * * * cd /path/to/repo && /usr/local/bin/precis --quiet
```

**systemd (user) timer**
`~/.config/systemd/user/precis@.service`

```
[Unit]
Description=Precis incremental run in %I
After=network.target

[Service]
Type=oneshot
WorkingDirectory=%I
ExecStart=/usr/local/bin/precis --quiet
```

`~/.config/systemd/user/precis@.timer`

```
[Unit]
Description=Run Precis periodically in %I

[Timer]
OnBootSec=2m
OnUnitActiveSec=15m
AccuracySec=1m
Persistent=true

[Install]
WantedBy=timers.target
```

Enable:

```
systemctl --user enable --now precis@/path/to/repo.timer
```

**inotifywait** (Linux):

```
inotifywait -m -r -e close_write,create,delete,move --exclude '(^\.git/|^\.precis/)' . \
| while read -r _ path file; do
  [ -n "$file" ] && precis --changed "$path$file" || true
done
```

**fswatch** (macOS/\*nix):

```
fswatch -or --one-per-batch --exclude '\.git|\.precis' . | xargs -n1 -I{} precis --quiet
```

**watchman** (cross‑platform, optional):

```
watchman -- trigger /path/to/repo precis-trigger '**/*' -- precis --quiet
```

### 12.4 CI/CD & remote

* Run Precis in CI (e.g., GitHub Actions) to validate summaries build cleanly and to publish `.precis/` as an artifact or commit.
* For monorepos, run Precis per package directory in parallel.

### 12.5 Operational notes

* Precis always ignores `.git/` and `.precis/` as inputs.
* Large/binary files are adapter‑capped; see `policies.yaml` for byte/token limits.
* Backoff & jitter can be enabled when running persistent watchers to avoid flapping on mass changes.
