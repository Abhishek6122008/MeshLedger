# MeshLedger docs

Start here. These docs explain MeshLedger in plain words: what it does, how it is built, how the AI part will work, and the plan to get there.

## The two labels

Every feature in these docs carries one of two labels.

- **Today (v1)**: it is in the code right now, and you can run it.
- **Planned (Day N)**: it is designed but not built yet. Day N is the day of the 30-day plan that builds it. The design can still change on that day. If it does, the doc changes in the same commit.

## No made-up numbers

There are no speed or accuracy numbers here yet. A number goes in only after it has been measured, and it comes with the exact command that measured it.

The only results quoted today are what we *observed* when we poked the v1 server with test packets. Those runs are saved in the workspace devlog.

## Reading order

| # | Doc | Read it to learn |
|---|---|---|
| 1 | [Overview](01-overview.md) | What MeshLedger does, the basement story, and how to try it |
| 2 | [High-level design](02-hld.md) | The big boxes, and how a payment moves through them |
| 3 | [Low-level design](03-lld.md) | Every class, table, endpoint and setting |
| 4 | [Money safety](04-money-safety.md) | How we stop tampering, duplicates, replays and double-spends |
| 5 | [The AI layer](05-ai-layer.md) | How the fraud models and the AI investigator will work |
| 6 | [Lessons from Apache Fineract](06-fineract-lessons.md) | Ideas I learned while contributing to Fineract, and where they go here |
| 7 | [Roadmap](07-roadmap.md) | The full plan, day by day |
| 8 | [Glossary](08-glossary.md) | Every term, in a line or two |

Short on time? Read 1 and 2. For the rest, read the "In one minute" box at the top of each doc.
