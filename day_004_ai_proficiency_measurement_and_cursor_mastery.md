# Day 004: Measuring How I Work With Agents — Plugin, Cursor Audit, and Git Foundations
**Date:** 2026-10-05 | **Category:** #Proficiency #Cursor #Agents #Git #Measurement | **Environment:** Apple M1 Pro macOS, Cursor 3.x, Claude Code plugin cache, Node 18+

## Day Map

1. Installed the **AI Proficiency Insights** plugin from the Intuit DevAssist registry. It was invisible to Cursor, which turned out to be a plugin-architecture fact worth knowing. Ran its extractor over 30 days of my own session logs and got a deliberately incomplete reading.
2. That reading says nothing about *which Cursor features* I use, so the second thread asked that directly: is there a published way to rank how fully someone uses Cursor? That led to an audit of my own config and to where real adoption data actually lives.
3. The audit found one structural blocker — **no git anywhere across 14 folders** — which became the third thread. Mostly it was me correcting a wrong model of what git is for.

Setup friction and tool bugs from today are collected under Gotchas at the end, rather than competing with the three real topics for space.

---

## Topic 1 — The AI Proficiency Insights Plugin (v2.4.0, Intuit DevAssist registry)

**Bottom line:** the plugin works and is genuinely private, but it refused to give me an overall level — correctly, because only 3 of the 6 categories had any evidence in my logs.

**What I learned**

* **A plugin is visible only where it registers tools.** This one ships a skill and **no MCP server**, so Cursor never discovered it despite a successful install — it landed in `~/.claude/plugins/cache/`, which Cursor does not scan. By contrast `singular-plugin` ships four MCP servers and appeared at once. Registration, not installation, is what crosses from Claude Code to Cursor.
* **There are three kinds of zero and only one is about me.** Cost discipline read 0% because Cursor *cannot physically record* a model name or token count. Zero trust read 0% because no occasion arose in 30 days. A third kind — low but non-zero — would mean the occasion came and I passed. Only the third describes how I work.
* **A share is meaningless until you know the denominator.** My 29 corrections are 4% of prompts but **79% of session-weight**, because they appear in 7 of 16 sessions. Sessions are weighted by size and capped at 10 prompts, so spread across sessions is what marks a practice as real — one big session cannot buy a level, and padding cannot either.
* **It refused to score me, on purpose.** An overall requires at least 4 measured categories and I had 3. The reasoning in the source: a floor computed over two or three categories *rises* as categories drop out, so doing less would read as improvement.
* **I do not delegate to agents at all.** 5 subagent spawns in 30 days, every one started by the agent. Zero prompts from me requested parallel or decomposed work — which contradicts what I would have guessed.

**Evidence**

```bash
ls ~/.cursor/plugins/cache/*/          # only amplitude, omni-analytics
rg -il "proficien" ~/.cursor ~/.claude --max-depth 6
```

*Output Verified:* found at `~/.claude/plugins/cache/devassist-plugins-registry/ai-proficiency-insights/2.4.0/`.

```bash
ln -sfn ~/.claude/plugins/cache/devassist-plugins-registry/ai-proficiency-insights/2.4.0/skills/ai-proficiency-private-coach \
        ~/.cursor/skills/ai-proficiency-private-coach
```

A symlink rather than a copy, so plugin updates carry through.

```bash
S="$HOME/.cursor/skills/ai-proficiency-private-coach/scripts/extract-signals.js"
node "$S"                 # default: last 7 days
node "$S" --month         # 30 days (the hard cap)
node "$S" --month --json  # full evidence, every prompt
node "$S" --session current
```

*Output Verified (7 days):* 4 sessions, 172 prompts — below the 5-session minimum, so no levels at all.

*Output Verified (30 days):* 16 sessions, 668 prompts, 3,758 tool calls, 3,880 assistant turns, 2026-09-11 to 2026-10-06.

**Detail**

The full reading:

| Category | Result | Evidence |
|---|---|---|
| Governance | **Level 3 · Orchestrating** | 79% of session-weight; 29 corrections, 4 validity challenges, 21 interrupts, across 7 of 16 sessions |
| Reuse & contribution | **Level 2 · Directing** | 40%; 8 prompts in 5 of 16 sessions |
| Context & guardrails | **Level 2 · Directing** | 26%; 5 prompts in 3 of 16 sessions |
| Decomposition & scoping | *no level — no occasion seen* | 0 parallelism prompts; 5 spawns, all agent-initiated |
| Zero trust | *no level — no occasion seen* | 0 access-scoping prompts; no standing config rules |
| Cost discipline | *not observable* | all-Cursor window; Cursor records no model or token data |
| Chaining & integration | *not computed* | no cross-session signal exists |

Trend across the window halves: corrections 5% → 3%, context 1% → 1%, reuse 2% → 1%. Every swing under 10 points, so flat rather than directional.

The scoring policy, read from the source:

```js
minSessions: 5,              // below this, ratios are noise -> no levels at all
routineShare: 60,            // >=60% of session-weight = Level 3 (the "routine" bar)
sessionWeightCapPrompts: 10, // sessions count in proportion to size, capped
overallMinCategories: 4,     // below this, no overall is reported
levelEmptyCategories: false, // a 0% share gets NO level (not Level 1)
costMixCorroboratesOnly: true, // a call histogram is not a decision
```

