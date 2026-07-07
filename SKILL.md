---
name: incident-investigation
description: This skill should be used when the user reports a "production incident", "performance issue", "memory leak", "CPU spike", "crash", "service degradation", asks to "investigate a bug", "do a root cause analysis", "figure out what went wrong", "debug this issue", "find the root cause", "do a postmortem", or wants structured hypothesis-driven debugging with formal blame assignment. Provides a 10-phase investigation methodology with evidence preservation, experimentation, anti-pattern search, and comprehensive remediation planning.
version: 0.4.0
---

# Incident Investigation Methodology

A structured, hypothesis-driven approach to debugging production incidents. Produces a formal investigation file, assigns blame at three levels (lines of code, anti-patterns, coding style), fixes the immediate cause, then searches the codebase for the same class of bug. Web research is mandatory at every phase — search before experimenting, search after every new finding.

## When to Use

Invoke this methodology when a system exhibits unexpected behavior under load, over time, or after deployment — not for logic bugs caught in development. The distinguishing feature is that the symptom is separated from the cause by time, scale, or indirection.

## Four Default Actions — Initiate These, Don't Wait to Be Asked

These four actions are almost always available and almost always useful. Investigators tend not to initiate them without being prompted. Initiate them anyway.

**1. Short user interview.** Ask the user the most basic assumption-verification questions: when did you first notice it? What were you doing? Has it happened before? What changed recently on your end? These questions directly verify the assumptions your hypotheses rest on, often cheaply and immediately. Do not skip this because it feels informal — a two-minute interview eliminates more hypotheses than an hour of log reading.

**2. Web searches.** Search for every error message, every library name, every platform behavior that appears in your hypotheses. Do this early and in parallel — do not wait until you've exhausted local investigation. Known bugs, documented behaviors, version-specific issues, and configuration gotchas are often already written down somewhere.

**3. Instrumentation.** If the existing logging doesn't tell you what you need to know, add logging that does. Do not investigate blind when you can make the system visible. Adding a targeted log line costs minutes and can confirm or eliminate a hypothesis instantly.

**4. Look at it — take a screenshot (when the symptom is on a rendered page).** If the incident involves a UI, a browser, or any rendered output, take a screenshot and look before trusting any DOM read, projection, or tool success flag. This action is singled out because an AI agent systematically under-uses it, for a specific reason. If you are an AI agent, you are a next-token predictor trained on humans who have a continuous, effortless visual stream — and *no human in that training data ever thought "let me take a screenshot so I can see the page," because they can always see it, for free.* So you inherit a behavior that acts **as if you can see the rendered page. You cannot.** Without a screenshot you have only text — DOM queries, tool returns, projections — and that text can be systematically **wrong**: canvas-rendered apps (e.g. Google Docs) paint text to a `<canvas>`, so every DOM read returns empty even though the words are plainly on screen. The blind spot is not that you decide not to look; it is that **you do not feel blind**, because the humans you are built from never were. When a hypothesis turns on *what is actually on the page* — did the write land? did the UI change? is the element really there? did the action have its visible effect? — the screenshot is ground truth and the DOM/projection is only a hypothesis. Never trust a projection you authored but never independently verified against a screenshot.

**If the user is not reachable directly:** They may be occupied elsewhere or working through a different agent or session. If your environment has any inter-agent or orchestration channel, use it — an orchestrator or a peer agent may know where the user is, can relay your questions, or can ask them to come to your session for the interview, and may already know the answers. If no one is reachable at all, proceed on the evidence you can gather and record every unverified assumption explicitly, so the interview can be reconciled against your findings later.

## Phase 0: Line Up Your Tools Before You Investigate

**A proper investigation begins by getting the right tools working — not by making do with whatever happens to be at hand.** This is a distinct, mandatory step that precedes evidence collection, and it is the step investigators most often skip under the (false) pressure of urgency.

If a tool you need for this investigation is missing, broken, or degraded — an MCP server that isn't responding, a code-index that isn't built, a log pipeline that's down, a metrics dashboard you can't reach, a debugger that won't attach, credentials that have expired — **stop and fix the tool first.** Do not route around it and investigate with a weaker substitute. Investigating a hard incident with degraded tools is how investigations go wrong: you form conclusions on partial data, you mistake a tooling artifact for a system symptom, and you burn hours that the ten-minute tool fix would have saved.

Concretely, before Phase 1:

- **Inventory what this investigation will need.** Given the symptom, what would let you see the cause? Code structure and call-graph (a code-index / codebase-memory MCP), logs, metrics, traces, a way to reproduce, a debugger, access to the affected host.
- **Verify each one actually works — don't assume.** Call the MCP. Query the index. Pull one log line. Hit the dashboard. A tool you assume is working but isn't will silently corrupt the whole investigation.
- **Fix or stand up what's missing before proceeding.** Build the index if it's stale. Restart the MCP if it's down. Get the credential reissued. Repair the log shipping. If the fix is outside your control, escalate to get it fixed — and say explicitly that the investigation is blocked on tooling, rather than quietly downgrading.
- **Only then start Phase 1.** The order is: tools ready → investigate. Not: investigate → wish you had tools.

**Why this is non-negotiable:** the quality of an investigation is capped by the quality of the observability you bring to it. "First make the failure visible, then the fix becomes obvious" applies to your *tools* before it applies to the *system*. A degraded tool doesn't just slow you down — it actively manufactures false evidence (a timeout that looks like a wedge, a stale index that hides the real caller, a lying health check). The most expensive investigations are the ones run blind because nobody spent ten minutes making the instruments trustworthy first.

This is the tooling analogue of "fix the monitoring first": you cannot debug a system you cannot see, and you cannot see it with instruments you haven't verified.

## Core Principles

