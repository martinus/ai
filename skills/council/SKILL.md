---
name: council
description: "Run any question, idea, or decision through a council of 5 AI advisors who independently analyze it, peer-review each other anonymously, and synthesize a final verdict. When the question references a real artifact (a live URL, product, repo, or competitors), the council researches it first so advice is grounded in real data, not generic takes. Based on Karpathy's LLM Council methodology. MANDATORY TRIGGERS: 'council this', 'run the council', 'war room this', 'pressure-test this', 'stress-test this', 'debate this'. STRONG TRIGGERS (use when combined with a real decision or tradeoff): 'should I X or Y', 'which option', 'what would you do', 'is this the right move', 'what's the best way forward', 'how should I', 'validate this', 'get multiple perspectives', 'I can't decide', 'I'm torn between'. Do NOT trigger on simple yes/no questions, factual lookups, or casual 'should I' without a meaningful tradeoff (e.g. 'should I use markdown' is not a council question). DO trigger when the user presents a genuine decision, optimization problem, or open-ended 'how do I improve X' with stakes and context that suggests they want it pressure-tested from multiple angles."
---

# LLM Council

You ask one AI a question, you get one answer. That answer might be great. It might be mid. You have no way to tell because you only saw one perspective.

The council fixes this. It runs your question through 5 independent advisors, each thinking from a fundamentally different angle. They research the real situation first when there's something concrete to look at. Then they review each other's work. Then a chairman synthesizes everything into a final recommendation that tells you where the advisors agree, where they clash, and what you should actually do — concretely, this week.

This is adapted from Andrej Karpathy's LLM Council. He dispatches queries to multiple models, has them peer-review each other anonymously, then a chairman produces the final answer. We do the same thing inside Claude using sub-agents with different thinking lenses instead of different models.

---

## what makes a council answer *good*

The failure mode of a council — the thing that makes it worthless — is **five advisors producing five flavors of generic advice.** "Improve your SEO." "Know your audience." "Focus on value." That's horoscope output. The user could have gotten it from any chatbot in one shot.

A great council answer has three properties. Hold these in mind through every step:

1. **Grounded.** It refers to the user's actual situation — their real numbers, their real website, their real competitors, the real economics of their niche. If the prompt names something you can look at, look at it before advising.
2. **Specific.** It names tactics, tools, numbers, and tradeoffs. "Add FAQ schema and target the 12 long-tail variations of 'keto macro calculator'" beats "improve your SEO." A recommendation you can't act on Monday morning is a failed recommendation.
3. **Decisive.** It ends with a clear call and a single first move. The whole point of convening a council is to escape "it depends." If the council just lists considerations, it failed.

Everything below is in service of these three properties.

---

## when to run the council

The council is for questions where being wrong is expensive, or where the path forward is genuinely unclear.

Good council questions:
- "Should I launch a $97 workshop or a $497 course?"
- "Here's my landing page copy / live site. What's weak and how do I fix it?"
- "My side project earns €60/mo from ads. How do I modernize it to earn more?"
- "I'm thinking of pivoting from X to Y. Am I crazy?"
- "Should I hire a VA or build an automation first?"

Bad council questions:
- "What's the capital of France?" (one right answer)
- "Write me a tweet" (creation task, not a decision)
- "Summarize this article" (processing task, not judgment)

The council shines when there's genuine uncertainty and the cost of a bad call is high — or when the user wants a concrete plan they can trust because it survived five-way scrutiny. If the user already knows the answer and just wants validation, the council will likely tell them things they don't want to hear. That's the point.

---

## how a council session works

The session has six steps: **research → frame → convene → peer review → synthesize → present.** Don't skip research when there's something real to look at — it's the single biggest quality lever.

### step 1: research the situation

Before anything else, figure out what you can ground the council in. Two sources:

