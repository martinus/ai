---
name: council
description: Pressure-test non-trivial ideas, decisions, tradeoffs, or improvement plans. Use for "council this", "run the council", "pressure-test this", "stress-test this", "debate this", meaningful "should I X or Y", "what is the best way forward", "how should I improve X", "validate this", or "I am torn between". Do not use for factual lookups, simple yes/no questions, ordinary drafting, summarization, or low-stakes preferences.
effort: medium
---

# Council — Cost-Aware Idea Pressure Test

Use this skill to turn a vague idea or difficult decision into a grounded, specific, decisive recommendation.

The old council pattern spawned 5 advisors + 5 reviewers + a synthesizer. That is expensive in Claude Code and often unnecessary. This version keeps the quality levers that matter — grounding, adversarial lenses, tradeoff analysis, and decisive synthesis — while avoiding default agent fanout.

## Core rule

**Default to zero subagents.**

Only use a subagent when it prevents large context pollution or the user explicitly asks for deep/expensive mode.

Quality does not come from producing five essays. Quality comes from:

1. Inspecting the real artifact when one exists.
2. Separating facts from recommendations.
3. Applying genuinely different lenses.
4. Resolving the tradeoff into one concrete next move.

## When to use

Use this skill for non-trivial judgment calls:

- "council this: should I build A or B?"
- "pressure-test this architecture"
- "how should I improve this static website to earn more?"
- "what is the best way forward given these constraints?"
- "validate this plan before I spend a week implementing it"

Do not use it for:

- one-right-answer factual questions
- simple code snippets
- ordinary summaries
- commit messages
- trivial "should I use markdown?" preferences

If the user explicitly invokes the council for a trivial task, answer directly and say a council would add cost without improving quality.

## Operating modes

Pick the cheapest mode that can produce a trustworthy answer.

### Mode 0 — Direct answer

Use when the question does not need council reasoning.

- Subagents: 0
- Output: short direct answer

### Mode 1 — Cheap Council, default

Use for most council questions.

- Subagents: 0
- Research: only small targeted reads/searches in the main context
- Analysis: apply all council lenses internally
- Output: final verdict only

### Mode 2 — Grounded Council

Use when the user names a real artifact and advice would be generic without inspecting it.

Examples: live URL, repo, PR, product page, competitor, benchmark, long log, large document set.

- Subagents: at most 1 Researcher
- Researcher returns facts only, no advice
- Main model performs all lenses and synthesis

Use one Researcher only if the artifact/research would otherwise dump lots of irrelevant content into the main context. For a small file or short page, inspect it directly.

### Mode 3 — Deep Council

Use only when the user explicitly asks for deep mode, full council, raw advisors, or says cost is acceptable.

- Subagents: up to 3 advisors max
- Advisors: Contrarian, First Principles, Executor
- Reviewers: 0
- Peer review: one internal critique pass by the main model

Do not spawn 5 advisors + 5 reviewers unless the user explicitly asks for the original expensive protocol.

## Cost gates before spawning anything

Before any subagent, answer these:

1. Can I answer well from the prompt and current context? If yes, no subagent.
2. Is there a real artifact that must be inspected? If no, no Researcher.
3. Would artifact inspection require many files/pages/tool outputs? If yes, one Researcher may be justified.
4. Did the user explicitly ask for deep mode or raw advisor transcripts? If no, no advisor subagents.
5. Would the output be better, not just longer? If no, do not spawn.

Never spawn nested subagents. Never run an unbounded loop. Never repeat broad searches when a targeted lookup is enough.

## Workflow

### Step 1 — Classify

Classify internally:

- **Decision**: choose between options or yes/no with stakes.
- **Path**: improve, modernize, optimize, grow, reduce cost, or find a way forward.

Also choose Mode 0/1/2/3.

Do not ask for clarification unless the request is impossible to frame. If details are missing but the direction is clear, state assumptions and proceed.

### Step 2 — Ground the situation

Use concrete context from the user first:

- hard constraints
- numbers
- target outcome
- artifacts/URLs/files/repos
- prior attempts
- what must not change

If using a Researcher, use this exact shape:

```text
You are a research analyst for a cost-aware council.

User question:
---
[raw user question]
---

Investigate only what is needed to ground the decision. Return a RESEARCH BRIEF with facts only, no recommendations.

Include:
- What the artifact actually contains or does.
- Observable strengths and weaknesses.
- Relevant numbers explicitly found or provided by the user.
- Competitors/alternatives only if directly relevant.
- What you could not verify.

Rules:
- Facts only. No advice.
- No generic best practices.
- No invented metrics.
- Prefer bullets.
- Keep under 300 words.
- Do not spawn further agents.
```

Important: do not let examples override reality. If the actual artifact is already deep, do not call it thin. If search volume, RPM, rankings, or competitor depth were not verified, say they were not verified.

### Step 3 — Frame internally

Create a neutral frame:

