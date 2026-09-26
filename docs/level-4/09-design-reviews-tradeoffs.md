---
description: "How to write a system design document engineers will read: structure, requirements, alternatives considered, trade-off tables, decision records, and running reviews."
---

# 09 · Writing Design Docs & Defending Trade-offs

In real engineering organizations, the design of anything significant is written down
before it is built, and other engineers review it. A design doc forces clear thinking
("if I can't explain it, I don't understand it"), surfaces disagreements while they are
cheap to resolve, and leaves a record of *why* the system looks the way it does — which is
the first thing the next engineer will want to know.

## A structure that works

Formats vary between teams; this outline covers what most good design docs contain.

1. **Title, author(s), status, date, reviewers.**
2. **Context and problem.** What is broken or missing, for whom, and why now. Link data.
3. **Goals and non-goals.** Non-goals are as important: they prevent scope creep and
   pointless review debates.
4. **Requirements.** Functional, and non-functional with numbers (scale, latency
   percentiles, availability, RPO/RTO, cost targets).
5. **Estimates.** The back-of-envelope arithmetic and what it implies.
6. **Proposed design.** Architecture diagram, request paths for key operations, data model,
   API, and how it meets each requirement.
7. **Alternatives considered.** At least two, each with why it was not chosen.
8. **Failure modes and operations.** What fails, what users see, how it is detected
   (metrics, alerts), how it recovers. Rollout and rollback plan.
9. **Security and privacy.** Data handled, trust boundaries, access control.
10. **Cost.** Estimated unit cost and the dominant term.
11. **Open questions and risks.**
12. **Milestones.**

## Writing well

- **Lead with the decision.** Busy reviewers read the summary; put the proposal and its key
  trade-off in the first paragraph.
- **Numbers over adjectives.** "Fast" is unreviewable; "p99 under 200 ms at 5,000 req/s" is.
- **Diagrams with labeled arrows** (what flows, sync or async) rather than unlabeled boxes.
- **One idea per section**, short paragraphs, tables for comparisons.
- **Be honest about weaknesses.** Reviewers will find them anyway; naming them first builds
  trust and focuses the discussion.

## Trade-off tables

When comparing alternatives, make the criteria explicit and weight them by the
requirements, not by general preference:

| Criterion (weight) | A: Central rate-limit store | B: Local buckets + periodic sync | C: Per-instance only |
|---|---|---|---|
| Accuracy (high) | Exact | Within ~10% | Poor under uneven load |
| Added latency per request (high) | +1 network round trip | ~0 | 0 |
| Behavior if store fails (medium) | Must fail open or closed | Continues with local limits | Unaffected |
| Complexity (medium) | Low | Medium | Lowest |
| **Decision** | | **Chosen** | |

A table like this turns "I prefer B" into a reviewable argument: someone who disagrees can
point to a specific cell or weight.

## Worked example: a design-doc linter

A small script can catch the most common omissions before a human reviewer spends time.
It checks a Markdown design doc for required sections and a few smells.

```python
# doc_lint.py — check a Markdown design doc for required sections and vague language
import re, sys

REQUIRED = ["goals", "non-goals", "requirements", "estimates", "proposed design",
            "alternatives", "failure modes", "security", "cost", "open questions"]
VAGUE = ["fast", "scalable", "highly available", "robust", "real-time", "seamless"]

def lint(text):
    headings = [h.strip().lower() for h in re.findall(r"^#+\s*(.+)$", text, re.M)]
    problems = [f"missing section: {s}" for s in REQUIRED
                if not any(s in h for h in headings)]
    for word in VAGUE:
        for m in re.finditer(rf"\b{re.escape(word)}\b", text, re.I):
            line = text[: m.start()].count("\n") + 1
            line_text = text.splitlines()[line - 1]
            if not re.search(r"\d", line_text):       # vague claim, no number on that line
                problems.append(f"line {line}: '{word}' without a number")
    return problems

SAMPLE = """# Link service
## Goals
Make redirects fast and highly available.
## Requirements
p99 redirect latency under 50 ms at 10,000 req/s; 99.95% monthly availability.
## Proposed design
...
## Alternatives
...
"""

if __name__ == "__main__":
    text = open(sys.argv[1]).read() if len(sys.argv) > 1 else SAMPLE
    for p in lint(text):
        print(p)
```

On the sample it reports the missing sections (non-goals, estimates, failure modes,
security, cost, open questions) and flags "fast" and "highly available" in the Goals
section, which have no numbers attached — while the Requirements line, which quantifies
them, passes. The tool is crude on purpose; the habits it enforces are the point.

## Architecture decision records (ADRs)

Big design docs are for big changes. For individual decisions made along the way, many
teams keep **ADRs**: short, numbered, immutable records — context, decision, consequences.
"ADR-014: Use cursor pagination for all list endpoints." When a decision is reversed, a new
ADR supersedes the old one rather than editing it, so the history of reasoning survives.

## Running and responding to reviews

- **Send the doc ahead** and ask for written comments first; meetings then resolve
  disagreements instead of reading aloud.
- **Name the decisions you want input on.** "Please focus on the partitioning scheme and
  the failover plan" gets better reviews than "thoughts?".
- **Separate blocking concerns from suggestions.**
- **Respond to every comment**: accept, reject with reasoning, or record as an open
  question. Silence erodes trust.
- **Disagree with evidence**: a load test, a prototype, a calculation. "Let's measure"
  resolves more debates than eloquence.

## How It Actually Works

Design reviews catch problems for a structural reason: the author has the most context but
also the most blind spots — they know what they meant, so they read their own gaps as
filled. Reviewers approach from different experiences (operations, security, the team that
will call the API) and see failure modes the author did not. Writing forces the author to
make implicit assumptions explicit, which is where many problems surface before a reviewer
even reads the doc. The "alternatives considered" section is particularly valuable: it
shows reviewers the design space, prevents the same alternatives from being re-proposed
repeatedly, and makes the chosen trade-off falsifiable ("if our write rate exceeds X, we
should revisit alternative B").

## Common mistakes

- **Design docs written after the code** as documentation, missing the chance for input.
- **No alternatives section**, or strawman alternatives nobody would pick.
- **Unquantified requirements.**
- **Diagrams without a narrative** of request paths.
- **Treating review comments as attacks** rather than free testing.
- **Never updating the doc** when the implementation diverges.

## Exercise

1. Run `doc_lint.py` on the sample, then write a design doc for your news-feed project
   (Level 2) that passes it cleanly.
2. Write a trade-off table for the ID-generation choices in the Level 1 URL shortener,
   with weighted criteria derived from its requirements.
3. Write three ADRs for decisions in your chat-system design (Level 3), including one that
   supersedes an earlier one.
