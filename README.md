# Over-reliance on Specification: A Speckit Experiment

There is no holding back Generative AI in coding. It is a really powerful tool.

To set the context: I have been using these tools right from the ChatGPT release —
first as a chat assistant and Google Search / StackOverflow replacement, then through
the coding-assistant generation of Copilot, Google Gemini (Antigravity), Cursor, and
Claude Code. IDEs for semi-autonomous software development, Claude Code for fully
autonomous.

I have been a professional programmer and software engineer for over 20 years,
working from C++ to Python through a hundred frameworks and paradigms that were each,
in their day, *the* hype: Object Oriented, Distributed Objects (Microsoft COM),
remote procedural invocation (CORBA), Service Oriented Architecture (SOAP), SEDA
(Apache Camel), Message-Driven Architecture (JMS, ActiveMQ), Microservices (Docker,
Kubernetes), gRPC, SQL, NoSQL — you name it. Hundreds of thousands of lines of code
written and deployed.

Not that I'm a dinosaur yet. I've embraced Generative AI in my work and used it
extensively in semi-autonomous mode on my last project, and Claude Code in fully
autonomous mode (with various guardrails — all English instructions, in the end) on
several hobby projects.

At best I am an average programmer. But when people who have never written a line of
code show up with the recommendation that Specs are the silver bullet, my mind races
back to *The Mythical Man-Month* — **there are no silver bullets in software
development**.

When I started, Rational Rose was the most hyped product on the market and UML was
the silver bullet of its decade: draw rectangular squiggly diagrams and Rose will
convert them into code. I knew it was a fad and never used the tool or the diagrams.
Fortunately it died out very fast as others figured it out too.

When the current hype over specification-driven development started, I was thinking
the same thing. Dead things are walking again. Test-First Development, TDD, and BDD —
the irony today is that the same crowd that pushed back when *they* had to write the
tests now mandates them when it's a prompt instructing the poor LLM to do it ("or I
am going to never ever MERGE your PR!" — I have seen prompts like this meant for
agents in real repositories).

So here is an experiment. To be clear: what these frameworks do is genuinely good.
I'm going to keep using Speckit and similar in future projects — they cut out the
boilerplate prompting and impose real structure on vibe coding. Nothing wrong with
that. The problem is when this lands in the hands of people who think it is *the
answer* to Generative AI in software development.

The problem deep down is not Speckit. **It is the decision one makes between fully
autonomous or semi-autonomous development.**

> **From experience, I believe it is better to do the core of analysis, design, and
> structure yourself and use AI only to fill in the gaps.**

I remember, long back, that I always did better when I resisted the urge to start
implementing immediately — when I'd walk out of the workplace, look down the window,
and plan, and plan, and then come back and code very little.

Somehow, even with all the upfront planning Speckit and similar frameworks provide,
the planning and thought and exploration — and the resulting gems of design, ideas,
and craft that used to take weeks and months — compress down to a few minutes or
hours. There is no design finesse, no beauty in the design. It is straight surgical
code written to implement the user story.

The problem is that certain key elements of system design — the ones that set a
product apart, or give it high reliability, performance, scalability, or easy
debugging — are not there. This is okay for a lot of software work. It is usually
not okay when you are delivering a *software product* (as opposed to internal or
glue software): telecom management systems, analysis systems, flight-control
systems, real-time systems.

All this is rant — one person's view among a thousand louder, more polished voices.

So to get some credibility back, here is what the experiment actually showed.

It is strange that people will hope for and believe in anything that seems like it
could solve their problems. This is the hype cycle, until they get disillusioned and
reject everything — even the good parts.

Speckit, OpenSpec, `obra/superpowers`, and many others like them are great. But none
of them is the one silver bullet that is going to solve all your problems. None of
them, at least the open-source ones, claim to be. It is essential to stay grounded.

