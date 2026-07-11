# Incident Investigation

A structured, hypothesis-driven methodology for debugging production incidents — the kind where the symptom is separated from the cause by **time, scale, or indirection**, so a stack trace alone won't get you there.

It's packaged as an agent **skill** (works with Claude Code and any harness that loads Markdown skills), but the methodology stands on its own as a checklist a human can follow.

## What it gives you

A **10-phase investigation** that produces a single Markdown artifact as it goes — the investigation file *is* the deliverable:

1. Open the investigation (symptom, impact, an established timeline of *confirmed* events)
   - *(Phase 1.5)* Calibrate your generative prior — before writing a single hypothesis, set the expectation out loud: most will be wrong, the real cause is probably unlisted.
2. Initial hypotheses — each with a probability, its load-bearing assumptions, and a **mandatory "none of the above" row** carrying real mass
3. Evidence collection (non-destructive: read before you touch)
4. Revised hypotheses (probabilities move as evidence lands)
5. Experimentation (targeted, reversible, predicted-before-run)
6. Final hypothesis revision
7. **Blame at three levels** — the line, the anti-pattern behind the line, the practice that let the anti-pattern survive
8. Immediate fix (with a *failure-path* test)
9. Anti-pattern search — fix the instance, then find the whole class
10. Comprehensive remediation plan (three scopes, prioritized, with "what NOT to do")

Plus **Phase 0**: line up your tools *before* you investigate — degraded instruments manufacture false evidence.

## Four default actions — initiate these, don't wait to be asked

The methodology opens with four moves that are almost always available and almost always useful, and that investigators reliably *fail to start* on their own. The skill tells you to start them anyway:

