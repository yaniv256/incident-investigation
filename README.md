# Incident Investigation

A structured, hypothesis-driven methodology for debugging production incidents — the kind where the symptom is separated from the cause by **time, scale, or indirection**, so a stack trace alone won't get you there.

It's packaged as an agent **skill** (works with Claude Code and any harness that loads Markdown skills), but the methodology stands on its own as a checklist a human can follow.

## What it gives you

A **10-phase investigation** that produces a single Markdown artifact as it goes — the investigation file *is* the deliverable:

1. Open the investigation (symptom, impact, an established timeline of *confirmed* events)
2. Initial hypotheses — each with a probability and its load-bearing assumptions
3. Evidence collection (non-destructive: read before you touch)
4. Revised hypotheses (probabilities move as evidence lands)
5. Experimentation (targeted, reversible, predicted-before-run)
6. Final hypothesis revision
7. **Blame at three levels** — the line, the anti-pattern behind the line, the practice that let the anti-pattern survive
8. Immediate fix (with a *failure-path* test)
9. Anti-pattern search — fix the instance, then find the whole class
10. Comprehensive remediation plan (three scopes, prioritized, with "what NOT to do")

Plus **Phase 0**: line up your tools *before* you investigate — degraded instruments manufacture false evidence.

## The idea it's built around: the maximum-pain principle

> Of the competing explanations, the one that is most **painful to accept** — the one that implicates your own recent change, your own model, your own tooling — is disproportionately likely to be true. Test it first.

This isn't self-flagellation; it's calibration. Three reasons it holds:

- **Selection bias** — the comfortable explanations were already caught by tests and review; what survives to become a mystery is the class you have a motive to skip.
- **Motivated reasoning** — you're not a neutral estimator of "did *my* change break this?"; the same thing that makes a hypothesis painful is what makes you under-rate its probability. The pain marks where your estimate is biased, and in which direction.
- **Base rates** — most incidents are a recent change by someone with access, not a dormant bug in a mature dependency. Painful *and* common.

Truth and pain are positively correlated in debugging. Sorting hypotheses by descending pain converges faster than sorting by descending comfort — which is the sort your unaided intuition performs.

The full argument, and how to turn it into a probability "pain pass," is in [`skills/incident-investigation/SKILL.md`](skills/incident-investigation/SKILL.md).

## Other load-bearing principles

- **Hypotheses span multiple categories** — a monoculture of hypotheses is the biggest risk; three strikes in one category triggers a mandatory pivot.
- **The code graph, not grep** — reason about code through a code-index (callers, callees, data-flow), which answers "who calls this?" and "which of these lacks the paired guard?" — the questions an investigation actually asks. Grep is the fallback, not the default.
- **Fix the instance, then the class** — the bug that caused this incident almost always exists elsewhere.

## Layout

```
skills/incident-investigation/
├── SKILL.md                       # the methodology (start here)
├── references/
│   ├── anti-pattern-catalog.md    # 8 standard anti-patterns + how to find them
│   ├── assumption-lock-in.md      # the category-pivot protocol (three strikes)
│   ├── audit-report-template.md   # merge parallel search findings
│   └── investigation-template.md  # copy this to start a new investigation
└── examples/
    └── session-cache-oom-investigation.md   # a full worked walkthrough
```

## Using it as a Claude Code skill

Drop `skills/incident-investigation/` into your project's skills directory (e.g. `.claude/skills/`), or add this repo as a plugin. The skill activates when you ask to investigate a bug, do a root-cause analysis, or run a postmortem. Or just open `SKILL.md` and follow it by hand.

## License

MIT — see [LICENSE](LICENSE).
