---
title: "Vertical Code Review: Using Claude Code to Bridge the Trust Gap Between Human and AI Review"
layout: "post"
---
# Vertical Code Review: Using Claude Code to Bridge the Trust Gap Between Human and AI Review

## The Problem With AI-Assisted Code Review Today

There's a growing tension in how engineering teams approach code review. On one side, traditional file-by-file review is slow but trustworthy — you read every line, you understand every change, and you can confidently stamp your approval. On the other side, AI review tools promise speed, but leave you uncertain. They'll flag things — or miss things — and at the end, you still don't know if you can trust the result. So you end up reviewing line by line anyway, making the AI pass little more than a warm-up exercise.

The root cause isn't that AI is bad at spotting issues. It's that the review paradigm itself is wrong. Traditional code review follows the structure of the codebase — file by file, alphabetically or by directory. This is a **horizontal approach**: you move laterally across the project's file tree. But humans don't understand systems horizontally. We understand them **vertically** — through behaviour, through user actions, through the question *"what happens when someone does X?"*

What if we restructured AI-assisted review around vertical slices, and used the AI not as an autonomous reviewer, but as a **guided walkthrough narrator** that takes you through each logical flow, layer by layer?

## The Concept: Vertical Slice Review

Consider a typical pull request that implements a new REST endpoint — say, `GET /users` — listing all users in a system. A traditional review might touch these files in order:

```
UserController.java
UserDTO.java
UserMapper.java
UserRepository.java
UserService.java
application.yml
schema.sql
```

You review the controller, hold its contract in your head, jump to the DTO, then the mapper, then eventually reach the service and repository. Your brain does the stitching. For small PRs this works fine. For anything substantial, context constantly leaks.

A vertical slice review reorders this around the **flow of a request**:

1. **HTTP / Resource Layer** — How does the request arrive? What's the route, the method, the response contract? What about error handling, authentication, pagination?
2. **Business / Service Layer** — What logic transforms the request into a result? What validations exist? What side effects?
3. **Persistence Layer** — How is data fetched? What queries run? Are there N+1 risks, missing indexes, transaction boundaries?
4. **Cross-Cutting Concerns** — Configuration changes, migrations, security, test coverage, observability.

Each vertical slice follows one action from entry to exit. You never lose context because the narrative is continuous.

## How This Works With Claude Code

Claude Code is well-suited to this because it can read entire codebases, understand architectural patterns, and — critically — restructure its output around any axis you choose. Instead of asking it to "review this PR," you ask it to **guide you through it**.

### The Workflow

**Phase 1 — Orientation.** Claude analyses the full changeset and produces a high-level summary: what's the intent, what's the architectural approach, how many distinct vertical slices exist, and in what order should you walk through them.

**Phase 2 — Guided Vertical Walkthrough.** For each slice, Claude takes you layer by layer through the relevant changes. At each layer, it shows you the code, explains the approach, flags potential issues, and highlights interesting design decisions. You read, you understand, you ask questions. The stamp is yours, but you reach it faster.

**Phase 3 — Residual Sweep.** After all verticals are covered, Claude presents anything that doesn't belong to a clear flow — configuration changes, CI modifications, dependency updates, refactors that touch shared utilities. This is where things hide, so it matters.

### The Prompt

Below is a ready-to-use prompt for Claude Code that implements this workflow. Adapt the layer terminology to match your stack (the example below assumes a typical Java/Spring-style backend, but the approach works for any layered architecture).

```markdown
I need you to guide me through a code review of this changeset using a vertical slice approach.

## Phase 1: Orientation

Start by giving me:
- A one-paragraph summary of what this changeset does and why
- An architectural overview of the approach taken
- A numbered list of distinct vertical slices (user actions or code paths) that were
  added or modified, ordered by importance (main flows first, edge cases and error
  paths after)
- Any files or changes that don't belong to a clear vertical slice (config, CI,
  migrations, shared utilities) — we'll cover these last

## Phase 2: Vertical Walkthrough

For each vertical slice, walk me through it layer by layer, top-down. At each layer:

1. Show me the relevant code changes (only what's relevant to this slice)
2. Explain the approach — what does this code do and why was it written this way?
3. Flag any issues: bugs, security concerns, performance risks, missing edge cases,
   deviations from common conventions
4. Highlight anything well-done or interesting — good patterns deserve recognition
5. Note anything you're uncertain about or cannot fully verify (concurrency, runtime
   performance, integration behaviour)

Use these layers (adapt to our stack):
- **HTTP / Resource layer** — routes, controllers, request/response contracts,
  validation, auth
- **Business / Service layer** — orchestration, domain logic, validations,
  side effects
- **Persistence layer** — repositories, queries, transactions, migrations
- **Tests** — coverage for this specific slice

Wait for my confirmation before moving to the next layer or the next slice.
I may ask questions, request deeper analysis, or skip ahead.

## Phase 3: Residual Sweep

After all slices are covered, walk me through everything that remains:
- Configuration changes
- CI/CD changes
- Dependency updates
- Shared utility modifications
- Anything else not covered

For each, briefly explain what changed and flag anything noteworthy.

## Ground Rules

- Do not skip any changed file. Every modification must appear in at least one
  slice or in the residual sweep.
- If a file is relevant to multiple slices, show the relevant portion in each
  slice rather than saying "we already covered this."
- Be honest about what you can and cannot assess. Static analysis is your strength;
  runtime behaviour, performance under load, and subtle concurrency issues are not.
- I want to understand every change well enough to own the approval. Help me get
  there efficiently, but never at the cost of thoroughness.
```