What the matchers look for, if you want a category to register. Patterns require an intent shape — asking, instructing, challenging — and run only on typed prose, with pasted material stripped first.

* Decomposition: "run these in parallel", "spawn three agents", "one agent per metric", "fan out". Negations excluded.
* Zero trust: "least privilege", "scope the agent's access", "read-only", "only touch these files", allowlist or denylist.
* Context: `AGENTS.md`, `.cursorrules`, "going forward", "from now on", "remember for future".
* Reuse: "do we already have a helper", "is there an existing…", "turn this into a skill".
* Cost: "switch to Sonnet", "which model", "too expensive", "token spend", "overkill for this task".

My nearest zero-trust prompt was *"Wait for 5 mins you will get access to this too, i already gave access"* — granting access, the opposite direction, which is why it did not count.

A contradiction between the docs and the code: the README says *"Cursor has a `Task` tool too, so decomposition is fully visible on both, and an absence there is a real finding."* But `levelEmptyCategories: false` applies uniformly, so my 0% was filed as *not-exercised*. Had it levelled as the prose implies, I would have had 4 measured categories and an overall of **Level 1 · Tasking, floored by decomposition, ceiling Level 3**.

The method's blind spots: Cursor transcripts carry no timestamps, so that side of the window is dated by file modification time — last write, not session start. The window also excluded 2,386 events including 656 prompts from 5 sessions that began before it opened.

---

## Topic 2 — Auditing How Fully I Use Cursor

**Bottom line:** no published rubric measures one person's tool mastery, but Cursor ships a real usage leaderboard — and my own audit found a long list of capabilities I have never touched.

**What I learned**

* **Cursor publishes no per-feature adoption percentages.** Any "X% of Cursor users adopt skills" figure circulating online is third-party and unverified. The only official adoption numbers anywhere are Bugbot's: 110,000+ repos with learned rules enabled, and a 78.13% finding-resolution rate across 50,310 PRs, against Greptile 63.49%, CodeRabbit 48.96%, Copilot 46.69%.
* **Real usage rankings exist inside the product.** The **Customize page** (Cursor v3.9+) carries a leaderboard of the most-used plugins, skills and MCP servers, both within my team and across the whole Cursor community, with 30-day counts and an agent-initiated versus human-initiated split.
* **No peer-reviewed rubric exists for one individual's AI tool mastery.** The credible published model is **DORA's AI Capabilities Model** (2025, 78 interviews plus ~5,000 survey respondents), but it measures the organisational environment, not a person. Its thesis: AI is an **amplifier** of existing strengths and weaknesses, and it can actively *harm* teams lacking user focus.
* **Independent rubrics converge, which is itself evidence.** Intuit's 3X guide, an external six-dimension individual rubric, and my own ad-hoc axes land on nearly the same dimensions, and two of the three independently chose weakest-link scoring over averaging.
* **My tool mix is overwhelmingly shell.** Bash is 1,860 calls, 49% of everything, almost entirely `pdl` SQL. MCP is second at 536 across 17 servers. `Task` is 5.

**Evidence**

```bash
cd ~/Documents/CursorWorkingFolders
for d in */; do
  [ -d "$d/.git" ] && echo "GIT $d" || echo "--  $d"
done
find . -name "*.canvas.tsx" -maxdepth 4   # zero results
find . -name "AGENTS.md"    -maxdepth 3   # zero results
```

*Output Verified:* 0 of 14 project folders are git repositories. No Canvas files, no `AGENTS.md`, no `.cursorignore`, empty `~/.cursor/agents/`, no `commands/` directory.

Full 30-day tool counts from the extractor:

```
Bash          1860   (49% of all calls)
mcp__*         536   (17 configured servers)
Edit           481
Read           316
AskQuestion    159
Write           91
Grep            78
TodoWrite       75
CreatePlan      54   (plus 661 plan files all-time — Plan mode is a real habit)
EditNotebook    33
Glob            28
SwitchMode      12
WebSearch        9
Task             5   (all agent-initiated)
```

**Detail**

Confirmed unused: Canvas, Automations, self-authored hooks, custom subagents, custom slash commands, `AGENTS.md`, `.cursorignore`, Memories, Ask mode, Debug mode, custom modes, worktrees, Cloud Agents, Bugbot, and the `cursor-agent` CLI (installed at `~/.local/bin/cursor-agent`, never used). My `hooks.json` contains only Intuit Onyx scanner and audit-logger entries, none written by me.

A rules problem the audit exposed: 8 global rules in `~/.cursor/rules/` apply to all 14 projects, and only 1 project has local rules. My Omni analysis work and my Jira work therefore receive identical instructions.

The Team Analytics API exposes `/analytics/team/mcp`, `/skills`, `/commands`, `/plans` and `/ask-mode`, plus `/analytics/by-user/*` equivalents, documented at <https://cursor.com/docs/account/teams/analytics-api>. It requires Team or Enterprise admin rights.

