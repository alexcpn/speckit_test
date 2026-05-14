# High-Level Sketch: The Limits of Specification-Driven Development (SDD) in AI Coding

## I. Introduction & Context: The AI-Native SDLC

- **The Autonomous Push**: SDD is implicitly marketed as the enabler for an "AI-native
  SDLC," which sets the expectation for fully autonomous software development.

- **Dismissing the Usual Objections**: Common criticisms of SDD miss the mark.

  - *Objection 1: LLMs are non-deterministic.* Reality: This is not a major problem.
    SDD frameworks essentially act as glorified prompts, and modern models produce
    highly consistent outputs when guided by them.

  - *Objection 2: Too much upfront documentation (like Waterfall).* Reality: This is
    also not a real problem. The upfront structure is genuinely good at cutting out
    boilerplate prompting and imposing order on chaotic "vibe coding".

- **The Real Dilemma**: The true problem isn't the framework itself, but the decision
  to use it for fully autonomous development rather than as a semi-autonomous
  assistant.

---

## II. The Core Argument: Implicit Assumptions and Fragile Foundations

- **The Assumption Trap**: SDD frameworks generate impressive-looking specifications.
  However, even though they ask the user for clarifications, many critical implicit
  assumptions slip through the cracks without the user realising it.

- **The Toy Project Threshold**: Any software beyond a simple toy project requires a
  solid technical stack and robust libraries to handle future non-functional
  requirements like reliability, high performance, and scalability. SDD frameworks
  often fail to build this solid base because of how their rules operate.

*Take from <https://github.com/alexcpn/speckit_test> README.md*

---

## III. The Experiment Walkthrough (Speckit / OpenSpec)

> Note: The user's experiment tested Speckit end-to-end, though frameworks like
> OpenSpec share the same fundamental mechanisms and flaws.

- **Phase 1 & 2 — Specify (The Hidden Assumptions)**: When given a vague prompt, the
  AI silently converts ambiguity into "assumptions" just to check off its
  requirements. In the experiment, the AI incorrectly assumed a hobby-scale dataset
  instead of the actual 23-billion data point requirement.

- **Phase 3 — Clarify (Pruning the Design Space)**: While this is the framework's
  strongest phase for catching missing information, the multiple-choice menus it
  presents to the user silently prune the design space, locking out robust
  architectural choices before they can even be considered.

- **Phase 4 — Plan (The Path of Least Resistance)**: Because the AI is penalised for
  introducing new dependencies, it defaults to the standard library rather than
  establishing a solid technical base for non-functional requirements. In the
  experiment, it chose SQLite because it required zero justification, actively
  rejecting better analytical libraries like Parquet or DuckDB.

---

## IV. The Necessity of Human Pushback

- **Challenging Arbitrary Rules**: The user must push back against the AI's
  constraints. When the user proposed a superior, heavy-duty Zarr-based architecture,
  the AI initially rejected it because it violated a 5-minute setup budget — an
  arbitrary number the AI had completely guessed earlier.

- **Selecting the Right Tech Stack**: To build non-toy software, an experienced
  engineer must step in to select the right algorithmic and storage tools (like an
  R-tree + Zarr store) using domain knowledge that the AI avoided.

---

## V. Conclusion & Takeaways

- **The Review Hazard**: Frameworks generate vast amounts of artefacts (sync impact
  reports, passing gate tables) that make the process look rigorous, which ironically
  makes critical human review easy to skip.

- **The Human Role**: Frameworks like Speckit and OpenSpec are great tools, but there
  are no silver bullets. A specification produced by an agent must be actively
  reviewed by an engineer willing to discard the AI's safe architectural choices to
  ensure the software has the solid technical foundation it needs.