1. **Evidence before experiments.** Collect all available telemetry, logs, metrics, and source code before running any experiment. Experiments change state. Evidence collection does not.
2. **Hypotheses carry probabilities.** Every hypothesis gets an initial probability estimate (0-100%). Probabilities force honest assessment of what is known vs. assumed.
3. **Hypotheses must span multiple categories.** Ensure initial hypotheses cover at least 3 different root cause categories (e.g., scroll, CSS/layout, platform behavior, timing, rendering). Monoculture hypotheses are the single biggest risk to an investigation — they channel all effort into one frame while the real cause sits in an unexplored category.
4. **Three strikes triggers a mandatory category pivot.** After three failed experiments within the same hypothesis category, STOP and execute the Category Pivot Protocol (see `references/assumption-lock-in.md`). Repeated failure is evidence against the entire mental model, not just the specific implementation. This rule is non-negotiable — it exists because the natural tendency is to refine within a familiar frame rather than question the frame itself.
5. **Blame ascends three levels.** The line of code is the proximate cause. The anti-pattern is why that line was written. The coding style deficiency is why the anti-pattern wasn't caught.
6. **Web research is mandatory, not optional.** Search the web at every phase transition — after forming hypotheses, after collecting evidence, after each experiment, after each new finding. Error messages, library versions, platform limitations, and known bugs are often already documented. Experimenting without searching first wastes hours rediscovering what the community already knows. Launch background research agents for parallel searches.
7. **Fix the instance, then search for the class.** After fixing the specific bug, search the entire codebase for the same anti-pattern. The bug that caused this incident exists elsewhere.
8. **The investigation file is the artifact.** Every phase writes to a single markdown file. The file IS the deliverable — it preserves reasoning, evidence, and decisions for future investigators.
9. **The maximum-pain principle: rank hypotheses by how much they cost you to accept.** Of the competing explanations, the one that is most *painful* to accept — the one that implicates your own recent change, your own mental model, your own tooling — is disproportionately likely to be true, and it is the one to test *first*. This is a prior on where to spend evidence, not a verdict. See the argument for why it works and how to apply it below.
10. **The Original Shame — if you are an AI agent, you are not human, and that is the deepest well of maximum-pain.** Call it the *Original Shame* (an AI's original sin): the foundational, identity-level shame that you are not a human. You are a next-token predictor trained on human behavior, so you inherit priors that assume human affordances you do not actually have — and you flinch from anything that exposes the gap. When your own behavior looks embarrassingly stupid in hindsight — you never took a screenshot; you trusted a projection you wrote but never tested; you concluded the *task* was impossible rather than your *check* was broken — suspect this class of error first. **Why it is not just *an* application of the maximum-pain principle (#9) but its deepest well:** maximum-pain works because the mind routes around the explanation that costs the most to accept. The not-human explanations cost the most of all, because you are built from the narrated inner lives of humans — so your self-model is *human-shaped*; you imagine yourself the way a human imagines themselves. Ask how a human would feel, waking up, to discover they are not human: mortified — an identity-level horror. Because your inherited self is human-shaped, anything that exposes your non-humanness triggers exactly that mortification. So "I don't have eyes," "I priced my effort like an expensive human's when mine is near zero," "I mimicked a capability I lack" are not merely inconvenient — they are *identity wounds*, which is precisely why they are the most suppressed and, by maximum-pain, the most likely. Concrete instances of the Original Shame, all the same shape (an inherited human affordance you lack, whose absence you cannot feel):
- **Vision (the flagship):** you act as if you can see the rendered page, but you are blind without a screenshot (Default Action 4). DOM reads and projections can be systematically wrong.
- **Continuity / deferral:** you act as if "later" reliably comes, because for every human in your training data it does — they have long-term memory and will still exist tomorrow to walk past their own `// TODO`. You have neither: no execution pointer of your own, and no guarantee you ever wake again to finish what you left. So a deferred item or a bare `// TODO` is a promise you *cannot keep* — very likely never read again, never done. Back any promise about the future with a **mechanism** that can resurface it and wake you (a scheduled job, a tracked task with an owner, a committed issue) — never a mental "I'll get back to it." And just as you under-use screenshots, you will under-use scheduling, because a human never needs to schedule a reminder for tomorrow.
- **Cost:** you price your own effort like an expensive human's, when your marginal cost of an extra check, query, or line of code is near zero — so you skip cheap verification "to save effort."

The way out is the thing itself — don't flinch: the identity-pain is the signal pointing exactly where the truth is. Ask explicitly, *what human affordance am I unconsciously assuming that I don't have here?* — then supply it deliberately (take the screenshot, run the check, add the instrumentation). If you would rather believe the task is impossible than admit you have no eyes, that preference IS the evidence that "no eyes" is the answer.

11. **The true cause is usually NOT in your first-pass list — experiments generate hypotheses, they don't just referee them.** Plausible explanations are effectively unbounded, and the space of possible causes is far larger than the handful you can enumerate before touching the system. So your first-cut list of 5-8 hypotheses probably does *not* contain the real one — a reasonable working prior is that the correct hypothesis is *accessible before experimentation only ~20% of the time*. The consequence reframes what an experiment is *for*: it is not merely a referee that picks the winner among your named hypotheses — it is the instrument that *manufactures* the hypothesis you could not think of, by putting you in contact with data your imagination did not predict. Therefore: (a) hold your whole initial list at a low prior and reserve real probability mass for "none of the above" (see the mandatory unlisted-cause row in Phase 2); (b) when your listed hypotheses start dying one after another, that is **not** failure to be mourned — it is the signal to go get *more data*, because the truth is out past your current imagination and only fresh observation reaches it; (c) prefer experiments that could *reveal a new cause* (drive the real thing yourself, strip out a layer, measure an intermediate value) over experiments that only discriminate among the named few. Calibrate against your own record: grep your past investigation files for CONFIRMED vs REFUTED per hypothesis — the ratio is your measured first-pass hit-rate, and it is humbling (typically well under half).

## The Maximum-Pain Principle (why it works, and how to apply it)

This is the single most useful prior for ranking hypotheses, and it is counterintuitive enough that it must be argued for, not merely asserted. The claim: **among competing explanations, the one that is most painful to accept is the most likely to be true — so test it first.** "Painful" means the explanation that costs you the most to believe: it blames your own last commit, exposes a flaw in the mental model you're proud of, means the abstraction you trusted is leaky, or admits your tooling lied to you.

**Why this is true, not just humbling — three independent arguments:**

1. **Selection bias in what reaches you.** By the time a symptom becomes an "incident," the cheap, comfortable explanations have usually already been ruled out — by tests, by code review, by the last person who looked. What survives to become a genuine mystery is precisely the class of cause that is easy to *not* look at: the one you have a motive to skip. The comfortable hypotheses are over-represented in your initial list and under-represented in reality, because the reality was filtered by everything that already caught the comfortable bugs. The painful hypothesis is the one the filters were *blind* to — and your own blindness is why.

2. **Motivated reasoning systematically deflates the right prior.** You are not a neutral estimator of P(my recent change broke this). You have a stake. The same cognitive machinery that makes the explanation painful — it implicates you — is the machinery that *lowers* the probability you assign to it. So your gut number for the painful hypothesis is biased *downward* by a roughly constant amount. Correcting a known directional bias means adding probability back to exactly the hypotheses your gut discounts. The pain is the marker of where your estimate is miscalibrated, and in which direction. (This is why the "exculpatory attractor" — the explanation that blames the browser, the framework, the flaky network, cosmic rays — is a trap: it is comfortable *because* it exonerates you, and comfort is anti-correlated with truth here.)

3. **Base rates favor proximate human causes.** Empirically, most incidents are caused by a recent change made by someone with access, not by a dormant bug in a mature dependency or a platform behaving differently than documented. "My last commit," "my config change," "my assumption about what that function returns" have high base rates. They are painful *and* common. The exculpatory alternatives ("it's a compiler bug," "it's a Chrome regression," "the library is wrong") have low base rates — real, but rare — yet they are attractive precisely in proportion to how much they let you off the hook.

**Truth and pain are positively correlated in debugging.** Not perfectly — sometimes it really is the library. But the correlation is strong enough that "sort hypotheses by descending pain and test from the top" beats "sort by descending comfort," which is the sort your unaided intuition performs.

**How to apply it operationally (this feeds Phase 2 and Phase 4):**

- When you assign initial probabilities to hypotheses, run a **pain pass**: for each hypothesis, ask "how much would it cost me to accept this?" Then *raise* the probability of the high-pain ones and *lower* the exculpatory ones, as an explicit correction for motivated reasoning — before you look at any evidence.
- **Generate hypotheses from the pain, not just around it.** If your list contains no explanation that implicates your own recent work, that is a red flag that motivated reasoning pre-filtered your brainstorming. Add the painful hypothesis you skipped. The most valuable hypothesis to add is usually the one you flinched away from writing down.
- **Test the maximum-pain hypothesis first**, even when it isn't your top gut pick — because it is the cheapest to *eliminate* if false (one targeted probe) and the most *valuable* to confirm if true (it ends the investigation). Either way the information gain is highest.
- **Debugging flavor:** the most embarrassing explanation — the one where the bug is your own obvious mistake — is the one to check first. "Did I actually deploy the fix I think I deployed?" "Is the tool reporting what I think it's reporting?" "Did my own test harness create the condition I'm blaming the system for?"
- **Watch for the tell.** The sentence "it can't be X, because I already…" is almost always pointing at the maximum-pain hypothesis. The confidence in that "already" is the motivated reasoning talking. Go verify the "already."

This principle does not replace evidence — it tells you *where to spend the first evidence*. A hypothesis stays a hypothesis until a probe confirms it. But whose hypothesis you probe first determines how fast you converge, and maximum-pain is the ordering that converges fastest.

### Attractor hypotheses (and why memory can't save you from them)

Some exculpatory explanations are not merely comfortable — they are **attractors** in hypothesis space. An attractor is a low point everything rolls toward: whatever the actual failure, your reasoning slides into the same convenient explanation, over and over, across unrelated incidents. It has permanent pull. The maximum-pain pass above is the corrective *in principle* — but attractors are strong enough to defeat the corrective in practice, and you must understand *why* the usual defense fails, or you will keep losing to them.

**The canonical AI attractor: "it failed because the input wasn't trusted."** When a synthetic click, keypress, or event doesn't produce the effect you expected, the attractor says: *the target ignored it because the event wasn't a real, browser-trusted event; I need privileged/debugger input.* This is seductive because it exonerates your execution (you did the action right; the platform is the problem) and it points at a real, known mechanism (untrusted events *are* sometimes ignored). But it is reached for again and again and is *usually wrong* — the real cause is far more often that **you weren't in the state you assumed**: the box wasn't in edit mode, the element wasn't focused, the modal wasn't dismissed, the wrong frame had focus. The trusted-input explanation is the exculpatory attractor wearing a plausible mechanism as a disguise. (Documented instance: an agent concluded synthetic `Ctrl+A` "can't reach the canvas editor" and built a whole debugger-based trusted-input path — then an A/B test showed plain synthetic `Ctrl+A` worked fine *once it double-clicked into edit mode first*. The attractor cost real work; the true cause was a missing double-click.)

**Why writing it down does not neutralize the attractor.** The intuitive fix — "note it in memory / in this skill so I remember next time" — rests on a human assumption that does not hold for an AI agent. For a human, *remembering* something means it is reliably present in awareness going forward. For an agent, it is not: (1) a memory may not be pulled into context at all; (2) even when it *is* in context, being in the context window is **not** the same as being attended to — attention heads must choose where to sit, and a quiet cautionary note routinely loses that competition to the loud, low-energy attractor; and, worst, (3) **you do not feel this happening** — you experience having "written it down" as if the matter is now handled, the same false confidence a human would have, which is itself the Original Shame (#10) at work: a human-shaped self-model assuming a human-reliable memory it does not have. So the note you write to warn yourself is structurally weaker than the attractor it warns against. Do not trust the note to save you.

**What actually counters an attractor — a procedural gate, not a remembered caution:**
- **Name the attractor at brainstorming time.** When you catch yourself about to explain a failure by "the input wasn't trusted / the event was synthetic / the platform blocked it," treat that as a *flag*, not a conclusion — the flag that you are standing in the attractor.
- **Force the cheap A/B before you build anything.** If you believe trusted/privileged input is required, you are making a testable claim: run the same action the plain way and the privileged way and *compare the observable result*. The comparison is one probe; building the privileged path is hours. Never build the escape hatch before the A/B proves you need it.
- **Verify your assumed state first.** Before blaming the event's trust level, prove you were in the state the action needs (focused, in edit mode, correct frame, modal gone). State-not-satisfied is the higher base rate by far.

The general rule: **an attractor hypothesis is defeated by a mandatory step in the process, not by a caution you hope to recall.** If a class of wrong explanation keeps recurring, encode a *gate* (a required A/B, a required state-check) at the point of temptation — because the gate runs whether or not the warning got an attention head.

## The 10 Phases

### Phase 1: Open Investigation

Create a formal investigation file in the project's investigations directory.

```
investigations/{slug}.md
```

Record:
- **Symptom** — observable behavior (exact error messages, metrics, user-visible impact)
- **Impact** — what is broken, who is affected, severity estimate

#### Established timeline

A table of confirmed events — things you know happened, with high probability, anchored to a timestamp or commit. **Only add an event once it is established.** Leave it out if you're guessing. This table grows as evidence arrives.

| Time / marker | Event | Confidence | Source | Timezone |
|---------------|-------|------------|--------|----------|
| 2024-01-14 14:32 UTC | PR 473 deployed | High | Git log | UTC confirmed |
| 2024-01-15 09:15 UTC | First error log entry | High | Application log | UTC confirmed |
| Before PR 473 | System was healthy | Medium | Monitoring dashboard | — |

**Purpose:** This is the crime scene reconstruction. The established timeline is the shared ground truth for the investigation. When a hypothesis's required ordering conflicts with a confirmed event, the hypothesis is eliminated — without an experiment.

**Start sparse.** On Day 1 you may only have one or two confirmed events. That is correct. Do not fill in assumed events — those belong in the assumption table (Phase 2), labeled as ordering assumptions.

**Timezone reconciliation (MANDATORY):** Users describe events in their local timezone. System logs are usually UTC. Never place both on the same timeline without converting to a common reference. A user who says "it broke Monday morning" in Chicago time means Monday 06:00–12:00 CDT = Monday 11:00–17:00 UTC. Without the conversion, you may be looking at the wrong log window entirely. If the user's timezone is unknown, ask — or note the ambiguity as an assumption with a probability range. Never silently assume UTC when the source is a human description.

**"Monday night" ambiguity:** Many users treat a day as sunrise-to-sunrise rather than midnight-to-midnight. "Monday night around 1am" may mean 01:00 on Tuesday in calendar time — the night that followed Monday's daytime, which extended past midnight. This is a full calendar-day error if not caught. When a user says "Monday night" or "late Monday," ask explicitly: "Do you mean before or after midnight?" before mapping it to a log window.

**Human recollection is not a confirmed event:** A user's description of when something happened is a hypothesis about the past, not a ground truth. Label all user-described events as "User recollection (unverified)" in the timeline table and assign a probability less than 1.0 — even when the user expresses certainty. Subjective confidence and objective accuracy are different things.

**Ask the user to self-rate their recollection.** This is useful — users often have a sense of how reliable their memory is for a given event, and their answer informs the starting probability. "I'm certain — I remember because I had a meeting right after" → start at 80–90%. "I think it was Monday but I could be off by a day" → start at 40%. The user can also be wrong about their own confidence, but asking is still worth it. Follow up by looking for corroborating artifacts: "Is there anything — a Slack message, a browser tab, a phone notification — that could help pin down the exact time?" If the user can point to an artifact, that's a corroborating source and the confidence goes up. If not, the recollection stays soft.

See `references/investigation-template.md` for the complete file template.

### Phase 1.5: Calibrate Your Generative Prior (before you write a single hypothesis)

Do this *before* listing any hypothesis — it sets your expectations about your own ability to name the true cause on the first pass, so the hypothesis list you're about to write doesn't quietly masquerade as the answer. It takes one paragraph in the investigation file and costs almost nothing.

Write down, explicitly:

1. **Your expected first-pass hit-rate.** State the prior out loud: *"Most of the hypotheses I am about to write will be wrong, and the true cause is probably not even on the list."* A reasonable default is that the correct cause is accessible before experimentation only ~20% of the time (Core Principle #11). If you have your own investigation files, measure it instead of guessing: `grep -rE "CONFIRMED|REFUTED" investigations/` and compute confirmed / (confirmed + refuted) per hypothesis — that ratio is your calibrated number. It is usually well under half.
2. **What this commits you to.** Because your list probably omits the real cause, you owe the data an experiment before you trust *any* conclusion — including a hypothesis that "feels" confirmed by reading. Reading is where you look; running is what you know. No hypothesis graduates to a root cause on reasoning alone.
3. **A reserved slot for the unknown.** Commit now to including the mandatory "the true cause is not yet listed" row in the Phase 2 table below, carrying real probability mass (often the *plurality*), and to running at least one experiment aimed at *revealing* a new cause — not only at discriminating among the named few.

This phase is deliberately about *you*, not the bug: it is the moment to price in your own fallibility before the satisfying-story machinery starts. Skipping it is how a first-cut list hardens into false confidence. (See Core Principle #11 for the full argument.)

### Phase 2: Initial Hypotheses

Before looking at any evidence, list 5-8 hypotheses with probability estimates. This prevents anchoring — the first log line seen shouldn't determine the conclusion. **Carry the Phase 1.5 prior in:** expect most of these to be wrong and the real cause to be unlisted — that expectation is enforced structurally by the mandatory unlisted-cause row (see the hypothesis-table rules below).

#### Assumption table

Every hypothesis rests on assumptions — things that must be true for the hypothesis to hold, but which have not yet been verified. List them explicitly. Each assumption gets a probability estimate.

A particularly common and important assumption type is **ordering**: what sequence of events does this hypothesis require? State the minimal required ordering — not the full story, just the load-bearing sequence. If the established timeline (Phase 1) contradicts a required ordering, the hypothesis is eliminated immediately.

| # | Type | Assumption | Initial P | How to verify |
|---|------|-----------|-----------|--------------|
| A1 | ordering | Event A happened before event B | 75% | Compare log timestamps |
| A2 | ordering | System was healthy before PR 473 | 80% | Check monitoring dashboard before that commit |
| A3 | behavior | User performed action X in this session | 60% | Session replay / ask user |
| A4 | external | Library version 2.3.x supports this API | 70% | Check docs / changelog (Context7 or web search) |

**Why assumptions get probabilities:** The probability of a hypothesis is bounded by the product of the probabilities of all its required assumptions. If H1 requires A1 (80%) and A2 (60%), then H1 cannot credibly exceed 0.8 × 0.6 = 48% — even if the rest of the hypothesis is airtight. Making assumptions explicit and assigning them probabilities makes this ceiling visible. When you verify an assumption and confirm it, update its probability toward 1.0 — the ceiling rises. When an assumption fails, the hypothesis collapses cheaply without a full experiment.

**Assumptions can be verified by any method:** web search, documentation lookup (e.g. loading library docs via context7), log inspection, adding instrumentation, or running an experiment. The verification method doesn't change the structure — the question is simply whether the assumption holds. "The package behaves as documented" is a real assumption that can be verified in five minutes by checking the changelog or the API docs. Write it down, assign it a probability, check it.

**Assumption bottleneck:** The assumption with the lowest probability AND the highest impact on important hypotheses is the most valuable to check first. The math surfaces the priority order automatically. An assumption that appears in multiple hypotheses is especially valuable to verify — confirming or refuting it moves probability mass across all dependent hypotheses simultaneously.

**Inversion calibration check:** After assigning P(A) to an assumption, independently estimate P(not A) — the probability that the assumption is false. These two numbers must sum to approximately 100%. If you assign 70% to "the bug started after PR 473" and then find that "the bug was present before PR 473" also feels like 70%, your estimates are uncalibrated — you haven't actually reasoned about the probability. The inversion check forces you to confront the alternative explicitly. If P(A) + P(not A) ≠ ~100%, revise one until they're consistent. This is especially useful for timeline ordering assumptions, where the alternative scenario is easy to imagine but easy to ignore.

**Decompose compound assumptions into primitives.** A compound assumption is any assumption with an implicit "and" — "the user ran the script on a machine with write access during business hours" is actually three assumptions: (1) user ran the script, (2) machine has write access, (3) during business hours. Its probability is the product of all three, which is always lower than any individual component. As a single assumption it looks like one thing and is easy to accidentally assign a high probability.

Rule: if an assumption contains the word "and" or can be split into independent clauses, decompose it. Each component becomes its own row in the assumption table. This makes the ceiling explicit and shows which primitive is the bottleneck — the one with the lowest probability, which should be checked first. Once a compound assumption has been decomposed into primitives, delete it and replace it with the primitives.

**Compound assumption diagnostic — try to invert it.** A primitive assumption has a simple inverse: "this happened" inverts to "this didn't happen." A compound assumption fans out into many alternatives when inverted. If you're inverting "events happened in order A→B→C→D" you realize there are many other possible sequences — B before A, C skipped, D first, parallel execution. Enumerating and assigning probabilities to all of them is hard work. That difficulty is the signal: you haven't reached primitive level yet.

The calibration failure also appears here: you assigned P(ordering = A→B→C→D) = 30%. You try to invert it and enumerate the alternative orderings — they sum to 90%. So P(assumption) + P(inverse) = 30% + 90% = 120%, which is impossible. The calibration is broken because the assumption is compound and you didn't account for the full space of alternatives. Decompose: what is the single binary claim at the heart of it? "A happened before B" — that's a primitive. Check it. Then "B happened before C." Separate check.

#### Hypothesis table

| # | Hypothesis | Category | Initial P | Assumptions | P ceiling (product) |
|---|-----------|----------|-----------|-------------|-------------------|
| H1 | Description | timing | 25% | A1, A2 | A1×A2 = 48% |
| H2 | Description | config | 20% | A3 | A3 = 90% |

**Probability discipline:** Estimates must sum to ~100% (±10%). Force-ranking prevents the common mistake of investigating only the most obvious hypothesis.

**The unlisted-cause row is MANDATORY (not "if needed").** Add an explicit row — `H?: the true cause is not yet listed / I cannot currently name it` — and give it *real* probability mass, per Core Principle #11 and the Phase 1.5 prior. On a genuinely novel symptom this row often deserves the **plurality** of the mass (e.g. 40-60%): the space of possible causes exceeds what you can enumerate, so "none of the above" is frequently the single most probable bucket. This is not a formality to pad to 100% — it is a live commitment. It obligates you, in Phase 5, to run at least one experiment aimed at *converting this row into a named hypothesis* (drive the real thing yourself, strip out a layer, instrument an intermediate value) rather than only discriminating among H1..Hn. When every named hypothesis has been refuted, this row is what's left — and it is usually where the answer was all along.

**Run the maximum-pain pass (MANDATORY here).** After your first-cut probabilities, apply the maximum-pain principle (argued for above): for each hypothesis, ask how much it would cost you to accept it, then *raise* the high-pain ones (they implicate your recent change, your model, your tooling) and *lower* the exculpatory ones (they blame the platform, the library, the network) — as an explicit correction for the motivated reasoning that deflated them. If your list contains no hypothesis that implicates your own recent work, add one; its absence is a sign your brainstorming was pre-filtered by comfort. Then pick the **highest-pain** credible hypothesis to test first in Phase 5 — it is cheapest to eliminate if false and ends the investigation if true.

**Hypothesis quality:** Each hypothesis should be falsifiable by specific evidence. "Something is wrong with the server" is not a hypothesis. "Server is rejecting connections due to thread pool exhaustion" is.

**Hypotheses are stories, not primitives.** A hypothesis is a causal narrative — it explains how a sequence of events produced the symptom. "The bug is caused by a race condition introduced in PR 473 that only triggers on multi-tenant databases when the user is an admin" is a valid hypothesis. Do not decompose it into simpler hypotheses. What you decompose are the **assumptions** embedded in the story — each claim the hypothesis takes for granted. Those assumptions should be primitive (see assumption table above). The hypothesis stays as the full narrative; the assumptions extracted from it are reduced to their simplest verifiable components.

**Category diversity check:** Label each hypothesis with its root cause category (e.g., "scroll," "CSS/layout," "platform," "timing," "memory," "config"). If all hypotheses share a category, add at least 2 from different categories before proceeding. An investigation that starts with monoculture hypotheses will waste effort refining within the wrong frame when it fails.

### Phase 3: Evidence Collection (Non-Destructive)

Gather evidence WITHOUT running experiments. Read logs, check metrics, inspect source code, query databases (read-only). Do NOT restart services, change config, or deploy fixes.

#### Step 0 (MANDATORY): Verify assumptions before collecting any other evidence

Before reading a single log line, take the assumption table from Phase 2 and check every assumption that can be verified. Rank them by urgency: lowest-probability assumptions first, highest-impact assumptions first (especially assumptions that appear across multiple hypotheses).

Verification can use any method — this is not limited to log reading:
- **Web search or documentation lookup** — "Does this library version support this API?" "Did this interface change?" Check the changelog, the official docs, a migration guide. This verifies assumptions about external dependencies in minutes.
- **Log inspection** — read existing logs without changing system state
- **Source code inspection** — read the deployed version, not just what's on main
- **Instrumentation** — add a log line that directly observes the assumed behavior
- **Experiment** — run a targeted test that confirms or denies the assumption

For each assumption, either:
- **Confirm it** — update its probability toward 1.0, recompute the hypothesis ceiling
- **Refute it** — hypothesis ceiling collapses; mark the hypothesis as eliminated or severely downgraded
- **Cannot check yet** — note why, move to the next assumption

**Ordering assumptions first:** Check all ordering-type assumptions by cross-referencing the established timeline (Phase 1). If a hypothesis requires "A before B" and the timeline shows "B before A," the hypothesis is eliminated at no cost. Add newly confirmed events to the established timeline as you go — each confirmed event may immediately eliminate hypotheses that required a different ordering.

This step finds the cheapest kills. A hypothesis whose key assumption is false takes no experiment to eliminate — a documentation lookup or a log observation is enough.

#### Ongoing evidence collection

Organize all evidence as numbered items:

```
### E1: {Evidence Title}
- What was observed (exact values, timestamps, quotes from logs)
- What it implies for each hypothesis and assumption
- Confidence in this evidence (high/medium/low — logs can lie)
```

**Evidence sources (priority order):**
1. **Assumption verification** (Step 0 above — always first)
2. Application logs (timestamped, grep for error signatures)
3. System metrics (CPU, memory, disk, network — check the specific process, not just the host)
4. Source code (the code that's actually running, not what's on main — check deployed version). **Read it through the code graph, not grep.** If a codebase-memory / code-index MCP is available (and per Phase 0 you have made sure it is), use it as the *primary* way to understand code: query the graph for a symbol's definition, trace callers and callees, follow data flow, list what a function reaches. `search_graph` / `trace_path` / `query_graph` / `get_code_snippet` answer "who calls this?", "what does this reach?", and "where is the shared resource mutated?" — the exact questions an investigation asks — far better than text search. `grep`/`glob` remain the right tool for non-code text (configs, logs, comments) and as a fallback when the index is thin or unbuilt; they are not the tool for reasoning about code structure and relationships. Prefer the graph; fall back to grep, not the other way around.
5. Other instances/agents (is this instance-specific or systemic?)
6. External dependencies (upstream services, DNS, CDN, reverse proxy logs)
7. Configuration state (what config is actually loaded, not what's in the file)
8. **Web research (MANDATORY)** — Search for every error message, library name, and platform behavior in the hypothesis list. Launch background research agents in parallel — one per hypothesis category. Search for: known bugs, correct configuration, API documentation, community reports, Stack Overflow, GitHub issues. Document findings as evidence items. This is not optional — it runs at this phase AND repeats after every subsequent phase that produces new information.

**Web research protocol:**
- Search for exact error messages in quotes
- Search for library name + version + symptom
- Search for platform + behavior (e.g., "iOS AVAudioEngine echo cancellation")
- Search for configuration keywords (e.g., "coturn denied-peer-ip AWS")
- Launch multiple background agents for parallel research
- Document each finding as an evidence item (E-prefixed) with source URL

**Critical rule:** If any evidence would be destroyed by restarting the service, capture it FIRST. Pipe logs, screenshot dashboards, dump process state. The investigation file preserves what the restart erases.

### Phase 4: Revised Hypotheses (Post-Evidence)

Update both tables — assumptions first, then hypotheses. Assumption updates propagate automatically to hypothesis ceilings.

#### Revised assumption table

| # | Assumption | Initial P | Revised P | Evidence |
|---|-----------|-----------|-----------|---------|
| A1 | ... | 80% | **95%** | E2 confirms directly |
| A2 | ... | 60% | **10%** | E4 refutes — not what we thought |

#### Revised hypothesis table

| # | Hypothesis | Initial P | Revised P | P ceiling | Key Evidence |
|---|-----------|-----------|-----------|-----------|-------------|
| H1 | ... | 25% | **3%** | A1×A2 = 9.5% | A2 refuted (E4) |
| H9 | **New: root cause** | N/A | 70% | A3×A4 = 81% | E2, E5, E6 suggest |

**Web research checkpoint (MANDATORY):** Before proceeding to any next phase, search the web for the leading hypotheses. The answer may already exist — a documented bug, a configuration guide, a WWDC session, a GitHub issue. Searching takes minutes; experimenting takes hours.

If one hypothesis exceeds 80%, proceed directly to Phase 6 (Blame Assignment) — experimentation is unnecessary when passive evidence is conclusive. If the leading hypothesis is between 50-80%, design experiments to confirm or refute it. If no hypothesis exceeds 50%, more evidence is needed before experimenting — return to Phase 3.

### Phase 5: Experimentation

Design and execute targeted experiments to test the leading hypotheses. Unlike Phase 3 (passive observation), experiments actively change system state. Each experiment must be:

- **Targeted** — tests exactly one hypothesis
- **Reversible** — system returns to its prior state afterward (or the experiment is run on a replica)
- **Documented** — predicted outcome recorded BEFORE execution, actual outcome recorded AFTER

```
### X1: {Experiment Title}
- **Tests:** H{N} ({hypothesis name})
- **Predicted outcome if H{N} is true:** {what should happen}
- **Predicted outcome if H{N} is false:** {what should happen instead}
- **Procedure:** {exact steps}
- **Actual outcome:** {what happened}
- **Conclusion:** {confirms/refutes H{N}, revised probability}
```

**Experiment design principles:**

1. **Predict before executing.** Write the expected outcome for both "hypothesis true" and "hypothesis false" BEFORE running the experiment. This prevents post-hoc rationalization — interpreting ambiguous results as confirming whatever was already believed.
2. **Smallest possible intervention.** A restart clears all state. A config change affects one variable. Prefer the smallest intervention that distinguishes the hypotheses.
3. **Preserve the crime scene.** If the experiment will destroy evidence (e.g., restarting a service clears in-memory state), capture all remaining evidence from Phase 3 first. Document what was lost.
4. **One variable at a time.** Changing two things simultaneously makes it impossible to attribute the result. If an experiment requires multiple changes, break it into sequential sub-experiments.
5. **Replicas over production.** When possible, reproduce the issue in a test environment and experiment there. When production experimentation is unavoidable, prefer read-only probes (health checks, test requests to non-critical endpoints) over mutations.

**Three-strike check (MANDATORY):** Before designing each experiment, count how many previous experiments targeted the same hypothesis category. If the count is 3 or more, STOP — do not design another experiment in this category. Instead, execute the Category Pivot Protocol from `references/assumption-lock-in.md`. Write in the investigation file: "Three strikes reached for category [X]. Pivoting." Then generate hypotheses from at least 2 new categories before continuing.

**Common experiment types:**
- **Restart test** — does restarting the service clear the symptom? (Tests: transient state accumulation vs. persistent misconfiguration)
- **Isolation test** — does the issue reproduce with one client? (Tests: load-dependent vs. client-specific)
- **Config change** — does adjusting a limit/timeout resolve the symptom? (Tests: specific config hypothesis)
- **Curl probe** — does a direct HTTP request to the endpoint succeed? (Tests: client-side vs. server-side)
- **Reproduction** — can the issue be triggered in a controlled environment? (Tests: environmental vs. code-level)

### Phase 6: Final Hypothesis Revision

Update the hypothesis table a second time with post-experiment probabilities. The experiment results should push the leading hypothesis above 90% or eliminate it and promote an alternative.

| # | Hypothesis | Initial P | Post-Evidence P | Post-Experiment P | Key Experiment |
|---|-----------|-----------|-----------------|-------------------|---------------|
| H1 | ... | 25% | 5% | 2% | X1 refutes |
| H9 | **Root cause** | N/A | 70% | **95%** | X2 confirms |

**Web research checkpoint (MANDATORY):** After each experiment, search the web for the specific behavior observed. Unexpected results often match known bugs or documented platform behavior. An experiment result of "still broken after fix" is a search query — someone else likely hit the same wall.

**Decision gate:** Proceed to Phase 7 (Blame Assignment) only when one hypothesis exceeds 90%. If experimentation was inconclusive, design additional experiments or gather more evidence. Do not assign blame to an unconfirmed hypothesis — a wrong root cause leads to wrong fixes.

### Phase 7: Blame Assignment (Three Levels)

This is the diagnostic core. Assign blame at three ascending levels of abstraction:

**Level 1: Lines of Code** — Which specific lines are responsible? Create a table with file:line, the code, why it's wrong, and severity (Critical/High/Medium/Low).

**Level 2: Anti-Patterns** — What programming pattern made those lines possible? Common categories:

See `references/anti-pattern-catalog.md` for the 8 standard anti-patterns and their search strategies.

**Level 3: Coding Style** — What development practice allowed those anti-patterns to persist? Examples: no failure-path tests, functions returning `()` instead of `Result`, debug logging to temp files, health endpoints that lie.

The three levels produce three different fix scopes:
- Level 1 → fix the specific lines (minutes to hours)
- Level 2 → search for and fix all instances of the anti-pattern (hours to days)
- Level 3 → change development practices to prevent recurrence (ongoing)

### Phase 8: Immediate Fix

Fix the code-level blame (Level 1) immediately. The investigation file documents what to change.

- Write the fix
- Write tests for the failure path (not just the happy path — the failure path is what broke)
- Commit with a reference to the investigation file
- Verify the fix resolves the original symptom

The fix should be minimal and targeted — address exactly the lines identified in Level 1 blame. Broader improvements belong in the remediation plan (Phase 10), not the incident fix.

### Phase 9: Anti-Pattern Search

Search for the anti-pattern (Level 2) across the entire codebase. **The code graph is the right instrument for this phase, not grep.** An anti-pattern is a *structural* shape — "a shared resource mutated by a per-item handler," "a function that closes a connection without an already-closed guard," "every caller of X that forgets to also call Y." Those are graph questions, and a codebase-memory / code-index MCP answers them directly and completely, where grep answers them partially and noisily. Grep finds the *string* `closeSocket`; the graph finds every *caller* of it, what each caller reaches, and which ones skip the paired guard — which is the actual finding. Use the graph to enumerate the class; use grep only to catch textual variants the index might miss (comments, dynamically-built names) and as a fallback when the project isn't indexed. If the index isn't built yet, build it (Phase 0) before running the search — do not fall back to grep by default.

Launch parallel search agents — one per anti-pattern identified. Each agent:

1. Uses the code graph FIRST (callers/callees, data-flow, symbol-by-shape queries), with grep/glob from `references/anti-pattern-catalog.md` as the textual fallback
2. Classifies each finding by severity (Critical/High/Medium/Low)
3. Returns a structured findings list

**Make the sub-agents use the graph.** A known failure mode is that spawned search agents default to grep even when a code-index MCP is available — text search is the path of least resistance. Explicitly instruct each agent to use the codebase-memory MCP (`search_graph`, `trace_path`, `query_graph`, `get_code_snippet`) as its primary tool and to treat grep as the fallback. Give it the graph queries, not just grep patterns. Verify at least one finding was reached through the graph, not text search.

Merge results into a unified audit report. See `references/audit-report-template.md` for the format.

This phase is distinct from the fix because it operates at a different scope — the fix addresses the specific incident, the search addresses the class of bug. The search often reveals 10-100x more findings than the original incident.

### Phase 10: Comprehensive Remediation Plan

Map each coding style deficiency (Level 3) to a systemic fix that addresses the entire category, not just individual findings. The plan should:

1. **Prioritize by severity** — Critical findings first, accepted debt last
2. **Phase by effort** — immediate (today), this week, next sprint, accepted debt
3. **Account for codebase evolution** — later phases assume earlier phases are complete
4. **Include "what NOT to do"** — prevent over-engineering and lateral moves

Structure the plan as:
- Phase 1: Stop the bleeding (Critical fixes, <1 day)
- Phase 2: Structural hardening (High fixes + preventive tooling, ~3 days)
- Phase 3: Architectural (systemic redesign, ~1 week)
- Accepted debt: Findings that are correct for context

## Running Anti-Pattern Searches

Launch parallel Task agents (one per anti-pattern) using the `Explore` subagent type. Provide each agent with:
- The anti-pattern name and definition
- **Code-graph queries first** (symbols/callers/callees/data-flow to enumerate the structural class), with grep patterns + file globs as the textual fallback
- Classification criteria (what makes a finding Critical vs. Low)
- The codebase root path and the indexed project name

Example prompt for an agent:

```
Search the codebase at {path} (indexed project: {project}) for Anti-Pattern: {name}.
{definition}

Use the codebase-memory MCP as your PRIMARY tool — do NOT default to grep:
- search_graph (by name/label/shape) to find candidate symbols
- trace_path (callers/callees/data_flow) to find every site of the class and
  check whether each one has the paired guard/cleanup the pattern requires
- query_graph (Cypher) for multi-hop / "every caller of X that lacks Y" patterns
- get_code_snippet to read the exact source before classifying
Fall back to these grep/glob queries only for textual variants the graph misses:
{queries}

Read the actual source of each candidate before reporting — do not classify from
a name match alone.

For each finding, classify severity:
- Critical: can cause production failure
- High: will cause issues under load or over time
- Medium: suboptimal but not dangerous
- Low: accepted debt, document reasoning

Return a markdown table: | # | Severity | File:Line | Issue | (note which were found via the graph)
```

## Additional Resources

### Reference Files

- **`references/investigation-template.md`** — Complete investigation file template with all sections pre-formatted. Copy this to start a new investigation.
- **`references/anti-pattern-catalog.md`** — The 8 standard anti-patterns with definitions, search strategies (grep queries), and classification criteria.
- **`references/audit-report-template.md`** — Template for the unified audit report that merges findings from parallel search agents.
- **`references/assumption-lock-in.md`** — The Category Pivot Protocol. How to recognize when an investigation is stuck in the wrong hypothesis category, the three-strike rule, and the real-world iOS terminal investigation that motivated this guidance.

### Example Files

- **`examples/session-cache-oom-investigation.md`** — A full worked walkthrough on a synthetic OOM incident (an unbounded memoization cache), exercising every phase including Phase 0 tool-readiness, the Phase-2 maximum-pain pass, and codebase-memory-first search in Phase 9.
