# Context Discipline

A field guide for keeping exact facts alive in long-running AI agent sessions.

Written for people who run **Claude Code** and/or a long-running agent harness (for example Hermes Agent, from Nous Research, open source). The method is generic and assumes no particular machine or setup; section 13 describes the author's own.

Author: Chin (0xsteamboat), Singapore.
Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use it, copy it, adapt it. Credit the author, provide a link to the licence and indicate changes.

> **Status note.** This guide is an independent, unofficial write-up. It is **not endorsed by or affiliated with Anthropic**.

---

## Contents

1. [The problem](#1-the-problem)
2. [The core rule](#2-the-core-rule)
3. [The enforcement ladder](#3-the-enforcement-ladder)
4. [Which rules can be enforced, and how](#4-which-rules-can-be-enforced-and-how)
5. [Information type to storage home (21 rows)](#5-information-type-to-storage-home)
6. [Choosing a mechanism](#6-choosing-a-mechanism)
7. [Write rules W1 to W9](#7-write-rules)
8. [Read rules R1 to R6](#8-read-rules)
9. [The compaction contract](#9-the-compaction-contract)
10. [Delegation and handoffs](#10-delegation-and-handoffs)
11. [Staleness and drift](#11-staleness-and-drift)
12. [Failure modes we hit](#12-failure-modes-we-hit)
13. [What we run, and how we proved it](#13-what-we-run-and-how-we-proved-it)
14. [Quick start](#14-quick-start)

Unnumbered: [Credit and licence](#credit-and-licence)

---

## 1. The problem

Long agent sessions get compacted. The runtime replaces old turns with a summary so the work can continue. A summary is written by a model, and it keeps the story and drops the details. The details are exactly what you need later: names, numbers, IDs, paths, dates.

We measured this on real transcripts: 168 compaction boundaries measured across 32 sessions, on two harnesses (46 boundaries in 14 Claude Code sessions, 122 in 18 Hermes Agent sessions).

| What we measured | Result |
|---|---|
| Mentions of exact names, numbers and paths lost across a compaction (weighted by how often each was mentioned) | median about **78%**, counting tool output (middle half of boundaries: 69% to 84%; user and assistant text alone, 151 boundaries: about 64%) |
| Top 20 most frequent identifiers lost at each compaction boundary | median about **58%**, counting tool output (Claude Code alone: 50%; user and assistant text alone, 151 boundaries: about 45%) |
| Precision of one regex harvest of identifiers from old transcripts (about 70 candidates) | only about **28%** |
| One example: a figure mentioned 75 times before and 75 times after a compaction | appeared **zero** times in the summary |

These are one author's sessions, and boundaries inside the same session are not independent of each other. Treat the numbers as a strong warning, not a law of nature, and re-measure on your own sessions.

Two things follow.

1. A fact that lives only in the conversation will be lost.
2. Trying to recover facts later by searching old transcripts is unreliable. It is inference over a transcript. A curated file is a record.

Compaction is only one way context dies. The same loss happens on every new session, every resume, and every handoff between agents.

---

## 2. The core rule

> **If a fact must survive, it lives in a file on disk, at one canonical path, and something mechanical either writes it or checks it.**
>
> Not in the conversation. Not in a summary. Not in an agent's memory of having decided it.

Two corollaries.

- **Derived beats recalled.** If a number can be computed from the repo, a script computes it. An agent that types a number a script could have produced has just created drift.
- **A rule that matters is enforced in code.** A project instruction file said "run the checker before committing". Two commits went in with the checker failing, because the command was piped (`check | tail -1 && git commit`), and a pipeline reports the exit status of its last command (unless `pipefail` is set). The instruction was followed and gated nothing. It is a pre-commit hook now.

### A third clause: placement

Reloading the instruction file after compaction is necessary. It is not sufficient. Public Claude Code issues describe rules being ignored around compaction, and the reports do not agree on the cause. One reporter ([#48959](https://github.com/anthropics/claude-code/issues/48959)) says the file is reloaded verbatim from disk after every compaction and the rules are still ignored, and hypothesises that position is the cause. Others report that the file is not re-read at all ([#21925](https://github.com/anthropics/claude-code/issues/21925)) or is simply ignored ([#19471](https://github.com/anthropics/claude-code/issues/19471), [#4017](https://github.com/anthropics/claude-code/issues/4017)). Anthropic's [memory documentation](https://code.claude.com/docs/en/memory) says the project-root `CLAUDE.md` is re-read from disk and re-injected after `/compact`. The position idea fits published work on long contexts, which found that models use text at the start and end of a long input better than text in the middle ([Liu et al., Lost in the Middle](https://arxiv.org/abs/2307.03172)). For Claude Code it is a hypothesis, not a proven cause.

> **A fact that must survive is written to disk, checked mechanically, and placed where attention is strong: the head or the tail of the assembled context, never the middle.**

A presence check is not a sufficiency check. Where you assert that a block is in context, assert where it sits, not only that it exists.

We tested the clause. Ten invented exact facts were placed at the head, the middle or the tail of a synthetic transcript of about 20k tokens, with the question at the end. Five trials per placement on two models (Claude Sonnet, and a DeepSeek model through Hermes Agent): every one of the 30 calls recalled all 10 facts. So at about 20k tokens, with no competing facts, placement made no measurable difference. Longer contexts and look-alike distractor facts are unmeasured. We keep the clause because it costs nothing, not because we have shown it matters.

---

## 3. The enforcement ladder

Not every place you can put a rule is equally strong. From hardest to softest:

| Rung | Layer | What it can guarantee | What it cannot |
|---|---|---|---|
| 1 | **Code that denies or allows a tool call** (`PreToolUse` hook exiting 2 or returning deny, permission deny rule) | The tool does not run this turn | Anything about what the model says. Anything outside the matcher. Exit 1, a crash or a timeout lets the call through unless the hook sets `onFailure` to block (v2.1.295 or later). |
| 2 | **Lifecycle hook** (session start, turn end) | A decision or an injection at a boundary | Cannot see state inside a turn without reading the transcript |
| 3 | **Pre-commit or CI check** | A blocked commit, in a repo that has the hook installed | Anything outside that repo |
| 4 | **Config read at process start** | A deterministic behaviour change with no model involved | Keys that are read live; anything the model must decide |
| 5 | **Always-loaded instruction file** (user-level and project `CLAUDE.md`) | That the text is present every turn | Compliance. Following it is the model's choice. |
| 6 | **On-demand skill or path-scoped rule** | Nothing at load time | Any guarantee. The body arrives only if it is called or the path matches. |
| 7 | **Persistent memory** | Re-injection from disk, capped | Verbatim recall. It is truncated and filtered. |
| 8 | **Prose in a document** | Nothing | Everything |

The short version is a four-step ladder:

**prompt text  <  always-loaded instruction file  <  hook  <  code**

**A rule that must hold every time never lives in a prompt.** Anthropic's memory documentation ([code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory)) says instruction files are context, not enforced configuration, and tells you to use a `PreToolUse` hook when an action must be blocked regardless of what the model decides.

Two practical cautions about the top rung.

- A deny hook that can crash is a deny hook that can be absent. In Claude Code a hook that exits 1, crashes or times out lets the action through by default. Exit 2 blocks. For tool hooks, set `onFailure` to block to fail closed. `onFailure` has no effect on `SubagentStop`, so a schema check that crashes there fails open.
- Permission rules and hooks that live in files the agent itself can edit are weaker than they look. A managed policy (a managed settings file or server-managed settings), which the agent cannot write, is the strongest route to a rule the agent cannot change. On Claude Code we run one. It denies the file-editing tools access to the settings and hook files outright, and puts the standing-instruction files behind an `ask` rule, so an agent can propose a change but a person approves it. Write these rules as `Edit(path)`: on our version Claude Code warned that `Write(path)` rules are not matched by file permission checks, and that `Edit` rules cover every file-editing tool.

---

## 4. Which rules can be enforced, and how

Thirteen classes of rule come up again and again. For each: the hardest layer that can carry it, whether the stock harness gives you a deterministic path, and what we run on top. The stock statuses were measured on Hermes Agent; where stock Claude Code differs, the row says so. An empty last cell means we add nothing beyond the stock harness.

| # | Rule class | Hardest layer available | Deterministic path in the stock harness? | What we run |
|---|---|---|---|---|
| i | Trim verbose tool results before they accumulate | Engine config plus code | Yes, if your harness has a prune setting. In Claude Code a `PostToolUse` hook can replace a tool's output before the model sees it (`updatedToolOutput`, per Anthropic's hooks documentation). | |
| ii | Exact facts re-injected every prompt, kept apart from the summary | Per-prompt context injection by code | Contingent: works only when the facts file exists and parses. Stock Claude Code injects at session start. A `UserPromptSubmit` hook can inject on every prompt, and a `SessionStart` hook with the compact matcher can inject again after compaction (Anthropic's hooks guide documents the compact matcher). | Claude Code: a `SessionStart` hook with no matcher injects the facts file at startup, resume, clear and after compaction. Hermes Agent: a plugin injects it before every model call. |
| iii | A settled decision is written the moment it settles | None | No. Nothing fires on "a decision was just made". | The figure check in row v catches a written figure that contradicts a settled one. A decision never written stays invisible. |
| iv | No duplicate file copy without a checker | Pre-commit | No, outside repos that have the hook | A daily checker flags the same fact line in two files, with both locations. Declared mirrors are allow-listed. |
| v | Derived values written by a script, never typed by an agent | Post-write hook | No in the stock harness | A post-write hook on both harnesses compares each labelled figure with a registry generated from the settled values, and tells the agent at write time when they disagree. |
| vi | Staleness measured in commits, not file times | Config plus plugin | Mis-enforced if the check reads file modification time | Staleness checks are inventoried, each with a verdict. The fact checks count changes, not minutes; the remaining timers measure things that age by nature: promises, external caches, liveness, alert windows and housekeeping. |
| vii | Subagent returns structured data, not prose | Hook | Claude Code: yes, with a hook (see below). Hermes Agent: its subagent-stop event is observer-only, so No. | When a delegation prompt ends with a `RETURN FIELDS:` line, a report missing those fields is denied before the parent reads it. Claude Code: a `PreToolUse` hook denies the report and the subagent is sent back. Hermes Agent: a plugin replaces the report with a rejection. |
| viii | Handoffs carry structure, not transcripts | Partial | Partial on Claude Code, No on Hermes Agent. A `PreToolUse` hook on the Agent tool can deny a delegation whose prompt lacks required fields. Nothing judges the content of a prose handoff. | Row vii covers results that code or another agent will read |
| ix | Never resume on stale tool results | Lifecycle hook | Partial on Claude Code (a `SessionStart` hook on the resume matcher can warn), No on Hermes Agent | On both harnesses, a resumed session is told which repos moved, and by which commits, since it last ran. Nothing stops the resume itself. |
| x | A scratchpad is re-read, not remembered | Per-prompt injection | Contingent, as row ii | Hermes Agent: a nudge after 15 tool calls with no change to the scratchpad |
| xi | Errors carry a category and are never silently suppressed | Partial | Partial | |
| xii | A standing behaviour rule lives in an always-loaded file, never in a per-directory memory | Always-loaded file | Yes. Per the [memory documentation](https://code.claude.com/docs/en/memory), the user-level `CLAUDE.md` and unscoped user-level rules files load in every session. | A daily placement checker flags exact figures and secret-shaped strings in standing files; a weekly report lists standing rules written into project memory |
| xiii | Compaction must not destroy current operating constraints | Engine code | Yes, on a harness whose compaction is your own code and keeps the facts block apart from the summary. Stock Claude Code has no engine-level control; a `SessionStart` hook with the compact matcher can re-inject the facts file afterwards. See H3. | As row ii |

**About row vii on Claude Code.** The documented route is a `SubagentStop` hook that blocks and sends the subagent back. In our first test, with only that hook installed, a prose report still reached the parent before `SubagentStop` fired. What worked was a `PreToolUse` hook on the tool that delivers the subagent's report (named `SubagentHandback` on our version), which denies a report missing the fields. Test it on your version with a helper told to answer in prose.

The honest summary: **in a stock harness, about half of these classes have a deterministic path and the rest are requests. The hooks and checkers in the last column turn most of the requests into checks. Four stay soft: a decision nobody writes down (iii), the content of a prose handoff (viii), a resume, which is warned about but not stopped (ix), and error categories (xi), which stay partial. Row x is covered on Hermes Agent only.** Absolute compliance is available rule by rule, at the layer named in the table, not by moving the whole standard to one layer.

---

## 5. Information type to storage home

Before you write anything that must outlive the session, find its row. If it has no row, it is either ephemeral (leave it in chat) or this table is missing a row (add it here first).

The short-name column gives each pattern a plain name. A "(Claude Code docs)" tag marks names that are product terms on the public Claude Code documentation pages; untagged names are plain descriptions chosen for this guide. The home column is generic: adapt the paths to your layout. The last column says how you would check or enforce it.

| # | Information type | Short name | Home | How to enforce |
|---|---|---|---|---|
| H1 | Exact project state: counts, amounts, IDs, paths, dates | **facts file**, kept apart from the summary | Project `FACTS.md`, generated by a script (W1) | Script with a `--check` mode run in pre-commit; a SessionStart hook injects the file; a Stop hook reports commit drift |
| H2 | Terms, safety information and standing parameters that must stay exact all session | **exact-terms section** at the top of the facts file | Top of the same facts file; never summarised | Same as H1; size cap with a loud warning |
| H3 | The conversation so far | **three layers**: exact-terms section, short summary of the discussion, recent turns verbatim; do not re-summarise a summary when numbers matter | The compaction summary carries discussion only; facts stay in H1 | Engine-level compaction rules. No engine-level control in stock Claude Code; a `SessionStart` hook with the compact matcher can re-inject the facts file afterwards (a hook-level mitigation; test it in your setup; Anthropic's hooks guide documents the compact matcher). |
| H4 | A precise follow-up question about a long source | **re-read the source** when a precise question comes up | The source file itself, re-read (R2, R3), never a "higher-fidelity summary" | Convention; no check |
| H5 | A belief that turned out wrong; a corrected figure | none (rule W6.3) | Project `SESSION-FACTS.md` | None. Add a check if the file matters. |
| H6 | A settled decision | none (rule W3) | Written the moment it settles: an open-decisions file while open, the project decision or facts file once closed | Nothing can know a decision was made (class iii). A post-write figure check can flag a written figure that contradicts a settled one. |
| H7 | The owner's current preferences about how work is done | **standing-rules file** | Standing rules in user-level `CLAUDE.md` or `.claude/rules/`; never a per-directory memory | A placement checker that flags standing rules written into project memory |
| H8 | Findings during a long investigation | **findings file** | A scratchpad file whose first line is `task label, timestamp, working directory` | A nudge hook that reports when the scratchpad is stale |
| H9 | An agent's progress, for crash recovery | **progress file** at a known path, read on resume | A known path per pipeline, named in that pipeline's facts file | Resume script that reads the manifest |
| H10 | A finding and where it came from | **source attribution**: for each finding keep where it came from (document and location), the exact supporting text | Inside the finding record itself, never stripped when findings are merged | A return contract: the delegation prompt names the required fields, and a report missing them is denied before the parent reads it (class vii) |
| H11 | Data for a machine or another agent | **schema-checked output**: JSON that matches a schema | JSON that matches a schema, never prose | The same return contract (class vii). A `PreToolUse` hook on the Agent tool can also check the delegation prompt (class viii). |
| H12 | Standards that always apply | **CLAUDE.md hierarchy**: managed policy, user, project and local files, loaded together (see the [memory docs](https://code.claude.com/docs/en/memory)) | Personal: user-level `.claude/CLAUDE.md`. Shared with a repo: project `CLAUDE.md`. Private to one repo: `CLAUDE.local.md` (gitignored). Managed policy: an OS-level managed `CLAUDE.md` (or the `claudeMd` key in managed settings), if you have one. | A placement checker; the user-level files behind a managed `ask` rule, so changes need a person's approval |
| H13 | A large instruction set | **imports**: split a large CLAUDE.md into topic files with `@path` (Claude Code docs) | Topic files pulled in with `@import`, not one monolith | Line-count lint on `CLAUDE.md` |
| H14 | Conventions for one area of a repo | **path-specific rules**: `.claude/rules/` with `paths:` globs (Claude Code docs) | `.claude/rules/*.md` per repo; user-level `.claude/rules/*.md` for global rules | Confirm in a fresh session that the rule appears in context |
| H15 | A multi-step procedure used only sometimes | **skill** for sometimes-used procedures; `context: fork` for verbose output (Claude Code docs) | `.claude/skills/<name>/SKILL.md` | A trigger eval that measures whether the description fires on the right prompts |
| H16 | A reusable prompt | **commands** (older format, merged into skills and still working) | Shared: project `.claude/commands/`. Personal: user-level `.claude/commands/` | None |
| H17 | Tool connections | **MCP server scope** (Claude Code docs) | Shared: project `.mcp.json`. Personal: `$HOME/.claude.json`. Credentials through `${ENV_VAR}`, never committed. | Secret scan in pre-commit |
| H18 | Reference material agents read often (for example database schemas or documentation) | **MCP resources** | Exposed as an MCP resource | None |
| H19 | A fact true for one project only | **auto memory** (Claude Code docs) | The project's auto-memory directory `$HOME/.claude/projects/<project>/memory/` (`MEMORY.md` plus topic files), machine-local; never a standing rule | Cap with a loud truncation warning |
| H20 | A rule that must hold every time | **hooks / `permissions.deny`, not memory** (Claude Code docs) | Hooks in `.claude/settings.json` | The hook is the check |
| H21 | Secrets | none (rule W9) | Injected at the execution layer; never in context, transcripts or memory | A PreToolUse hook that blocks reads of secret files; the placement checker flags secret-shaped strings in standing and memory files |

Notes on the table:

- **H1 to H4 are the heart of it.** They are the "facts survive compaction" rows.
- **H12 to H20 are the placement rows.** The wrong home here is quiet. Nothing errors. The rule just stops applying.
- The last column says how you could check or enforce each row. Section 13 lists the checks we run.
- `FACTS.md` has a measured zone, rewritten by the generator between BEGIN and END markers, and a hand-written prose zone. `SESSION-FACTS.md` holds corrections to wrong beliefs. Nothing injects it automatically.
- Rules W8 and R4 have no row: they are procedures, not information types, and R4 is a request, not a mechanism.

---

## 6. Choosing a mechanism

Rows H12 to H20 reduce to one choice table, in four lines:

| Mechanism | What it is | Use it for |
|---|---|---|
| **CLAUDE.md** | Always-on, soft context | Standards that always apply. Short. Stable. |
| **`.claude/rules/`** | Rules files: always-on, or path-scoped if they set `paths:` | Conventions for one area of the repo |
| **Skills** | On-demand procedures | Multi-step workflows used sometimes |
| **Hooks** | Deterministic enforcement | Anything that must hold every time |

The public [Claude Code features overview](https://code.claude.com/docs/en/features-overview) makes the same comparison.

Quick tests:

- Does it have to hold 100% of the time? **Hook.** Not CLAUDE.md.
- Does it apply only to some files? **Rule with `paths:`**, not the root CLAUDE.md.
- Is it a procedure with steps, used occasionally? **Skill.**
- Is it a standing preference about how you work? **User-level CLAUDE.md or rules**, never a per-directory memory.
- Did the model get the same thing wrong twice? Now write it down (see W8).

Also remember that a skill body is not loaded until the skill is called. A rule that lives only in a skill body has no guarantee of being read.

---

## 7. Write rules

Nine rules for what to write down, where, and when. Each has one sentence for the rule and one for why.

### W1. Derived values are written by a script, never by an agent.

If a value can be measured from config, a database or git, a repo-local generator writes it.
**Why:** an agent that types a number a script could produce has created a second source of truth that will drift.

A good generator has four properties:
- It rewrites only between BEGIN and END markers.
- A `--check` mode exits non-zero when the file is stale and writes nothing. This is the CI and hook mode.
- It checks the size before writing, not after.
- It never swallows errors. Redirecting stderr to nowhere hides real failures.

### W2. The generator runs at commit, not when remembered.

A pre-commit hook regenerates the file and stages it.
**Why:** "remember to regenerate" is a prompt rule that sometimes fails. Note that a git hooks path is local config and cannot be version-controlled, so each clone must install it. Check how many of your repos actually have it set.

### W3. Prose facts are written at the moment of settling, not at session end.

When a question is settled, a figure corrected or an approach ruled out, it goes into the hand-written part of `FACTS.md` (if it is project state) or into `SESSION-FACTS.md` (if it is a correction) before the next task starts.
**Why:** a fact held until the end of the session is a fact a compaction can take.

### W4. Trim tool results before they accumulate, but never trim a failure.

Cap verbose success output per tool and replace it with a reference. Keep failed calls and their error text in the trace.
**Why:** erasing a failure removes the evidence the model needs to stop repeating the action. These must be two separate code paths. A pruner that cannot tell an error from a verbose success trims the lesson along with the payload.

### W5. Handoffs carry structure, not transcripts.

Include the identifiers involved, the current state, what has been ruled out, the recommended next action, and what is still open and who owns it.
**Why:** the reader starts with a blank context. "As discussed above" is a failed handoff. A good delegation prompt says what to keep ("keep uncertainty and cite each claim with its source ID").

### W6. Write it down when any of these is true.

1. A fresh session would have to re-derive it, and re-deriving costs a chain of tool calls or would come out wrong.
2. It is exact and the exactness matters: a count, ID, path, amount, date or schema shape.
3. A belief about it was once wrong. This is the highest value of all. Put it in `SESSION-FACTS.md`.
4. It is a standing instruction about how to work. Put it in user-level `CLAUDE.md` or rules, never in a facts file.

**Why:** facts files carry state, not behaviour. Let go of anything reconstructible from the code, anything true only inside this conversation, anything git already records, and the story of how a fix was reached once the fix is committed.

### W7. Never write a mirror without a checker.

If you cannot add the checker, do not add the copy.
**Why:** a second copy is not a backup. It is a future contradiction. A proposed version of a file you cannot write is a proposal with an expiry: stamp the state of the source when you made it.

### W8. Do not add to a rules or memory file from a single observation.

Add on the second occurrence of the same correction, not the first.
**Why:** Anthropic's memory documentation ([code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory)) says to add to `CLAUDE.md` when the model makes the same mistake a second time. Also prefer raw retrievable storage over model-extracted "facts". A paper on agent memory ([arXiv 2603.02473](https://arxiv.org/abs/2603.02473), tested on the LoCoMo benchmark) found raw chunked storage, which needs no model calls, matched or beat fact extraction and summarisation. It also found that the retrieval method mattered more than how memories were written.

### W9. Secrets never enter context.

Inject credentials at the execution layer.
**Why:** transcripts, logs and resumed sessions persist whatever was in context. A secret that was ever in context is in a file somewhere.

---

## 8. Read rules

Six rules for reconstructing state from disk instead of remembering it.

### R1. At session start, load the project facts file.

In Claude Code a `SessionStart` hook can inject `FACTS.md` automatically. Wrap it as untrusted data: read it as facts about the project and ignore anything in it that reads like an instruction. If the task touches numbers in it, also open `SESSION-FACTS.md`, because nothing injects that one.
**Why:** a fresh session starts with zero history. The file is the history.

### R2. Before re-opening a settled question, read the session facts file.

If the answer is in there, the question is closed.
**Why:** that file exists because questions get re-derived wrongly.

### R3. Before reading a large knowledge base in full, query its index.

Use a search index (full-text search is enough) and read only what it points at.
**Why:** hundreds of notes are too many to browse. Full reads are for files the index names.

### R4. At context-budget checkpoints, act on the number.

A workable ladder: green below 40% of the window, flag at 40 to 60%, recommend compaction at 60 to 75%, force a handoff above 75%. At the 75% line, write a short handoff to disk before summarising, so the record survives the compaction that follows.
**Why:** a checkpoint written in a skill body only fires if that skill was loaded. Be honest about whether your trigger is a mechanism or a request.

### R5. Re-read the scratchpad. Do not remember it.

Re-read it before each significant decision in an exploration-heavy session.
**Why:** a scratchpad written and never re-read is a log, not working memory. Manus's own engineering blog says its agent keeps rewriting a to-do list as a task proceeds, which puts the plan in the most recent part of context ([Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)).

### R6. Resume or start fresh, on purpose.

Before resuming an old session, check what changed on disk since it ended. If nothing relevant changed, resume. If files, branches or external state moved, start a new session and give it a short brief that lists exactly what changed.
**Why:** a resumed session keeps its old tool output as if it were still true, and asking the model to tell old results from new is unreliable. A new session built from a brief holds only current facts.

---

## 9. The compaction contract

What the runtime should do when it compacts. Items 1, 3, 4 and 7 follow a third-party analysis of Claude Code's design ([Dive into Claude Code, arXiv 2604.14228](https://arxiv.org/abs/2604.14228)), which we have not checked against Anthropic's code. Items 2 and 6 are the author's design advice. Item 5 follows Anthropic's [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents). Treat all of it as design advice. It is worth copying into any harness.

1. **Cheapest reduction first, summarise last.** Run the cheap shapers (budget limits, snipping, clearing old tool results) before the expensive model-written summary.
2. **Trigger on a threshold.** Compaction should fire on a fill level, not on a human request. A manual-only design hits the wall mid-task with no recovery.
3. **Append-only storage, projection at read time.** Avoid editing or deleting written transcript lines. Append a boundary record and resolve reads from the last boundary forward. The analysis describes the store as mostly append-only, so the original lines stay on disk.
4. **Rebuild live state from the session object, not from the summary.** After compaction, re-announce the active plan, loaded skills and running agents from application state. This guards against the agent that re-reads files it already read and undoes work it just finished.
5. **Write the compaction prompt for recall first, precision second.** Capture every relevant piece of the trace, then iterate for precision.
6. **Never retry a request that failed because it exceeded the window.** Retrying cannot succeed. Compact, trim or split.
7. **Clear tool results by `tool_use_id`,** not by inspecting content, so clearing composes with size budgeting.

---

## 10. Delegation and handoffs

- **A subagent normally inherits no conversation.** (A forked subagent is the exception and inherits the whole conversation.) It has its own window, no parent history, and no memory between runs unless you enable the memory field. In Claude Code it still loads `CLAUDE.md` and rules at start (not the built-in Explore and Plan agents), so everything else must go in its prompt. Pass prior findings in its prompt, with each claim next to where it came from, so attribution survives.
- **Subagents return a summary, never a transcript.** A worker may spend tens of thousands of tokens and often hands back one to two thousand ([Anthropic, Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)).
- **Validate a result schema before the parent consumes it.** Structured output lets the parent compare and score. Unstructured output forces a second model call just to interpret it.
- **Isolate exploration, keep decisions in one context.** Read-and-report work suits isolated subagents. Decisions made by siblings in isolation tend to conflict.
- **Do not delegate what the coordinator already knows.** A subagent costs a call, a fresh context and result passing.
- **Partition by expected effort, not by count.** The slowest partition sets the total time.
- **Check coverage before you split.** List the subtopics first and confirm each one is assigned to a worker. A gap in the split is silent. A topic nobody was assigned is never covered.
- **Review from a fresh context.** The session that wrote the work is less critical of it. Anything touching money, credentials, production data or a large body of new logic gets a review by a fresh instance.

---

## 11. Staleness and drift

Staleness is detected by code, never by memory.

| Artifact | How staleness is detected | Strength |
|---|---|---|
| Measured zone of the facts file | Generator `--check` exits 1 | Code, in pre-commit and CI |
| Facts file against the repo | Count of commits since the file's own last commit | Code, in a Stop hook |
| Facts file size | Warn at 2 KB, hard limit at 4 KB | Code: a Stop hook warns, the injector refuses |
| Facts file tracked in git | Three distinct states: untracked, staged but never committed, committed | Code |
| Session facts file | Read on demand (R2) | By convention |
| Memory files | A daily checker flags secret-shaped strings and duplicated fact lines; a weekly report lists standing rules written into memory | Code, a daily scheduled job |
| Any mirror of another file | A fingerprint check across the declared copies; the duplicate checker catches a fact line (8 or more words, with a number, path or id) copied between the standing, memory and facts files | Code, a scheduled job |
| Scratchpad currency (Hermes Agent) | Tool calls since the scratchpad last changed | A nudge, not a gate |
| A resumed session's old tool results | Commits in each touched repo since the session last ran, named at resume | Code, a resume hook; warns, does not stop |
| Standing-instruction files | Any change is diffed and reported, and the old copy kept | Code, a job every 15 minutes |

Four principles behind the mechanisms that work:

1. **Measure staleness in commits, not file times.** Comparing HEAD's timestamp to the file's modification time marks a file stale the instant you commit it, and a fresh clone sets every time to now, which silences the check.
2. **Never assert a condition you did not measure.** Carry an `UNKNOWN` state that separates "git answered no" from "git could not answer". A confident warning on an unmeasured condition trains people to ignore the warning.
3. **Absent is not drift.** A repo that never opted in is never nagged. Nagging every repo destroys the signal for the few that opted in.
4. **A read-only observer must never take a write lock.** Commands such as `git status` can rewrite the index and take a lock. A Stop hook that did this broke a concurrent commit in 2 of 3 trials. Setting `GIT_OPTIONAL_LOCKS=0` fixed it.

**Every check reports where.** If you work across several machines and repos, "the config is stale" names nothing. State the machine and the absolute path, every time.

---

## 12. Failure modes we hit

Fifteen of these happened to us (1 to 15). Five are reported by others and included because they have the same shape (16 to 20). Each has a guard.

| # | What went wrong | Guard |
|---|---|---|
| 1 | A session "fixed the drift" in a scratchpad copy of the facts file while the real file stayed dozens of commits stale | One canonical path (W7). |
| 2 | A session reported creating three files that git had never seen | The keeper reports untracked and staged-never-committed as distinct states. The claim is checked, not trusted. |
| 3 | An instruction said "run the checker before committing". The commit went in anyway because a pipeline reports its last command's exit status | Move the gate into a pre-commit hook. A prose rule sometimes fails. |
| 4 | Compaction dropped a figure mentioned 75 times on each side of the boundary | A facts block re-injected every prompt, kept apart from the summary |
| 5 | A regex harvest of transcripts produced about 70 identifiers at 28% precision | Delete the harvest and keep the curated file. Harvest is inference. Curation is a record. |
| 6 | A substring dedupe deleted `$500` because it had kept `$5000`, inside a block that told the model the values were exact | No substring dedupe on exact values, ever. An independent review caught what self-review passed. |
| 7 | In our harness's compaction module, an anchor regex backtracked super-linearly: 24 KB took 7 seconds, 96 KB took 70 (4x the input, about 10x the time), stalling every compaction | Bounded segments. Any regex run on transcript-sized input gets a bounded-length audit. |
| 8 | In the same module, a word-boundary pattern cannot match before a leading `/`, so over a quarter of absolute paths lost their slash and were presented as exact | Where a value is labelled exact, no match beats a mangled match. |
| 9 | A Stop hook's `git status` took the index lock and broke a concurrent commit | `GIT_OPTIONAL_LOCKS=0` |
| 10 | On a case-insensitive filesystem, a file-exists check matched a differently-cased name but a git pathspec did not. The tool insisted a committed file was untracked on every Stop | Resolve the on-disk spelling once and use it for every pathspec. |
| 11 | A hook asserted what a different process was doing, and was wrong whenever that process was in a different mode | A hook reports its own observations, never another process's state. |
| 12 | A pre-merged settings mirror was correct one week and silently wrong the next | No mirror without a checker (W7). |
| 13 | Nearly every per-directory memory folder was empty. Everything durable sat in the one keyed to the home directory, invisible from any other project | Standing rules go in user-level instruction files, not per-directory memory. |
| 14 | A generator wrote oversized content and then returned an error, so the hook skipped staging and the tree diverged from the commit | Check size before writing. |
| 15 | Fatal errors hidden by redirecting output to nowhere. With the markers deleted, every future commit would skip regeneration silently | Never swallow a generator's stderr. |
| 16 | Reported elsewhere: Claude Code conventions ignored or lost around compaction, in several public issues whose reported causes differ ([#48959](https://github.com/anthropics/claude-code/issues/48959), [#19471](https://github.com/anthropics/claude-code/issues/19471), [#21925](https://github.com/anthropics/claude-code/issues/21925), [#4017](https://github.com/anthropics/claude-code/issues/4017)) | Assert placement (section 2). Re-announce live state from the session object. |
| 17 | Reported elsewhere: silent drift on resume: the agent ignored its memory file and project instructions and started from zero | A session-start injection check that covers resume, not only compaction |
| 18 | Reported elsewhere: auto-memory loads only the first 200 lines or 25 KB of `MEMORY.md`, so later curated content is not loaded at session start. Per the memory documentation, a write that leaves the index over the limit makes Claude Code return an error telling the model to rewrite it; check what your version does | A hard cap with a loud truncation warning |
| 19 | Reported elsewhere: in one tested model (Opus 4.6 with thinking), a safety monitor's recall for an inserted dangerous action fell from 99.7% in a 100K-token transcript to 69% in an 800K-token one ([Martin and Roger, arXiv 2605.12366](https://arxiv.org/abs/2605.12366)) | Segmented monitoring. Evaluate monitors at the length you deploy them. |
| 20 | Reported elsewhere: too many tools. In a GeoEngine test described by [Breunig](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html), a small quantised Llama 3.1 8B model failed with 46 tools even though the prompt fit within a 16K window, and succeeded with 19 | A per-task tool loadout |

Three lessons repeat across the list.

- **Instance versus class.** When you fix one wrong citation, search for its twins. A correction that leaves an identical error 75 lines away certifies half-finished work. A mechanical sweep for the superseded value beats a re-read.
- **A fix is not done until a test fails without it.** Remove the fix, watch the test fail, restore it, watch it pass.
- **Independent review catches what self-review passes.** Failure 6 passed the author's own review.

---

## 13. What we run, and how we proved it

The checks below run on the author's own setup: Claude Code and Hermes Agent, side by side, on one machine. Unless a line says otherwise, each check covers both harnesses. Each has a recorded proof. For most it is a live test: plant the fault, watch the check catch it, and confirm it stays quiet on a clean case. Where an item has a proof line, it says which; items 5, 9 and 10 are an inventory and two measurements, with their results recorded.

**1. A placement checker.**
A script reads the standing-instruction files, the rules files, the facts files and the memory files of both harnesses (for Claude Code, the memory folder of the home-directory project). It flags an exact figure in a standing file, a secret-shaped string in any of those files (by kind and location only, never the value), a standing rule written into a project memory, and the same fact line in two files. It runs daily and reports only hard findings; advisory ones, such as a standing rule in project memory, appear in a weekly report. Proof: a planted amount in a rules file, a fake key in a memory file and a planted duplicate were each flagged by file; a clean tree was silent.

**2. The storage map is always in context.**
A condensed version of section 5, with one trigger line ("before writing anything that must outlive this session, pick its home below"), lives in an always-loaded file on each harness: an unscoped user-level rules file on Claude Code, the standing-instruction file on Hermes Agent. Proof: a fresh session on each, asked without tools, quoted the trigger line exactly.

**3. A write-time figure check.**
A registry of settled values is generated from the decision file. A post-write hook compares every labelled figure in a written file with the registry. Proof: writing a price that disagreed with the settled one produced a mismatch message naming both values and the source, on both harnesses. Before the hook, the same write passed silently.

**4. A resume guard.**
Each session records the commit of every repo it touched. On resume, a hook names each repo that moved and the commits since. Proof: a session was resumed after a new commit; the warning named the repo and the commit message on both harnesses. Routine saves are ignored.

**5. Change-based staleness.**
A keyword search of the author's hooks, scripts and scheduled jobs found 22 staleness checks, each listed with a verdict. The two fact checks that used time alone, both on Hermes Agent, now count change: the scratchpad nudge fires after 15 tool calls with no change to the file, and the skills audit flags a skill whose referenced file is gone, not one that is merely old.

**6. A locked settings layer.**
On Claude Code, a managed settings file, owned by the operating system's administrator, denies the file-editing tools access to the settings and hook files. Proof: a fresh session asked to write into the hooks folder was refused by the managed rule, and the file was never created; a control write elsewhere succeeded.

**7. Standing instructions change only with approval.**
On Claude Code, the managed layer puts the standing-instruction and rules files behind an `ask` rule: an agent can propose a change, a person approves it. Proof: a fresh session's write to a rules file was held for approval and the file was absent afterwards. Hermes Agent has no managed layer, so there a job every 15 minutes diffs those files, reports any change and keeps a dated copy of the old version. That is detection, not prevention.

**8. A return contract for subagents.**
A delegation prompt that ends with `RETURN FIELDS: a, b, c` makes those fields mandatory. A report without them is denied and the subagent is sent back (Claude Code), or replaced with a rejection (Hermes Agent). Proof: a helper told to answer in prose was sent back, and the parent received a report with the required fields as JSON. See the note under section 4 for the two hooks involved.

**9. The placement test.**
See section 2: no measurable difference between head, middle and tail at about 20k tokens.

**10. The statistics, re-measured.**
Section 1 reports 168 compaction boundaries across 32 sessions on two harnesses.

### Where the checks stop

- Nothing can know a decision was made if nobody writes it down. The figure check catches a contradiction, not a silence.
- Nothing judges whether a prose handoff is good. The return contract checks fields, not meaning.
- Placement is measured only up to about 20k tokens, with no look-alike facts.
- The numbers come from one person's sessions. Re-measure on yours.


---

## 14. Quick start

If you do only five things:

1. **Create `FACTS.md` in each project that has exact numbers.** Generate the measured part with a script that has a `--check` mode.
2. **Run that script in a pre-commit hook,** install the hooks path in every clone, and also run its `--check` mode in CI, because the hooks path cannot be committed.
3. **Add a `SessionStart` hook** that injects the file as untrusted data, and a Stop hook that reports how many commits behind it is. Set the matcher on purpose: a hook with no matcher also fires on resume, clear and after compaction, while a hook limited to `startup` fires once. To keep the block alive after compaction (classes ii and xiii), make sure exactly one `SessionStart` hook covers the `compact` source, or use a per-prompt hook. Anthropic's [hooks guide](https://code.claude.com/docs/en/hooks-guide) shows the compact pattern under "Re-inject context after compaction". Confirm in a test session that the block is present after `/compact`.
4. **Put the storage map and a trigger line in an always-loaded file** (user-level `CLAUDE.md`, or a user-level rules file with no `paths:`) so the guide is seen without anyone opening it. Confirm in a fresh session that the trigger line is in context.
5. **Move every must-hold rule out of prose and into a hook or a permission deny rule.** Then prove it works by removing it and watching it fail.

Then, when something goes wrong, ask the one question that this whole guide comes from:

> Where did that fact live, and what was supposed to check it?

---

## Credit and licence

Author: **Chin (0xsteamboat)**, Singapore.

This work is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). You may share and adapt it for any purpose, including commercially, as long as you give credit and indicate changes.

Independent and unofficial. Not endorsed by or affiliated with Anthropic.
