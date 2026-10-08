# How Family Capsule is built

Family Capsule is developed by one person working with AI coding agents (Claude Code and Codex). The interesting part is not that agents write code; it is the process that makes their work **resumable, verifiable and safe for security-critical software**.

## Sections, not tasks

All work is cut into atomic **sections**. Each has an ID, a risk tier, its dependencies, a definition of done and its own tests. An index of sections plus one detail file per long section is the plan, the backlog and the status board at once. Dependency-aware statuses (`todo`, `in-progress`, `green`, `needs-reverify`, `blocked`) mean that changing one section marks the ones built on it for re-verification.

## The repository is the memory

An agent session can be cut off at any moment by a usage limit, so nothing important lives in chat:

- a **runbook** of binding rules, kept short because it is loaded on every turn, with the reasoning behind each rule kept separately;
- a **current-state handoff**, edited in place: current section, verified commit, results, the next concrete step;
- an **append-only dev log** of every action, decision and incident;
- notes on **what each shape of section actually cost**, used to plan the next one.

A cold session reads the runbook, the handoff, the section index and `git log`, and can continue where the last one stopped — on the same branch, without reconstructing anything from memory.

## Risk tiers and verification

- **Security-critical (SEC) and domain sections are test-first.** SEC tests must include negative and attack cases (tamper, wrong key, man-in-the-middle) and external anchors (interop vectors), not just the happy path.
- **A test is not considered written until it has been seen failing against the specific defect it claims to catch.** When the code is already fixed, the defect is re-introduced on purpose. Independent defects can be batched, but the set of failing tests must match the expected set exactly.
- **A security review gates every SEC merge**, run against a fixed checklist by a separate strong-model reviewer on the real diff. Cross-cutting mechanisms (middleware, rate limiting, wire formats) are reviewed once more as soon as their shape is decided, before tests are built around them.
- **Retries are budgeted.** At most three fix attempts per failure, each based on all errors at once. After that, or at any design question, the agent stops and asks instead of guessing.
- **The human owns the verdict.** Features with a visible surface end in a manual smoke test on real devices, and a section merges only after it passes.

## Golden rules

1. **Blind server.** The API project may reference only the contracts project; plaintext and private keys never reach the server.
2. **No invented crypto.** The wire format is fixed and verified; no in-house PAKE.
3. **Never log secrets**, not even in development logs.
4. **Version-tag every ciphertext** from day one.

## Keeping agents efficient

Agent cost is dominated by round trips, not by the code written. Each round trip re-reads the whole context. So the process treats context as a budget:

- documents read at every start are kept small: the plan is an index plus per-section files, the history lives in archives that are opened only to answer "why";
- builds and tests run through a wrapper script that returns a digest instead of a log, waits by itself instead of letting the model poll, and never kills a run;
- work-in-progress verification is targeted (one project, filtered tests); the full solution build and full suite run once, at the section's final gate.

These rules came from measurement: build logs, commit counts and usage windows per section, reviewed and adjusted as the project goes.
