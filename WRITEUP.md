# Write-Up: Two-Agent AI Blog QA Pipeline

**Task:** Task 2 (Advanced) — Two-Agent AI QA for a Markdown Blog Post in GitHub
**Author:** Vaughn
**Date:** March 2026
**Time Spent:** ~3.5–4 hours across 4 sessions

---

## Table of Contents

1. [Why I Chose This Task](#1-why-i-chose-this-task)
2. [What the Pipeline Does](#2-what-the-pipeline-does)
3. [How It Works Step by Step](#3-how-it-works-step-by-step)
4. [Key Design Decisions](#4-key-design-decisions)
   - [Teaching the AI to Count Lines](#41-teaching-the-ai-to-count-lines)
   - [Choosing the Right AI Model for Each Job](#42-choosing-the-right-ai-model-for-each-job)
   - [Preventing Content Loss with Defence-in-Depth](#43-preventing-content-loss-with-defence-in-depth)
   - [Making Quality Scores Meaningful](#44-making-quality-scores-meaningful)
   - [Handling Broken JSON from AI Models](#45-handling-broken-json-from-ai-models)
   - [Building a Safe Patch Engine](#46-building-a-safe-patch-engine)
5. [How I Refined the AI Prompts](#5-how-i-refined-the-ai-prompts)
6. [What's Real vs What's Simplified](#6-whats-real-vs-whats-simplified)
7. [Error Handling](#7-error-handling)
8. [Known Limitations and What I'd Improve](#8-known-limitations-and-what-id-improve)
9. [Future Extensions](#9-future-extensions)
10. [Time Breakdown](#10-time-breakdown)
11. [Screenshots](#11-screenshots)

---

## 1. Why I Chose This Task

Out of the 10 tasks available, I chose Task 2 — the one explicitly marked as "Advanced" with a warning to only attempt it if you're comfortable with GitHub, diffs, and structured AI output.

I chose it for three reasons:

- **It's the most relevant to real marketing automation work.** Content teams need automated quality checks that don't overwrite their writers' work. This task solves exactly that problem — AI reviews a blog post, suggests specific fixes, and a human approves them through a Pull Request.

- **It touches the most real-world skills at once.** GitHub integration (reading files, creating branches, committing changes, opening PRs), structured AI output (getting two different models to produce reliable JSON), safe file editing (applying line-level changes without corrupting the document), and human-in-the-loop review (changes go through a PR, never directly to the live branch).

- **It forced me to solve hard problems that matter.** As I'll explain below, AI models can't reliably count lines in a document. They sometimes return broken JSON. They occasionally delete content when they're supposed to be fixing it. These aren't edge cases — they're the core challenges of building reliable AI-powered editing systems. Solving them taught me things I'll use on every future project.

---

## 2. What the Pipeline Does

In plain English: the pipeline reads a blog post from GitHub, has one AI agent review it for quality issues, has a second AI agent write specific fix instructions, applies those fixes safely to the original document, and then opens a Pull Request so a human can review and approve the changes before they go live.

Nothing gets published without human approval. The AI suggests — the human decides.

### Components

| Component | What It Does |
|---|---|
| **Main Workflow** | Orchestrates the entire pipeline from start to finish |
| **Agent 1 — QA Critic** | Reads the blog post and produces a structured list of quality issues (grammar, tone, clarity, readability, structure) with a scored breakdown |
| **Agent 2 — Editor** | Takes the list of issues and converts each one into a specific edit instruction (which line to change, what the original text says, what it should say instead) |
| **Patch Engine** | A JavaScript code node that applies the edit instructions to the original document, line by line, with safety checks |
| **Error Handler** | A separate workflow that catches any failures and logs them to a data table for debugging |

### Architecture

```
Manual Trigger (repo, file path)
       │
       ▼
Fetch blog post from GitHub
       │
       ▼
Add line numbers ([L1], [L2], etc.)
       │
       ▼
┌─────────────────────────┐
│  Agent 1 — QA Critic    │
│  (Claude Sonnet)        │
│  Finds quality issues   │
└────────────┬────────────┘
             │
       Any issues found?
       ├── No → Stop (article is clean)
       │
       ▼ Yes
┌─────────────────────────┐
│  Agent 2 — Editor       │
│  (GPT-4o)               │
│  Creates edit instructions│
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Patch Engine           │
│  Applies edits safely   │
│  (reverse sort, conflict│
│   detection, fallbacks) │
└────────────┬────────────┘
             │
             ▼
Generate QA Report (Markdown)
             │
             ▼
Create GitHub branch
  → Commit patched blog post
  → Commit QA report
  → Open Pull Request
             │
             ▼
Log successful run to data table
```

---

## 3. How It Works Step by Step

Here's what happens when you click "Run" on the main workflow:

1. **Set Input Variables** — The workflow starts with the GitHub repository details: repo owner (`Vaughnai2023`), repo name (`longship-n8n-blog-qa`), and the file path to the blog post (`sample-content/draft-content-strategy.md`).

2. **Fetch the Blog Post** — An n8n GitHub node reads the file from the repository. GitHub returns the file content as Base64-encoded text (a way computers store file data), so the next step decodes it back into readable Markdown.

3. **Add Line Numbers** — Before sending the content to the AI, every line gets a marker like `[L1]`, `[L2]`, `[L3]` prepended to it. This is critical — I'll explain why in the [design decisions section](#41-teaching-the-ai-to-count-lines).

4. **Agent 1 Analyses the Content** — The numbered blog post is sent to Claude Sonnet (via OpenRouter) with a detailed prompt that includes brand voice guidelines. Agent 1 reads every line and returns a structured JSON object containing:
   - A summary of the article's overall quality
   - A quality score with a breakdown across 5 dimensions (grammar, brand voice, clarity, structure, completeness)
   - A list of specific issues, each with a line number, issue type, severity, description, and suggested fix

5. **Check If Issues Were Found** — An IF node checks whether Agent 1 found any issues. If the article is clean (no issues), the workflow stops here — no unnecessary edits or PRs.

6. **Agent 2 Creates Edit Instructions** — If there are issues, the original blog post content and the list of issues are sent to GPT-4o (via OpenRouter). Agent 2 converts each issue into a concrete edit instruction: which line to change, what the original text says, and what it should say instead. Some issues get intentionally skipped if they're too complex for a line-level edit (for example, restructuring entire sections).

7. **Patch Engine Applies Edits** — A JavaScript Code node takes the edit instructions and applies them to the original document. It processes edits from bottom to top (so earlier line numbers don't shift when later lines change). For each edit, it verifies the original text actually matches before making the change. If something doesn't match, it searches the whole document for the text as a fallback. If more than 50% of lines would be changed, it aborts entirely — the article probably needs a full rewrite, not patches.

8. **Generate QA Report** — A Code node produces a Markdown report containing the quality score breakdown, all issues found, and which edits were applied vs skipped. This report gets committed alongside the patched blog post.

9. **Create PR** — The workflow creates a new GitHub branch (named `ai-qa/{filename}-{date}`), commits the patched blog post and QA report to it, and opens a Pull Request against the main branch. The PR title and body include the quality score and a summary of changes.

10. **Log the Run** — Finally, the workflow logs the run details (timestamp, PR URL, quality score, number of edits applied/skipped) to an n8n Data Table for tracking.

---

## 4. Key Design Decisions

These are the most interesting problems I solved during the build. Each one taught me something about working with AI models in automation pipelines.

### 4.1 Teaching the AI to Count Lines

**The problem:** The QA pipeline needs accurate line numbers. Agent 1 says "there's a grammar error on line 27," and the patch engine needs to find line 27 and fix it. If the line number is wrong, the edit either goes to the wrong place or gets skipped entirely.

But here's the thing: **AI language models cannot reliably count lines in a document.** They don't process text the way we do — they break it into tokens (small chunks of text), and those tokens don't map neatly to lines. When I first tested Agent 1, it got **0 out of 5 line numbers correct.** Some were off by 4–6 lines.

This isn't a bug in any specific model. It's a well-known limitation across all large language models. Production AI code review tools like Ellipsis and CodeRabbit have the same challenge — and they solve it by using text-based matching instead of trusting line numbers.

**What I tried first:** Better prompt instructions. I told the model to count carefully, to count blank lines, to not split wrapped lines. After 4 rounds of prompt tweaking, accuracy improved slightly but was still unreliable.

**What actually worked:** Instead of asking the AI to count, I gave it the answers. Before sending the blog post to Agent 1, I prepend a marker to every line:

```
[L1] # Why Most B2B Blog Posts Never Get Read
[L2]
[L3] Here's a stat that should make every content marketer uncomfortable...
```

Now the AI doesn't need to count — it just reads the `[L#]` marker next to each line. The prompt instructs it to always use the number from the marker, never to count on its own.

**Result:** Line number accuracy went from **0 out of 5 correct to 5 out of 5 correct.** A simple fix for a fundamental limitation.

---

### 4.2 Choosing the Right AI Model for Each Job

Not all AI models are equally good at every task. I used two different models for the two agents because they have different strengths:

- **Agent 1 (QA Critic) — Claude Sonnet:** Editorial judgment requires nuance. The model needs to understand tone, brand voice, readability, and structural flow. Claude Sonnet is strong at this kind of qualitative analysis. I set its temperature to 0.2 (a setting that controls how creative vs predictable the output is — lower means more consistent).

- **Agent 2 (Editor) — GPT-4o:** Generating edit instructions is a more mechanical task. The model needs to produce precise JSON with exact text matches. GPT-4o is reliable at structured output generation. Temperature set to 0.1 for maximum precision.

**Why not use one model for both?** I tried. I also tried GPT-4o Mini (a faster, cheaper version) for Agent 2. It produced:
- Duplicate edits targeting the same line
- Partial text fragments instead of complete line content
- Conflicting edits that would overlap and corrupt the document
- Erroneous punctuation changes

GPT-4o solved all of these problems. The cost difference between Mini and the full model is minimal for this use case — a few cents per run — so the trade-off was clear: reliability over cost savings.

---

### 4.3 Preventing Content Loss with Defence-in-Depth

**The problem:** When Agent 2 creates an edit instruction, it includes the `original_text` (what the line currently says) and the `new_text` (what it should say instead). The patch engine then swaps one for the other.

But some lines in a blog post are entire paragraphs — multiple sentences that wrap across what looks like many lines but is actually a single line in the Markdown file. When Agent 2 fixed a grammar error in sentence 3 of a paragraph, it sometimes returned only sentence 3 as the `new_text`, not the full paragraph. The patch engine would then replace the entire paragraph with just that one sentence, deleting sentences 1 and 2.

**How I fixed it (two layers):**

**Layer 1 — Prompt engineering:** I added an explicit rule to Agent 2's prompt (Rule 8): "Your `original_text` and `new_text` must contain the COMPLETE line content, never just a snippet. If the line is a full paragraph, include the entire paragraph." I also updated the JSON example in the prompt to show paragraph-length content so the model could see what was expected.

**Layer 2 — Code-level safety net:** Even with the prompt rule, AI models don't follow instructions 100% of the time. So I added a second layer of protection in the patch engine itself. When applying an edit, it checks: does the `original_text` match the entire line, or just part of it?

- If it matches the full line → replace the whole line (normal behavior)
- If it matches only a portion of the line → do a targeted find-and-replace *within* the line, leaving the rest of the content intact

This is called **defence-in-depth**: the prompt tries to prevent the problem, and the code handles it safely if the problem still occurs. Each edit logs whether it used `full_line` or `substring` replacement mode for transparency.

**Result:** No content loss on any line, even when the AI returns partial text.

---

### 4.4 Making Quality Scores Meaningful

**The problem:** Agent 1 assigns a quality score to the blog post. Originally, this was a single number from 0 to 100 with vague bands ("90-100: Publication-ready," "70-89: Good but needs some fixes"). There was no defined criteria for what drives the score — a 70 and a 75 were indistinguishable. The agent was essentially guessing, and the scores weren't reproducible or comparable across different articles.

**The fix — a weighted scoring rubric:** I replaced the single score with 5 independently scored dimensions:

| Dimension | Weight | What It Measures |
|---|---|---|
| Grammar & Mechanics | 25% | Spelling, punctuation, grammar — objective right/wrong |
| Brand Voice | 25% | Does the tone match the brand guidelines? |
| Clarity & Readability | 20% | Are sentences clear? Is jargon avoided? Is it scannable? |
| Structure & Flow | 15% | Logical ordering, transitions between sections, heading hierarchy |
| Completeness & Depth | 15% | Are key points covered? Are claims supported with evidence? |

Each dimension has its own scoring bands (defined in the prompt), and Agent 1 must provide a written rationale for each score, citing specific content from the article. The final quality score is a weighted sum of all five dimensions.

**An important detail:** The report node recalculates the weighted score from the individual dimension scores rather than trusting the AI's arithmetic. AI models are good at judgment but unreliable at math — so I let the model do what it's good at (evaluating quality) and let the code do what it's good at (multiplication and addition).

**Result:** The latest run produced a score of 75/100 with a clear breakdown:
- Grammar & Mechanics: 55 (3 hard errors found)
- Brand Voice: 80 (one overly casual phrase)
- Clarity & Readability: 82 (one vague sentence)
- Structure & Flow: 85 (smooth flow, one minor ordering note)
- Completeness & Depth: 78 (one central claim lacked supporting evidence)

A reviewer can now see exactly *why* the article scored 75 and which areas need the most attention.

---

### 4.5 Handling Broken JSON from AI Models

**The problem:** Both agents return their output as JSON (structured data). The pipeline needs to parse this JSON to extract the list of issues and edit instructions. But AI models don't always produce perfectly valid JSON.

Specifically, Claude Sonnet occasionally outputs square brackets `[` and `]` where curly braces `{` and `}` should be. For example:

```json
// What the model returned (invalid):
["line_number": 11, "issue_type": "grammar", ...]

// What it should have been (valid):
{"line_number": 11, "issue_type": "grammar", ...}
```

This is syntactically invalid JSON and crashes a standard parser.

**The fix:** I built a 7-step defensive JSON parser that cleans up common AI output mistakes before parsing:

1. Strip code fences (` ```json ` wrappers the model sometimes adds)
2. Remove trailing commas (a common mistake in generated JSON)
3. Fix bracket/brace confusion (three regex passes that convert `["key":` to `{"key":`, and the corresponding closing brackets)
4. Parse the cleaned JSON
5. Validate the expected structure exists
6. Extract the data
7. Provide clear error messages if parsing still fails

This parser has handled every output variation I've encountered across dozens of test runs. It's a reminder that when working with AI output, you need to build for what the model *actually* produces, not just what you *asked* it to produce.

---

### 4.6 Building a Safe Patch Engine

The patch engine is the part of the pipeline that actually modifies the blog post. It takes the edit instructions from Agent 2 and applies them to the original document. Getting this right is critical — a bug here could corrupt the content.

Here are the safety measures built into the engine:

- **Reverse sort:** Edits are applied from the bottom of the document upward. This prevents a common problem: if you insert or delete a line near the top, all the line numbers below it shift. By working bottom-up, earlier edits don't affect the line numbers of later edits.

- **Text-based matching as the primary anchor:** The `original_text` field (what the line currently says) is the source of truth — not the line number. The engine first checks if the text matches at the given line number. If it doesn't (because the AI's line number was slightly off), it searches the entire document for the text. This makes the system resilient to small line-number errors.

- **Conflict detection:** If the `original_text` doesn't match anywhere in the document, the edit is skipped and logged. This prevents applying stale or incorrect edits.

- **50% abort rule:** If more than half the lines in the document would be modified, the engine aborts entirely. The article probably needs a full rewrite, not line-level patches. This prevents the AI from silently rewriting the whole piece.

- **Empty-line guard:** A subtle JavaScript bug: the expression `"any string".includes("")` always returns `true`. Without a guard, the engine would "match" blank lines against any edit instruction. I added an explicit length check to prevent this.

- **Detailed logging:** Every applied edit records which line it was actually applied to, whether it matched by line number or by text search, and whether it used full-line or substring replacement. This makes debugging straightforward.

---

## 5. How I Refined the AI Prompts

AI prompts rarely work perfectly on the first try. Here's the iteration process I went through:

### Agent 1 — QA Critic (4 iterations)

**Iteration 1:** The initial prompt used `"type": "system"` for the system message. n8n's AI Agent node (version 1.9) requires the full name `"type": "SystemMessagePromptTemplate"`. Fixed.

**Iteration 2:** The output parser had "Auto-Fix" enabled, which requires a separate AI model node to be connected. Since Claude returns valid JSON reliably, I turned this off and relied on the structured output parser plus my custom JSON cleanup code instead.

**Iteration 3:** The `hasOutputParser` flag wasn't enabled. This meant the output parser node was connected but not actually being used — the agent returned raw text instead of parsed JSON. Enabling this flag fixed the issue.

**Iteration 4:** Line number accuracy and quality score scale. Added the `[L#]` marker injection approach (described above). Also strengthened the quality score instructions — the model was stubbornly returning scores on a 0-10 scale despite being told to use 0-100. Added explicit examples of correct vs incorrect scoring. As a safety net, I also added normalization code in the report node to multiply by 10 if the score comes back in the 0-10 range.

### Agent 2 — Editor (2 key changes)

**Change 1 — Model swap:** Replaced GPT-4o Mini with GPT-4o after Mini produced unreliable output (duplicate edits, partial text, overlapping changes).

**Change 2 — Rule 8:** Added an explicit prompt rule requiring `original_text` and `new_text` to contain the complete line content, never just a snippet. Updated the JSON example to show paragraph-length content. Combined with the substring replacement safety net in the patch engine, this solved the content-loss problem entirely.

---

## 6. What's Real vs What's Simplified

Longship's brief says to clearly indicate which parts are real integrations and which are mocked. Here's the breakdown:

| Component | Status | Details |
|---|---|---|
| GitHub file reading | **Real** | Reads from a real GitHub repository via API |
| AI analysis (Agent 1) | **Real** | Real Claude Sonnet calls via OpenRouter |
| AI editing (Agent 2) | **Real** | Real GPT-4o calls via OpenRouter |
| Patch engine | **Real** | JavaScript Code node with full safety logic |
| QA report generation | **Real** | Markdown report committed to the PR |
| Branch creation | **Real** | Creates real branches in the GitHub repo |
| File commits | **Real** | Commits patched file and report to the branch |
| Pull Request | **Real** | Opens a real PR with title, summary, and score |
| Error logging | **Real** | Logs to n8n Data Tables (qa_run_log, qa_error_log) |
| Brand voice context | **Simplified** | Hardcoded in Agent 1's prompt instead of fetched dynamically |

**Why is brand voice hardcoded?** In a production system, brand voice guidelines, messaging pillars, and tone rules would be stored in a database (like Airtable) and fetched dynamically by brand ID. For this prototype, I hardcoded a realistic brand voice profile (professional, approachable, concise, confident, human) directly in the prompt. This keeps the scope focused while still demonstrating that the agent can evaluate content against specific brand guidelines.

**A note on the GitHub integration:** The workflow uses a mix of native n8n GitHub nodes (for reading files) and HTTP Request nodes (for creating branches, committing files, and opening PRs). This isn't inconsistency — it's because the native GitHub node doesn't support all GitHub API operations (like creating Git refs or using the contents API's PUT endpoint). The HTTP Request nodes fill the gaps.

---

## 7. Error Handling

The pipeline has three layers of error handling:

### Layer 1: Workflow-Level Error Handler
A dedicated error handler workflow catches any unhandled failures in the main pipeline. When an error occurs:
1. The Error Trigger node fires
2. A Code node extracts the error details (message, workflow name, node that failed, execution ID)
3. The error is logged to the `qa_error_log` Data Table with a timestamp

This means even if the pipeline crashes, we have a record of what happened and where.

### Layer 2: Patch Engine Safety Rules
- **Out-of-range line numbers** → Edit is skipped and logged
- **Text doesn't match** → Edit is skipped and logged (conflict detection)
- **More than 50% of lines modified** → Entire patching operation aborted, original file returned unchanged
- **Any unexpected error** → Original file returned unchanged, failure logged

### Layer 3: Flow Control
- **No issues found** → An IF node skips Agent 2 entirely. No unnecessary AI calls, no empty PRs, no wasted time or money.

---

## 8. Known Limitations and What I'd Improve

Being honest about what's not perfect:

### Idempotency
If the workflow runs twice on the same file on the same day, the second run will fail because the branch name `ai-qa/{filename}-{date}` already exists. **Fix:** Add a short timestamp or random hash to the branch name to guarantee uniqueness (e.g., `ai-qa/draft-content-strategy-2026-03-12-1430`).

### No Retry on Agent 2 Failure
If Agent 2 returns invalid JSON that even the defensive parser can't fix, the workflow errors out. The structured output parser handles most cases, and the error handler logs the failure, but there's no automatic retry. **Fix:** Add a retry loop (try once, and if it fails, try again with a slightly modified prompt before giving up).

### Brand Voice Is Static
The brand voice guidelines are hardcoded in Agent 1's prompt. **Fix:** Store brand voice profiles in Airtable or a database, and have the workflow fetch them dynamically based on a `brand_id` parameter. This would let the same pipeline serve multiple brands without prompt changes.

### Branch Name Date Granularity
The branch name only includes the date, not the time. Multiple runs on the same day collide. **Fix:** Include hours and minutes in the branch name, or use a sequential counter.

Longship's brief says they don't expect full production hardening. These are all things I'm aware of and would address with more time.

---

## 9. Future Extensions

Things I would build next if this were a production system:

- **Webhook trigger:** Replace the manual trigger with a webhook so the workflow can be called from a CMS, Slack bot, or content scheduler. A writer could push a draft and the pipeline runs automatically.

- **Dynamic brand context:** Fetch brand voice guidelines, messaging pillars, and persona details from an Airtable table by `brand_id`. This makes the pipeline reusable across multiple brands without changing the prompt.

- **Multi-file batch processing:** Loop through all `.md` files in a directory for bulk QA. Useful for teams that want to audit their entire blog in one pass.

- **Upstream automation:** Auto-trigger when a writer opens a PR or marks an article as "Ready for QA" in a CMS. This removes the manual step entirely.

- **QA log aggregation and scoring dashboards:** Store QA results in a database to build a content quality scorecard over time. Track whether average quality improves, which types of issues are most common, and which writers might benefit from specific feedback.

- **Downstream distribution:** On PR merge, trigger social post generation, email newsletter inclusion, or Slack notifications to the content team.

---

## 10. Time Breakdown

| Phase | Time |
|---|---|
| Setup (GitHub repo, n8n environment, credentials, sample blog post) | ~30 min |
| Agent 1 — QA Critic (sub-workflow + prompt design + 4 iterations) | ~40 min |
| Agent 2 — Editor (sub-workflow + prompt design + model testing) | ~30 min |
| Patch Engine (Code node + safety logic + fallback search) | ~30 min |
| Main Workflow wiring + GitHub integration | ~30 min |
| Error Handler | ~15 min |
| End-to-end testing + debugging + bug fixes | ~45 min |
| Weighted scoring rubric (Agent 1 prompt + report node) | ~20 min |
| **Total** | **~3.5–4 hours** |

---

## 11. Screenshots

All screenshots are in the `assets/` directory:

| Screenshot | What It Shows |
|---|---|
| [main-workflow.png](assets/main-workflow.png) | The full main workflow canvas — fetch → analyse → edit → patch → PR |
| [agent1-qa-critic.png](assets/agent1-qa-critic.png) | Agent 1 sub-workflow — Claude Sonnet analysing content quality |
| [agent2-editor.png](assets/agent2-editor.png) | Agent 2 sub-workflow — GPT-4o generating edit instructions |
| [error-handler.png](assets/error-handler.png) | Error handler workflow — catches failures and logs to data table |
| [example-pr.png](assets/example-pr.png) | A real Pull Request created by the pipeline with quality score and edit summary |
| [error-log.png](assets/error-log.png) | The error log data table showing a captured failure |