1. **Short user interview** — "When did you first notice it? What changed on your end? Has it happened before?" A two-minute interview eliminates more hypotheses than an hour of log reading, because it directly verifies the assumptions your hypotheses rest on.
2. **Web searches** — search every error string, library name, and platform behavior *early and in parallel*. Known bugs, version gotchas, and documented behaviors are usually already written down. (This isn't a one-time step — web research is **mandatory at every phase transition**, with explicit checkpoints after evidence, after each experiment, and during the anti-pattern search.)
3. **Instrumentation** — if the existing logging doesn't show you what you need, *add a log line that does*. Don't investigate blind when you can make the system visible; a targeted probe can confirm or kill a hypothesis in minutes.
4. **Look at it, then validate the screenshot instrument** — rendered evidence catches DOM/projection lies, but unchanged pixels are not clearance until target identity and freshness are established. A screenshot is a strong positive oracle and can be a high-false-negative silence on dormant/occluded canvas apps; triangulate with an independent application model or export.

## How the rigor actually works

Underneath the phases is a small set of mechanics that make the probabilities honest rather than decorative:

- **Assumptions carry probabilities, and they *cap* the hypothesis.** Every hypothesis rests on load-bearing assumptions, each with its own probability. A hypothesis can't credibly exceed the *product* of its assumptions — if H1 needs A1 (80%) and A2 (60%), H1 is capped at 48% however airtight the rest of the story. So the cheapest kills come from verifying a shared, low-probability assumption, not from running experiments.
- **The established timeline is crime-scene ground truth.** It holds only *confirmed* events. When a hypothesis's required ordering conflicts with the timeline, the hypothesis is eliminated *with no experiment*. And a user's recollection of "when it started" is a hypothesis about the past, not a confirmed event — it's labeled unverified and given a probability below 1.0, because subjective certainty and accuracy are different things.
- **An inversion calibration check keeps estimates real.** After assigning P(A), independently estimate P(not A); if they don't sum to ~100%, you haven't actually reasoned about it. This catches the compound-assumption trap where the alternatives you ignored secretly hold most of the mass.
- **Reason about code through the graph, not grep.** A code-index (callers, callees, data-flow) answers the questions an investigation actually asks — "who calls this?", "which caller lacks the paired guard?", "where is the shared resource mutated?" — completely, where text search answers them partially and noisily. Grep is the fallback for non-code text and unindexed projects, not the default. This matters most in the **anti-pattern search** (Phase 9), where an anti-pattern is a *structural shape*, and spawned search agents must be told explicitly to use the graph or they default to grep.

## The idea it's built around: the maximum-pain principle

> Of the competing explanations, the one that is most **painful to accept** — the one that implicates your own recent change, your own model, your own tooling — is disproportionately likely to be true. Test it first.

This isn't self-flagellation; it's calibration. Three reasons it holds:

- **Selection bias** — the comfortable explanations were already caught by tests and review; what survives to become a mystery is the class you have a motive to skip.
- **Motivated reasoning** — you're not a neutral estimator of "did *my* change break this?"; the same thing that makes a hypothesis painful is what makes you under-rate its probability. The pain marks where your estimate is biased, and in which direction.
- **Base rates** — most incidents are a recent change by someone with access, not a dormant bug in a mature dependency. Painful *and* common.

Truth and pain are positively correlated in debugging. Sorting hypotheses by descending pain converges faster than sorting by descending comfort — which is the sort your unaided intuition performs.

The full argument, and how to turn it into a probability "pain pass," is in [`SKILL.md`](SKILL.md).

## The deeper idea: you cannot think your way to the answer

Maximum-pain tells you how to *rank* the hypotheses you have. But a harder truth sits underneath it — the one most investigations quietly ignore: **the true cause is usually not on your list at all, and no amount of further thinking will put it there.**

- **Why your first-pass list misses (by construction).** On a system you built, every failure you could *imagine* you already defended against while building it — you guarded the race you foresaw, validated the input you pictured. So the bugs that survive to become incidents are, almost by definition, the ones you *didn't* imagine. Your hypothesis list is drawn from the very same imagination that already eliminated everything it could think of. That is why measured first-pass hit-rates are a small minority — often single digits, and **for no one, human or agent, anywhere near half.**

- **So an experiment isn't a referee — it's a generator.** Its job is not merely to pick the winner among your named hypotheses. It is to *manufacture* the hypothesis you could not think of, by putting you in contact with data your imagination did not predict. When your whole list is dying one hypothesis at a time, that is not failure — it is the signal to go get more data, because the answer lives in your blind spot and only fresh observation reaches it.

- **Surprise is the compass — so celebrate it.** Because the cause is outside your imagination, the felt-sense of a *healthy* investigation is not "aha, confirmed" — it is *"wait, what? that's not what I expected."* Confirmation keeps you inside your own model, which is exactly where the cause is not; surprise is the sound of the blind spot cracking open. The skill makes this a ritual: **mandatory champagne (🍾) every time you're shocked** — mark it in the file and stop to mine it, instead of the untrained reflex of explaining it away.

- **Probabilities are never fully objective.** The deepest consequence: your hypothesis probabilities sum to 1 over the hypotheses *you were able to generate*, so every one is secretly conditioned on your own imagination. The "none of the above" mass isn't a fact about the world — it's a measurement of *your own* generative limitation, a hypothesis about *you*. You can't factor the observer out of the estimate; the events outside you and your capacity to conceive them are entangled. Objectivity isn't reasoned to from the inside — it is *earned* by letting the world push back.

These live in `SKILL.md` as Core Principles #11–#12, and they reshape the whole method: hold your list at a low prior, keep a mandatory mass-carrying "none of the above" row, and spend your experiments trying to *reveal* a new cause rather than merely referee the named few.

## Other load-bearing principles

- **Hypotheses span multiple categories** — a monoculture of hypotheses is the biggest risk; three strikes in one category triggers a mandatory pivot (`references/assumption-lock-in.md`).
- **Fix the instance, then the class** — the bug that caused this incident almost always exists elsewhere; Phase 9 searches the whole codebase for the anti-pattern, often finding 10-100× more instances than the original.
- **Attractor hypotheses need a gate, not a caution** — some exculpatory explanations ("the input wasn't trusted," "the platform blocked it") are *attractors*: whatever the real failure, your reasoning keeps sliding into the same convenient story. A note-to-self doesn't defeat them (a quiet caution loses the competition for attention to the loud attractor); a **mandatory step at the point of temptation** does — e.g. a required cheap A/B before you build the privileged escape hatch.

## Layout

The repo root **is** the skill — so it drops straight into any framework's skills
directory with no wrapper:

```
SKILL.md                       # the methodology (start here)
references/
├── anti-pattern-catalog.md    # 8 standard anti-patterns + how to find them
├── assumption-lock-in.md      # the category-pivot protocol (three strikes)
├── audit-report-template.md   # merge parallel search findings
└── investigation-template.md  # copy this to start a new investigation
examples/
└── session-cache-oom-investigation.md   # a full worked walkthrough
```

(`README.md` and `LICENSE` sit alongside; skill loaders read `SKILL.md` and ignore the rest.)

## Using it

Because the repo root is the skill, including it is trivial in any agent
framework that loads Markdown skills:

- **As a git submodule (recommended):** add this repo directly at the skill's
  path in your project — e.g.
  `git submodule add https://github.com/yaniv256/incident-investigation.git <your-skills-dir>/incident-investigation`.
  `SKILL.md` lands exactly where the loader expects it, with no symlink or
  per-project wrapper.
- **As a copy:** drop the repo's contents into `<your-skills-dir>/incident-investigation/`.
- **By hand:** just open `SKILL.md` and follow the methodology.

The skill activates when you ask to investigate a bug, do a root-cause analysis, or run a postmortem.

## License

MIT — see [LICENSE](LICENSE).