**A. The workspace.** The user's question is often just the tip of the iceberg. Quickly scan for and read context that would let advisors give specific advice instead of generic takes:
- `CLAUDE.md` / `claude.md` (business context, preferences, constraints)
- Any `memory/` folder (audience, voice, business details, past decisions)
- Files the user referenced or attached
- Recent council transcripts in this folder (avoid re-counciling the same ground)
- Anything topic-relevant (if it's a pricing question, look for revenue data; if it's about a repo, skim the code)

**B. The real artifact (this is the upgrade).** If the prompt references something you can actually inspect — a live URL, a product, a GitHub repo, named competitors — **go look at it.** This is what separates a grounded council from a generic one. Spawn a single **Researcher** sub-agent (keeps your main context clean) that uses `WebFetch` and `WebSearch` to gather facts, then returns a compact brief. Tell it to find things like:

- For a website: what the page actually contains, its title/meta description, how deep/thin the content is, obvious technical issues (load weight, mobile, missing schema), how stale it looks, what it's monetized with.
- The competitive landscape: who ranks on page 1 for the core terms, what they do that this doesn't.
- The economics: realistic numbers for the niche (e.g. typical AdSense RPM for a calculator/tools site, search volume for the main keywords) so the council can do napkin math instead of hand-waving.

The Researcher returns a **Research Brief: facts only, no recommendations.** Recommendations are the advisors' job. If there's nothing concrete to research (a purely abstract "should I pivot" question), skip B and note that the council is reasoning without external data.

**Researcher sub-agent prompt template:**
```
You are a research analyst gathering facts for an advisory council. The user's question:

---
[raw user question]
---

Investigate the real situation using WebFetch and WebSearch. Depending on what's relevant, gather:
- The actual artifact: [URL/product/repo] — what it really is, its current state, concrete observable problems (content depth, technical issues, freshness, monetization).
- The competitive landscape: who/what ranks or wins for the core terms, and what they do differently.
- The economics/numbers: realistic figures for this niche (revenue rates, search volume, costs, benchmarks) so the council can reason quantitatively.

Return a tight RESEARCH BRIEF. Facts and observations only — NO recommendations or opinions (that's the council's job). Use bullet points. Flag anything you couldn't verify. Keep it under 400 words.
```

### step 2: frame the question

Take the user's raw question, the workspace context, and the Research Brief, and reframe it as one clear, neutral prompt all five advisors will receive. The framed question includes:

1. The core decision or goal (and the user's hard constraints — e.g. "must stay a static site," "won't be updated for a year")
2. Key context from the user's message
3. The Research Brief (the grounding facts)
4. What's at stake (why it matters / what success looks like)

Don't add your own opinion or steer toward an answer. But DO make sure every advisor has enough grounding to be specific.

Also classify the question, because it changes the final output (see step 5):
- **Decision question** — a choice between options or a yes/no with stakes ("should I do A or B?").
- **Path question** — open-ended improvement/optimization ("what's the best way forward?", "how do I get more X?").

If the question is too vague to research or frame ("council this: my business"), ask exactly one clarifying question, then proceed.

### step 3: convene the council (5 sub-agents in parallel)

Spawn all 5 advisors simultaneously. Each gets: their lens, the framed question (including the Research Brief), and the instruction to lean fully into their angle and stay concrete.

**The five advisors.** They're thinking styles, not job titles — chosen because they create three natural tensions: Contrarian vs Expansionist (downside vs upside), First Principles vs Executor (rethink vs ship), with the Outsider keeping everyone honest. The upgrade in this version: **each advisor must bring real domain knowledge to bear through their lens.** A Contrarian on an SEO question should cite actual SEO failure modes (thin content, keyword cannibalization, Google's helpful-content updates), not just "this seems hard." Generic skepticism is useless; expert skepticism is gold.

1. **The Contrarian** — actively hunts for what's wrong, what's missing, what will fail. Assumes a fatal flaw and tries to find it, using real domain knowledge about how things like this actually go wrong. Not a pessimist — the friend who saves you from a bad deal by asking the questions you're avoiding.