### Claude Code Custom Skill (CLAUDE.md)

If you use this approach regularly, you can encode it as a custom instruction in your project's `CLAUDE.md` file. This ensures Claude Code defaults to this review style whenever you ask for a review.

```markdown
# Code Review Protocol

When asked to review code, a PR, or a changeset, always use the vertical slice
review method unless explicitly told otherwise.

## Method

1. **Orientation first.** Summarise intent, architecture, and identify all distinct
   vertical slices (user actions / code paths). Order them: main flows first, then
   edge cases, then error paths.

2. **Walk through each slice top-down through architectural layers:**
   - HTTP / Resource layer (routes, controllers, DTOs, validation, auth)
   - Business / Service layer (orchestration, domain logic, side effects)
   - Persistence layer (repositories, queries, transactions, schema changes)
   - Tests (relevant test changes for this slice)

3. **At each layer, for each slice:**
   - Show relevant code changes only
   - Explain the approach and reasoning
   - Flag: bugs, security issues, performance risks, missing edge cases,
     convention violations
   - Highlight: good patterns, interesting decisions
   - Disclose: what you cannot verify (concurrency, runtime perf, integration
     behaviour)

4. **After all slices: residual sweep.** Cover config, CI, dependencies, shared
   utilities, and anything not part of a clear vertical.

5. **Completeness rule.** Every changed file must appear in at least one slice or
   in the residual. Never silently skip a file.

6. **Pacing.** Pause after each layer. Wait for confirmation before continuing.
   The reviewer may ask questions, request deeper analysis, or skip ahead.

## Tone

Be direct. Flag issues clearly with severity (critical / warning / nit). Explain
reasoning concisely. Acknowledge good work. Be explicit about the limits of static
analysis.
```

## When This Works Best

This approach has a sweet spot. It's not universally better than file-by-file review — it's better in specific conditions.

**Optimal for:**

- Feature PRs that add or modify clear user-facing behaviour (new endpoints, new workflows, new integrations)
- Medium-complexity changesets (roughly 5–30 files) where the vertical structure is clear but non-trivial to reconstruct mentally
- PRs that span multiple architectural layers, especially when the interesting decisions live in how the layers interact
- Onboarding reviewers into unfamiliar parts of the codebase — the guided narrative acts as both review and teaching tool
- Teams where reviewers frequently context-switch between different services or domains

**Less effective for:**

- Tiny PRs (under 5 files) where file-by-file review is already fast enough
- Large-scale refactors that change 100+ files in the same mechanical way — these are better reviewed by verifying the pattern once and spot-checking
- Pure infrastructure or configuration changes with no vertical flow to follow
- Performance-critical changes where the real review requires profiling, benchmarking, or load testing — static vertical walkthrough adds little value here

## The Risks

No approach is without trade-offs. Be aware of these:

**False sense of thoroughness.** The guided walkthrough *feels* comprehensive. Claude narrates confidently, layers connect logically, and you reach the end thinking you've seen everything. But Claude is doing static analysis of text — it cannot verify runtime behaviour, thread safety, performance characteristics, or subtle state interactions. The risk is that the polished narrative masks blind spots. Mitigation: insist that Claude explicitly states what it *cannot* assess at each layer, and treat those disclosures as action items for manual verification or testing.

**Increased review time for sprawling PRs.** If a changeset touches many verticals that share files, Claude will show you the same files through different lenses. For a PR with 8 overlapping slices, this can take longer than a single file-by-file pass. Know when to switch strategies — if you find yourself seeing `UserService.java` for the fifth time, a traditional review might be faster.

**Dependency on AI accuracy.** Claude might misidentify the architectural layers in your specific project, group changes into the wrong slices, or miss that a seemingly innocent utility change affects a critical path. Always cross-check the orientation summary against the actual file list in the PR. If Claude's slice map doesn't match your understanding, correct it early.

**Anchoring bias.** When Claude explains *why* code was written a certain way, it's often reasoning post-hoc from the code itself, not from the author's actual intent. This can anchor your judgment — you might accept a flawed approach because Claude's explanation sounds reasonable. Stay critical. Ask yourself: "Would I have accepted this explanation from the PR author?"

## Closing Thought

The fundamental insight here is simple: code review should follow the shape of human understanding, not the shape of the file tree. AI doesn't replace your judgment — but it can restructure the information so your judgment works faster and more naturally. The quality gate stays with you. The path to reaching it just got more efficient.