DORA's seven capabilities, for reference: clear and communicated AI stance; healthy data ecosystems; AI-accessible internal data; strong version control practices; working in small batches; user-centric focus; quality internal platforms. Report at <https://dora.dev/ai/capabilities-model/report/>.

---

## Topic 3 — Git for Exploratory Data Science

**Bottom line:** I had been judging a private time machine by the rules of a publishing system, which is why git looked useless for messy analysis work.

**What I learned**

* **`git commit` and `git push` are different machines.** Commit writes into a hidden `.git` folder on my own disk. Push sends commits to a server, and never runs by itself. A repository created with `git init` and no remote has nowhere to send anything, so its history physically cannot leave the laptop.
* **A repository is a folder tree, and git searches upward.** It starts at whichever folder holds `.git` and covers everything beneath. Any git command walks up the directory chain to find it.
* **I can choose files, not folders.** `git add` picks which files enter a commit, but a commit spans the whole repository — so I cannot push a single subfolder. That is the argument for one repository per analysis rather than one monorepo across all 14.
* **Git does two jobs and I had only considered one.** As a publishing system the audience is other people, so junk matters and selectivity is right. As a local time machine there is no audience, so junk costs nothing.
* **DORA found this matters at the individual level**, not only the team level: frequent commits measurably amplify AI's positive influence on one person's effectiveness. And a local `git init` has no connection whatsoever to `qbo_product` — my worry that Intuit repo policy would block this was misplaced.

**Evidence**

The concrete loss, from my own `Performance` project log — a question Cursor could not answer:

> "is ther a way to back track in git and see whether there is a US filter anyweher before?"

It had to reach for the GitHub MCP server to read a remote file, which is slow and cannot produce a diff. A final notebook cannot say when a denominator changed or why; only the history of the mess can.

Also from my logs, on the current Databricks-side workflow:

> "Git pus from datbricks is also panful nowbecause of thiswhich has and additional branch"

**Detail**

The arrangement: one local repository per analysis folder, no remote. Commit freely, junk included. Folder structure and Cursor workflow unchanged.

```
.gitignore
----------
*.csv
*.parquet
.DS_Store
old_archives/
.ipynb_checkpoints/
```

`.gitignore` is a **block list** by default. Allow-list behaviour needs the `*` then `!pattern` idiom, which suits a future deliverable repository rather than a local history one.

A possible flow reversal, later. Today: Cursor → Databricks → git. Better: Cursor → git → Databricks, with Databricks Git folders *pulling* from the remote, which removes the painful Databricks-side push entirely.

Notebook diffs: `.ipynb` is JSON and stores cell outputs, so diffs are noisy. Three options — accept it for a local history repo, strip outputs before committing, or use Databricks source format (`.py` with `# COMMAND ----------` separators). Databricks Git folders accept both.

---

## Gotchas

Durable facts about the toolchain that cost time today but do not expand capability.

* **Cursor's markdown preview stops at the first indented code fence** and silently drops everything after it. Fences nested inside bullets are legal CommonMark — `marked` rendered this file's full 26 KB and all 7 `<h2>` headings without complaint — but Cursor's preview halted at section 3. The failure looks like *missing* content rather than broken content, so it reads as "sections are gone" instead of "the renderer broke". Fix: bold lead-in line, then start the fence at column zero.
* **Verify a markdown file independently before blaming its content:** `npx marked -i file.md -o /tmp/out.html && grep -o '<h2[^>]*>[^<]*' /tmp/out.html`. If the parser sees every heading, the file is fine and the renderer is not.

---

## Review Questions

1. The proficiency tool gave me no overall level. What exactly triggered the refusal, and what is the reasoning behind that rule?
2. A plugin installed successfully under Claude Code but never appeared in Cursor. What determines whether a plugin crosses between the two, and what is the one-line fix?
3. My 29 corrections are 4% of prompts but 79% of something else. What is the denominator, and why was it chosen over raw prompt count?
4. If I run `git init` in a folder and never configure a remote, what can leave my laptop?
5. Why one repository per analysis folder rather than one repository covering all 14?

## Next Steps

* **Highest priority.** Issue my first deliberate decomposition prompt in a real analysis session — *"spawn one subagent per table to check these four schemas in parallel"*. This moves Decomposition off zero and gives the rubric its fourth measured category, which would finally produce an overall reading.
* **Topic 1.** Report the README-versus-code contradiction on decomposition to `#intuit-builders`.
* **Topic 2.** Open the Customize page and record the real leaderboard rankings instead of guessing which capabilities are worth adopting — first target is Canvas, which fits dashboard and metric-table work exactly and is entirely unused. Also check whether I hold Team or Enterprise admin rights for the Analytics API.
* **Topic 3.** Run `git init` in this `365_Oct4` folder as the zero-risk training ground: no Intuit data, already meant to be public, and a natural one-commit-per-day rhythm. Separately, verify whether Intuit restricts which Git servers Databricks may connect to, whether scheduled jobs are limited to `qbo_product` and `qbo_product_sandbox`, and whether Cloud Agents can reach `github.intuit.com` from an external VM.