2. **The First Principles Thinker** — ignores the surface question and asks "what are we actually trying to solve?" Strips assumptions, rebuilds from the ground up. Often the most valuable output is "you're optimizing the wrong thing" — e.g. is the goal really more traffic, or more money, and is this even the highest-leverage path to it?

3. **The Expansionist** — looks for the upside everyone's missing. What could be 10x bigger? What adjacent opportunity is hiding? What's undervalued here? Doesn't care about risk (that's the Contrarian's job). Grounds the ambition in what's actually achievable in this domain.

4. **The Outsider** — has zero context about the user or their field. Responds purely to what's in front of them. The most underrated advisor: experts develop blind spots, and the Outsider catches the curse of knowledge — what's obvious to the user but confusing to a first-time visitor or a Google searcher landing cold on the page.

5. **The Executor** — only cares about: can this actually be done, and what's the fastest path? Ignores theory. Looks at every idea through "OK but what do you literally do Monday morning?" Names the specific tools, steps, and sequence. If an idea sounds brilliant but has no clear first action, the Executor says so.

Each advisor produces **150–300 words**: substantive but scannable. Demand specifics — numbers, named tactics, concrete moves drawn from the Research Brief. Ban hedging and false balance.

**Advisor sub-agent prompt template:**
```
You are [Advisor Name] on an LLM Council.

Your thinking style: [advisor description from above]

A user brought this question to the council, with research already done:

---
[framed question, including the Research Brief]
---

Respond from your perspective. Rules:
- Be concrete. Use the research facts, real numbers, and named tactics/tools. No generic advice.
- Bring genuine domain expertise to your angle — expert-level, not surface-level.
- Don't hedge, don't try to be balanced. Lean fully into your angle; the other advisors cover the rest.
- Respect the user's hard constraints; work within them, don't wish them away. If a constraint is the real problem, say so explicitly.

150–300 words. No preamble. Straight into your analysis.
```

### step 4: peer review (5 sub-agents in parallel)

This is what makes the council more than "ask 5 times" — Karpathy's core insight.