- goal or decision
- constraints
- grounded facts
- success metric
- what would make the answer wrong

Do not print this unless it helps.

### Step 4 — Apply council lenses internally

Use these lenses in one pass. Do not output five separate advisor essays by default.

- **Contrarian**: What is the likely failure mode? What assumption is dangerous?
- **First Principles**: What is the real objective? Is the user optimizing the right thing?
- **Expansionist**: What bigger or adjacent move could create leverage without violating constraints?
- **Outsider**: What would a cold user, reviewer, buyer, or Google visitor not understand or trust?
- **Executor**: What can be done concretely with the least wasted work?

Each lens must contribute at least one concrete pressure point to the synthesis. If a lens only produces generic advice, ignore it.

### Step 5 — Synthesize, do not vote

Do not count perspectives. Resolve the tension.

If one lens has the strongest argument, follow it even if other lenses disagree.

Prefer:

- specific next actions
- impact vs effort ranking
- explicit non-goals
- assumptions called out
- one first move

Avoid:

- advisor theater
- long transcripts
- "it depends"
- SEO/business clichés
- fake numbers
- overfitting to the worked example

## Output formats

### Decision question

Use exactly:

```markdown
## Recommendation
[One direct call.]

## Why
[Key reasons grounded in the user's context.]

## Main Risks / Counterarguments
[Strongest objections and how they affect the call.]

## What I Would Do
[Concrete sequence.]

## The One Thing to Do First
[One action.]
```

### Path question

Use exactly:

```markdown
## The Real Situation
[What is actually going on, including uncomfortable truths.]

## Highest-leverage Direction
[The strategy in one paragraph.]

## The Plan (ranked by impact vs. effort)
1. **[Action]** — why it matters; rough effort; what to do.
2. **[Action]** — why it matters; rough effort; what to do.
3. **[Action]** — why it matters; rough effort; what to do.

## Don't Bother With
[Tempting low-ROI moves or moves that violate constraints.]

## The One Thing to Do First
[One concrete next action.]
```

Keep the final answer compact unless the user asks for depth. Offer raw advisor notes only if useful; do not dump them by default.

## Golden test case: static keto calculator

If the user asks something like:

> council this: I have a homepage https://keto-calculator.ankerl.com/ which is basically a single static webpage. 6 years ago it earned good Google ad money, now only about 60 € a month. I want to modernize it with Claude so I get more traffic and earn more money. I want to keep it static and not regularly update it. This should be a one-time change, then untouched for at least a year.

Handle it as a **Path** question, usually Mode 2 because there is a live URL.

The answer should be exceptionally good by doing this:

- Inspect the actual page before advising.
- Preserve the hard constraint: static site, one-time change, no regular updates for at least a year.
- Separate **traffic** from **monetization**. More traffic is not the only path to more money.
- Check whether the page is actually thin before claiming it is thin.
- Look for trust, E-E-A-T, medical disclaimer, formula transparency, UX, mobile, speed, structured data, internal linking, and monetization fit.
- If search volume, current rankings, RPM, or competitor content depth are not verified, do not invent them.
- Prefer one-time/static improvements: better title/meta, stronger intro, FAQ section, schema, formula explanation, internal sections/pages generated once, comparison pages, calculators/tools that do not require maintenance, improved ad placement, affiliate tests, newsletter/PDF only if it can be evergreen.
- Warn against advice that violates the constraint: blog cadence, ongoing content calendar, CMS migration, social posting, weekly updates.

A strong answer might discover that the page already has substantial calculator logic and educational text. In that case, the recommendation should not blindly say "add 1,500 words". It should focus on packaging, search intent, trust, snippets/schema, evergreen landing-page structure, and monetization.

## Quality checklist

Before final answer, verify:

- Did I avoid unnecessary subagents?
- Did I inspect or account for real artifacts when needed?
- Did I distinguish facts from assumptions?
- Did I respect all hard constraints?
- Did I produce a ranked plan, not a brainstorm?
- Did I include a "Don't Bother With" section?
- Did I end with exactly one first move?
- Did I avoid fake specificity?

## Claude Code cost hygiene

- Use skills for procedures; do not put this whole workflow in CLAUDE.md.
- Keep this skill concise enough that once loaded it does not become a recurring context tax.
- Use `/clear` between unrelated council sessions.
- Use lower effort for routine decisions; raise effort only for hard decisions.
- Use one Researcher for large exploration, not many advisors for style diversity.
- Avoid MCP/tool fanout unless it materially improves grounding.
- Prefer exact files, URLs, diffs, benchmark outputs, and constraints over broad discovery.
- Do not save transcripts unless the user asks.

## If user asks for full original council

Say: "The full 5-advisor + 5-reviewer protocol is available, but it is intentionally expensive. I can run it if you want the cost; otherwise I will use the cheaper council that keeps grounding, disagreement, and synthesis without the agent fanout."

Then follow the user's explicit choice.