Tomorrow, if AGI is released, all that I wrote here will be obsolete. But until then,
there is a goal, a personality, a hidden artist in all good programmers that is not
in even the latest AI models. That one deficiency makes fully autonomous SW
development a sub-par alternative to what a motivated human programmer can do.

Until then.

> Notes from a single-session experiment running the Speckit framework end-to-end on a
> realistic Python module (US elevation profile over USGS 3DEP NED tiles), then
> stress-testing the resulting plan with a domain-driven alternative.
>
> All quoted excerpts cite line numbers in
> [`speckit_user_claude.txt`](./speckit_user_claude.txt) (the recorded transcript),
> [`.specify/memory/constitution.md`](./.specify/memory/constitution.md),
> [`specs/001-us-elevation-profile/spec.md`](./specs/001-us-elevation-profile/spec.md),
> and [`specs/001-us-elevation-profile/plan.md`](./specs/001-us-elevation-profile/plan.md).
> The references are reproducible — you can open the file at the line number and read
> the surrounding context yourself.

## Tools and Environment

- `uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@v0.8.9`
- Claude Code (latest) with the Claude Opus 4.7 model

Followed the phases outlined in the Speckit documentation
(<https://github.com/github/spec-kit>): Init → Constitution → Specify → Clarify → Plan.

## Premise


A specification framework — even a thoughtful one like Speckit — is only as good as
the assumptions it bakes in. When the spec is produced by an external agent rather
than by the engineer who will build (and live with) the system, two things go wrong:

1. The agent locks in **measurable outcomes** before the architectural drivers are
   known. Those outcomes then act as gravitational pull on later phases, rejecting
   superior designs because they "violate the spec".
2. The agent defaults to **path-of-least-resistance stacks** that pass the
   constitution gates trivially, rather than stacks an experienced practitioner
   would reach for after a couple of Google searches and pattern recall.

The result is a process that *looks* rigorous (numbered FRs, SCs, gates, Sync Impact
Reports) but quietly steers away from the better engineering choice. A human review
step is supposed to catch this. In practice, the volume of generated artefact is
exactly what makes the review step easy to skip.

What follows is a phase-by-phase walkthrough of where the process worked, where it
went sideways, and where the better answer was always one Google search away.

---

## Phase 1 — `/speckit.constitution`

**Transcript:** [`speckit_user_claude.txt:10`](./speckit_user_claude.txt) — user input.

```
/speckit.constitution Create principles focused on code quality, test driven development
consistency, and performance requirements
```

The agent produced [`.specify/memory/constitution.md`](./.specify/memory/constitution.md)
with four non-negotiable principles (Code Quality, TDD, Consistency, Performance).
The principles themselves are reasonable. **Two of them quietly load the dice for
later phases:**

### Principle III — Consistency

> "New dependencies MUST be justified in writing: problem solved, alternatives
> considered, license verified. Adding a dependency that duplicates an existing one's
> purpose is forbidden." — `constitution.md:106`

This is sensible policy in isolation. But it creates an asymmetric cost: **the
stdlib option always wins ties** because it costs zero justification entries.
We will see this become decisive in Phase 4.

### Principle IV — Performance Requirements

> "Every feature plan MUST declare measurable performance budgets in `plan.md` …
> A regression exceeding 5% on a declared budget MUST block merge until either fixed
> or formally re-baselined with documented justification." — `constitution.md:119–126`

Also sensible. But the budget is declared *before* the architecture is chosen.
Once a number is on paper, it becomes the criterion against which all future
architectures are judged — even when the number was a guess.

**Phase verdict:** Good. The constitution itself is fine. The hidden tax — that
stdlib beats deps and that early-declared budgets harden into gates — is invisible
at this stage.

---

## Phase 2 — `/speckit.specify`

**Transcript:** [`speckit_user_claude.txt:496–497`](./speckit_user_claude.txt) — user input.

```
/speckit.specify I want a module in python to store US elevation data files and between
two points of latitude, longitude - of receiver and transmitter
```

A single sentence. No mention of scale, format, accuracy, latency, deployment
context, or interactivity. The kind of thing a teammate would clarify in chat before
opening a doc.

The agent produced a spec anyway. From the report at
[`speckit_user_claude.txt:1046–1054`](./speckit_user_claude.txt):

> "All 16 quality checks pass on the first iteration. No `[NEEDS CLARIFICATION]`
> markers remain. Three scope decisions were resolved as **Assumptions** rather than
> open questions:
>
> | Decision | Default chosen |
> |---|---|
> | Coverage area | Continental US (CONUS) only; Alaska/Hawaii/territories opt-in |
> | Data acquisition | User supplies files; no auto-download in v1 |
> | RF analysis scope | Elevation profile only; LOS/Fresnel/link-budget deferred |"

Two of those three assumptions turned out to be **wrong**. The first session's
spec assumed CONUS-only and a roughly hobby-scale dataset. The real target
(revealed in Phase 3) is full CONUS + Hawaii + Puerto Rico + Alaska,
~1,756 NED tiles, ~23 billion samples, ≈ 180 GB raw — a different
architectural problem.

**Phase verdict:** The agent silently converted ambiguity into "assumptions" and
declared the spec complete. The 16 checklist boxes were green, the FRs were
testable, and the architectural drivers were still missing. This is the
specification-fragility failure mode: the artefact looks done because every
template slot is filled, not because the engineering question is answered.

---

## Phase 3 — `/speckit.clarify`

**Transcript:** [`speckit_user_claude.txt:1083`](./speckit_user_claude.txt) onward.

This is the phase that actually worked.

### What clarify caught

[`speckit_user_claude.txt:1403–1406`](./speckit_user_claude.txt):

> "I scanned the spec against the clarification taxonomy. … The genuinely unresolved
> high-impact items are: dataset scale, supported source formats, and the performance
> budget for paths beyond 100 km."

Three questions, each driving a real architectural lever. The dataset-scale answer
([`speckit_user_claude.txt:1440–1442`](./speckit_user_claude.txt)) reset the entire
problem framing:

> User: "it is C full CONUS at native high resolution and also US territories like
> Puerto Rico, Hawaii. There are about 1756 NED files making a total 23 billion data
> points. Each data point is a 64 bit float."

A second clarify session then pinned tile-index persistence
([`speckit_user_claude.txt:2587–2603`](./speckit_user_claude.txt)) — and this is
where the framework's gravity began to assert itself. The agent presented three
options:

| Option | What's persisted |
|---|---|
| A | Nothing — rebuild every process start |
| B | Small metadata-index sidecar only (Recommended) |
| C | Metadata index + cached decoded tile payload on disk |

The user picked B. **B is reasonable, but it implicitly closes the door on a class
of architectures — including the one we'll see in Phase 5.** The "cached decoded
payload" framing of option C made it sound like a heavyweight extra; in reality, a
single transcoded Zarr store is roughly that same idea, but it is a different
*product shape*, not an optional accelerator. The clarify menu didn't present that
shape as a peer.

This is the most subtle failure in the framework: a clarify question's option
*menu* is itself a design decision, made by the agent, without the user being
told it is one.

**Phase verdict:** Up to and including clarify, the framework was net-positive.
The questions were sharp; the dataset-scale answer in particular saved hours of
wrong-direction work. The hidden cost is that menus phrased by the agent prune
the design space silently. The user, here, did not know that picking B closed
out the Zarr-transcode option later.

> *Aside: this also gave the agent room to deduce some performance assumptions
> directly from the constitution — e.g., "5 min cold registration" — as if those
> numbers had been independently engineered. They had not. They were anchored by
> a feel for what "fast enough" sounded like for a clarify menu.*

---

## Phase 4 — `/speckit.plan`

**Transcript:** [`speckit_user_claude.txt:3015`](./speckit_user_claude.txt) onward.

The plan ([`specs/001-us-elevation-profile/plan.md`](./specs/001-us-elevation-profile/plan.md))
selected the following stack:

> "Index storage │ SQLite via stdlib `sqlite3` + `rtree` virtual table" —
> [`speckit_user_claude.txt:3323`](./speckit_user_claude.txt)
>
> "Sidecar index: SQLite via stdlib `sqlite3` — single-file
> `.us_elevation_profile_index.sqlite` … Stdlib-only (no new dependency); atomic
> writes; supports point-in-polygon queries via straightforward bbox filters" —
> [`plan.md:60–65`](./specs/001-us-elevation-profile/plan.md)

When asked to justify the choice
([`speckit_user_claude.txt:3470–3530`](./speckit_user_claude.txt)), the agent
listed four reasons:

> 1. "Stdlib, zero new dependency. … Under Principle III's 'new dependency must be
>    justified in writing' bar, that's the cheapest possible answer."
> 2. "Atomic single-row UPDATE is what FR-018 literally asks for."
> 3. "R-tree virtual table makes the spatial lookup an indexed query for free."
> 4. "WAL mode is crash-safe."

Re-read reason #1 carefully. **"Stdlib, zero new dependency"** is doing most of
the work. Principle III, recall, says new deps need a justification entry. That
made the comparison structurally unfair: SQLite started ahead of every other
candidate by exactly one paragraph of writing the agent didn't have to produce.

Reasons #2–#4 are real, but they are also things any reasonable storage layer
would deliver (atomic row replace, spatial index, crash safety). The unique
property SQLite owns is the stdlib status. The plan let that one property
dominate.

The rejected alternatives table at [`speckit_user_claude.txt:3509–3519`](./speckit_user_claude.txt)
is revealing:

> | Alternative | Failed on |
> |---|---|
> | JSON sidecar | No atomic single-row update; full read-modify-write |
> | Parquet | Columnar scans are great, but updates rewrite the whole file. Plus a pyarrow runtime dep — fails Principle III's bar |
> | LMDB / BerkeleyDB | Fast KV, but no spatial indexing, harder to inspect, extra dep |
> | DuckDB embedded | Powerful query engine for a workload that's not analytical. Extra dep without a matching benefit |

Notice the phrasing. Three of the four rejections invoke Principle III's
dependency bar. **The constitution did the rejecting; the agent was just the
microphone.**

**Phase verdict:** The plan is internally consistent and would pass review. It is
also a *local optimum*, reached by minimising friction against the constitution
rather than by reasoning about the problem. The lower-friction choice is not the
same as the better choice.

---

## Phase 5 — The human pushback

This is the post-Speckit phase the framework doesn't have a name for. After the
plan was committed
([`speckit_user_claude.txt:3424`](./speckit_user_claude.txt) — commit `4f1b85a`),
the user asked the obvious follow-up:

> "why did you chose Index storage │ SQLite via stdlib sqlite3 + rtree virtual table"
> — [`speckit_user_claude.txt:3465`](./speckit_user_claude.txt)
>
> "why not use zarr as a tensor store" —
> [`speckit_user_claude.txt:3534`](./speckit_user_claude.txt)
>
> "why not just transcode all files which is a one time activity into a zarr store;
> there is no 5 mt starup delay then" —
> [`speckit_user_claude.txt:3609–3610`](./speckit_user_claude.txt)

The agent's first response (Zarr would "blow past SC-008") at
[`speckit_user_claude.txt:3569–3572`](./speckit_user_claude.txt):

> "The transcode cost blows past SC-008. SC-008 budgets 5 minutes for cold
> registration. … reading + decompressing + recompressing + writing through Zarr
> takes 10–30 minutes realistically. You'd either miss the budget or have to declare
> a separate one-time 'first-time setup' phase outside the spec."

Read that closely. **The constitution-derived budget is being used as the reason
a better architecture cannot be considered.** The 5-minute number is itself a
choice — made by the agent in Phase 2/3 with no engineering data behind it — and
it has now hardened into a constraint that disqualifies designs.

When the user pushed back, the agent walked the position back
([`speckit_user_claude.txt:3616–3617`](./speckit_user_claude.txt)):

> "Fair push-back — and I want to be honest about where my prior answer overstated
> the case. Let me re-do the trade-off properly."

And again ([`speckit_user_claude.txt:3656–3660`](./speckit_user_claude.txt)):

> "You're right that I overweighted reason #2 ('blows past SC-008'). That's only true
> if we keep SC-008 framed as 'registration must be fast.' Re-framing registration as
> 'one-time setup that may take 30 min' is a perfectly reasonable spec change."

The Zarr design needs nothing in the spec that isn't already there — except the
willingness to revise an arbitrary number the spec itself produced.

### The R-tree + Zarr alternative (proposed by the user in a follow-up session)

The user's proposal in
[`speckit_user_claude.txt:3704–3747`](./speckit_user_claude.txt) is a clean piece
of geospatial engineering anyone who has worked with NED tiles before would
recognise:

```python
def openZarrDataSet(directoryStore, rows=NED_FILE_HEIGHT, columns=NED_FILE_HEIGHT):
    store = zarr.LMDBStore(directoryStore, readonly=True, lock=False)
    cache = zarr.LRUStoreCache(store, max_size=2**33)
    dataset = zarr.open(
        store=cache,
        shape=[10**3, 10**3, 1, rows, columns],
        chunks=[1, 1, 1, rows, columns],
        fill_value=float("NaN"),
        ...
    )
```

The architecture, in three lines:

1. One-time transcode of GeoTIFFs into a Zarr store keyed by `(lat_idx, lon_idx)`,
   backed by LMDB (mmap'd random reads), wrapped in an `LRUStoreCache`.
2. An R-tree built from the per-tile bounding boxes — *the actual tiles registered,
   not a hardcoded territory grid* — for line-vs-tile intersection on profile
   queries.
3. Query path: pyproj geodesic line → R-tree intersect → chunk keys → Zarr reads
   → bilinear. No SQLite, no rasterio-per-file-open in the hot path, native
   support for the irregular US-territories footprint.

This is **better** than the plan-of-record on at least three dimensions:

- **Right algorithmic tool**: R-tree is the textbook structure for line-segment
  intersection over irregular polygons. Plan-of-record uses R-tree too — but
  inside SQLite, as a virtual table, against a workload that doesn't need any of
  the row-store machinery wrapped around it.
- **Right storage tool**: chunked tensor reads are exactly what an elevation
  profile is. Zarr was built for this access pattern; rasterio + GeoTIFF is the
  legacy shape.
- **Native heterogeneity**: Alaska is 1°×2°, Hawaii is sparse, Guam is a point.
  R-tree handles that without any analytical floor/ceil shortcut that would have
  to be debugged later.

An experienced engineer arrives at this design with two Google searches: "USGS NED
tile lazy access python" and "geospatial chunk store python". The Speckit agent,
constrained by a constitution that penalised new dependencies and a clarify menu
that didn't list this shape as an option, took the SQLite path.

---

## Where each phase actually stood

| Phase | Net value | Failure mode |
|---|---|---|
| `/speckit.constitution` | Positive — principles are sound | Hidden tax: stdlib beats deps; early budgets harden into gates |
| `/speckit.specify` | Negative when input is vague | Silently converts ambiguity to "assumptions"; 16 green boxes ≠ specified |
| `/speckit.clarify` | **Strongly positive** | Menu choices prune the design space without saying so |
| `/speckit.plan` | Negative for non-trivial stacks | Defaults to the stack that passes the constitution most cheaply, not the one that fits the problem |
| Post-plan pushback (human) | Required | Recovers the better architecture, but only if the user knows enough to ask |

The pattern: **the framework's quality peaks at clarify, then declines as the
agent gets enough scaffolding to commit to choices.** Each later artefact looks
more authoritative than the last (numbered FRs, gate tables, rejected-alternatives
matrices), but the underlying decisions are increasingly anchored to earlier
arbitrary numbers rather than to the engineering problem.

---

## What this means for using Speckit (or any spec framework with an agent in it)

1. **Treat the constitution's "no new deps" type rules as biases, not neutrals.**
   They tilt every later decision toward stdlib stacks. If the better engineering
   answer requires a dep, you'll need to actively defend it; the framework will
   not.
2. **Treat measurable outcomes (SC-* numbers) as guesses until an engineer with
   domain knowledge ratifies them.** The agent will quote them back at you later
   as if they were measured. They weren't.
3. **Read every clarify menu as a design proposal in disguise.** The options
   listed are the design space the agent thinks exists. If the option you want
   isn't on the menu, that is the failure — not "I should pick the best of these
   three".
4. **The plan phase is the worst time to discover an architectural mismatch.**
   Plans are long, internally consistent documents that are exhausting to revise.
   Push back during clarify, or before.
5. **Generated artefact volume is a review hazard.** A 254-line plan with a Sync
   Impact Report and four passing gate tables takes longer to *read critically*
   than to *skim approvingly*. The friction sits in the wrong place.

The Speckit experience here is not an indictment of structured specification.
The 12 functional requirements, the bilinear-interpolation clarification, the
no-data-segment shape — all of those are better artefacts than what the user
would have produced free-hand. The lesson is narrower: **a specification produced
by an agent must be reviewed by an engineer who is willing to discard the agent's
architectural choices, not just to fact-check its requirements.** Without that
review, the framework delivers a polished local optimum and calls it done.

---

## Appendix — quick reference for the line cites

| Source | What's there |
|---|---|
| [`speckit_user_claude.txt:10`](./speckit_user_claude.txt) | `/speckit.constitution` invocation |
| [`speckit_user_claude.txt:496`](./speckit_user_claude.txt) | `/speckit.specify` invocation — one-sentence input |
| [`speckit_user_claude.txt:1046–1054`](./speckit_user_claude.txt) | Three "assumptions" silently chosen by specify |
| [`speckit_user_claude.txt:1440–1442`](./speckit_user_claude.txt) | User overturns CONUS-only assumption with real scale |
| [`speckit_user_claude.txt:2587–2603`](./speckit_user_claude.txt) | Clarify option menu (B chosen, C/Zarr-shaped not listed) |
| [`speckit_user_claude.txt:3323`](./speckit_user_claude.txt) | Plan selects SQLite + R-tree |
| [`speckit_user_claude.txt:3470–3530`](./speckit_user_claude.txt) | SQLite justification — "stdlib, zero new dependency" leads |
| [`speckit_user_claude.txt:3569–3572`](./speckit_user_claude.txt) | Zarr first rejected on SC-008 budget |
| [`speckit_user_claude.txt:3616–3617`](./speckit_user_claude.txt) | Agent walks back the rejection under pushback |
| [`speckit_user_claude.txt:3704–3747`](./speckit_user_claude.txt) | User's Zarr + R-tree proposal with working code |
| [`.specify/memory/constitution.md:106`](./.specify/memory/constitution.md) | Principle III "new deps must be justified" |
| [`.specify/memory/constitution.md:119–126`](./.specify/memory/constitution.md) | Principle IV performance-budget rule |
| [`specs/001-us-elevation-profile/spec.md:15`](./specs/001-us-elevation-profile/spec.md) | Clarified dataset scale (1,756 tiles, ~180 GB) |
| [`specs/001-us-elevation-profile/spec.md:125`](./specs/001-us-elevation-profile/spec.md) | SC-008 — the 5-minute budget that became a gate |
| [`specs/001-us-elevation-profile/plan.md:60–65`](./specs/001-us-elevation-profile/plan.md) | SQLite chosen as storage |