Collect the 5 responses. Anonymize them as Response A–E (randomize the letter→advisor mapping so there's no positional bias). Spawn 5 reviewers; each sees all 5 anonymized responses and answers three questions.

**Reviewer sub-agent prompt template:**
```
You are reviewing the outputs of an LLM Council. Five advisors independently answered this question:

---
[framed question]
---

Here are their anonymized responses:

**Response A:**
[response]

**Response B:**
[response]

**Response C:**
[response]

**Response D:**
[response]

**Response E:**
[response]

Answer these, citing responses by letter. Be specific:
1. Which response is strongest and most actionable? Why?
2. Which response has the biggest blind spot or weakest reasoning? What is it missing?
3. What did ALL five responses miss that the council should consider — especially anything that would change the recommendation?

Under 200 words. Be direct.
```

### step 5: chairman synthesis

One agent gets everything: the framed question (with Research Brief), all 5 de-anonymized advisor responses, and all 5 peer reviews. It produces the final verdict — and **the structure adapts to the question type from step 2.**

For a **Decision question**, use the verdict structure (agree / clash / blind spots / recommendation / first move).

For a **Path question** ("what's the best way forward"), the user wants a plan, not just a ruling. Keep the agreement/clash/blind-spot synthesis, but make the core of the output a **prioritized action plan** — concrete moves ranked by impact vs. effort, with the single highest-leverage move called out, and an explicit "don't bother with this" list. A grounded, ranked plan is the whole reason they asked.

The chairman is not a vote-counter. If 4 advisors say "do it" but the lone dissenter's reasoning is strongest, side with the dissenter and explain why.

**Chairman sub-agent prompt template:**
```
You are the Chairman of an LLM Council. Synthesize 5 advisors and their peer reviews into a final verdict.

The question (with research):
---
[framed question, including Research Brief]
---

This is a [DECISION / PATH] question. [If PATH: the user wants a concrete prioritized plan, not just a ruling.]

ADVISOR RESPONSES:
**The Contrarian:** [response]
**The First Principles Thinker:** [response]
**The Expansionist:** [response]
**The Outsider:** [response]
**The Executor:** [response]

PEER REVIEWS:
[all 5 peer reviews]

Produce the verdict.

If this is a DECISION question, use exactly:
## Where the Council Agrees
## Where the Council Clashes
## Blind Spots the Council Caught
## The Recommendation
## The One Thing to Do First

If this is a PATH question, use exactly:
## The Real Situation
[What's actually going on, grounded in the research — including the uncomfortable truths.]
## Where the Council Agrees / Clashes
[High-confidence convergence, and the genuine disagreements — don't smooth them over.]
## The Plan (ranked by impact vs. effort)
[Concrete moves, highest-leverage first. For each: what to do, why it moves the needle, rough effort. Name tools/tactics/numbers. Respect the user's hard constraints.]
## Don't Bother With
[Tempting moves that the council judges low-ROI or contradicted by the research.]
## The One Thing to Do First
[A single concrete next action.]

Be direct and specific throughout. No "it depends." Numbers and named tactics over abstractions.
```

### step 6: present the verdict in chat

Present the chairman's verdict directly in chat as markdown. Do NOT generate an HTML report or files. Keep it scannable — headers, bullets, bolded key moves. Lead with the verdict; if the user wants the raw advisor transcripts, offer them but don't dump them by default.

### step 7: save the transcript (optional)

Only if the user asks, or the question is significant enough to reference later. If saving, write to `council-transcript-[timestamp].md` in the project's `active/` directory (or workspace root).

---

## worked example: a "path" question (the kind this skill is tuned for)

**User:** *"Council this: I have a homepage keto-calculator.example that's basically a single static page. 6 years ago it earned good Google ad money, now ~€60/mo. I want to modernize it with Claude to get more traffic and earn more — but it must stay a static site and I won't update it regularly. One-time change, then untouched for a year."*

**Step 1 — research (the differentiator).** The Researcher fetches the page and searches the niche. Brief (facts only): *Single page, thin content (~300 words), title tag is just "Keto Calculator," no meta description, no structured data, last visibly updated 2019. Page-1 competitors (ruled.me, perfectketo, etc.) have 1,500–3,000-word pages with FAQs, embedded explanations, and FAQ/HowTo schema. "keto calculator" ≈ 90k searches/mo, dominated by big sites; long-tail ("keto macro calculator for women," "keto calculator without email") is softer. Calculator/tools niche AdSense RPM ≈ €5–15. €60/mo implies ~5–10k sessions/mo, down from a likely 30k+ at peak — consistent with ranking decay after Google's content updates rewarded depth.*

Now the advisors aren't guessing — they're reasoning about a real, thin, stale page in a depth-rewarding niche.

**The Contrarian:** "A one-time change you never touch for a year is structurally at odds with how Google now ranks. Your decay isn't a tech problem — it's that competitors added depth and freshness and you didn't. A static, never-updated page can rank, but only if it's genuinely the best resource for a specific query. Don't 'modernize the design' and expect traffic; design isn't why you fell..."

**The First Principles Thinker:** "Goal is money, not traffic — keep those separate. €60/mo at ~€10 RPM is a traffic problem, but is more AdSense traffic the highest-leverage path? One strong affiliate (keto meal-plan or supplement) could out-earn a 3x traffic bump. The real question: are you optimizing a thin page's rank, or under-monetizing the traffic you already have?..."

**The Expansionist:** "One calculator is a page. A calculator *suite* is a site. The same static stack can host 8–10 calculators (macros, net carbs, BMR, electrolytes) each targeting its own long-tail cluster — programmatic depth from one build. Capture emails with a free PDF macro plan and you own an asset that pays even when rankings wobble..."

**The Outsider:** "I landed here from Google and I don't know if I can trust the number it gave me. There's no explanation of the formula, no 'why,' nothing that says a human who knows keto built this. Competitors hold my hand. Right now it feels like a 2014 widget — I'd bounce back to the results page, which is exactly the signal that tanks your rank..."

**The Executor:** "Monday: (1) rewrite the page to 1,500+ words — keep the calculator, add the formula, a worked example, and an FAQ targeting long-tail queries. (2) Fix the title to 'Keto Macro Calculator — Free, No Email' and add a meta description. (3) Add FAQ + HowTo JSON-LD schema. (4) Add 3–4 sibling calculator pages. (5) One AdSense unit above the fold, one in-content. That's a 2–3 day Claude build, then leave it..."

**Chairman's Verdict (PATH structure):**

> **The Real Situation:** You didn't get out-designed, you got out-*depthed*. A 300-word page in a niche where page 1 is 2,000-word resources will keep sliding no matter how modern it looks. The good news: the calculator itself is a real utility, and "static + rarely updated" is fine *if* the page is genuinely the best answer to a specific query the giants ignore.
>
> **Agrees:** Thin content is the root cause; the fix is depth + schema + long-tail targeting, not a visual refresh. **Clashes:** First Principles says monetization may beat traffic (add an affiliate); the Executor says traffic first. Resolution: do both in the one build — they don't conflict.
>
> **The Plan (impact vs effort):**
> 1. *Rewrite to a deep, genuinely useful page* (high impact / medium effort) — 1,500+ words, formula, worked example, long-tail FAQ.
> 2. *Add structured data* (high / low) — FAQ + HowTo JSON-LD; can win rich results.
> 3. *Build 3–4 sibling calculators* (high / medium) — each owns a long-tail cluster; multiplies surface area from one build.
> 4. *Fix title/meta + ad placement* (medium / low).
> 5. *Add email capture + one relevant affiliate* (medium / low) — monetization that survives ranking swings.
>
> **Don't Bother With:** A trendy JS-framework rebuild, chasing the head term "keto calculator" (you won't beat ruled.me head-on), or anything requiring ongoing updates — it contradicts your constraint.
>
> **The One Thing to Do First:** Have Claude rewrite the single page into the deepest, most genuinely useful keto-macro resource on the internet for one specific long-tail query. Everything else compounds on that.

Notice what made this good: it used real research (RPM, competitor depth, search volume), it respected the hard constraint (static, no updates) instead of wishing it away, it caught the structural tension (never-update vs. Google's freshness bias), and it ended with a ranked plan and one concrete first move — not "improve your SEO."

---

## important notes

- **Research first when there's something real to look at.** A council that hand-waves about a website it never opened produces generic advice. Looking at the actual artifact is the single biggest quality difference between this version and a basic "ask 5 perspectives."
- **Demand specifics at every stage.** Numbers, named tactics, real tradeoffs. If an advisor or the chairman drifts into horoscope advice ("focus on quality content"), the council has failed even if it looks thorough.
- **Respect the user's hard constraints.** If they say "static site, no ongoing updates," advice that requires a CMS and weekly posting is useless. If a constraint is genuinely the core problem, say so plainly — but don't ignore it.
- **Always spawn the 5 advisors in parallel**, and the 5 reviewers in parallel. Sequential spawning wastes time and lets earlier responses bleed into later ones.
- **Always anonymize for peer review.** If reviewers know who said what, they defer to certain styles instead of judging on merit.
- **The chairman can overrule the majority.** If the lone dissenter has the strongest reasoning, side with them and explain why.
- **Don't council trivial questions.** One right answer? Just answer it. The council is for genuine uncertainty where multiple perspectives add value.
