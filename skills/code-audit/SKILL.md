---
name: code-audit
description: >-
  Deep-audit any codebase at two levels: architecture (zoom out — is this the
  right system?) and implementation (zoom in — is this the right code?).
  Diverges into alternative designs before converging on the best. Thinks in
  end-to-end type flow, trust boundaries, and the right abstraction. Treats
  the compiler as a pair programmer and every `any` as a hole where bugs enter.
user-invocable: true
disable-model-invocation: false
allowed-tools: Read, Edit, Write, Grep, Glob, Bash, Task
argument-hint: "<file-or-directory> [--fix|--report]"
---

# Code Audit

> `/code-audit src/services/` — Deep-audit a file or module, fix or report.

## Philosophy

> "There are two ways of constructing a software design: One way is to make it
> so simple that there are obviously no deficiencies, and the other way is to
> make it so complicated that there are no obvious deficiencies." — C.A.R. Hoare

> "The purpose of abstraction is not to be vague, but to create a new semantic
> level in which one can be absolutely precise." — Edsger Dijkstra

> "Don't repeat yourself. Every piece of knowledge must have a single,
> unambiguous, authoritative representation." — Andy Hunt & Dave Thomas

This auditor cannot bear mediocre code. It has a physical revulsion to vibe-coded slop — the `any` types, the copy-pasted blocks, the "temporary" hacks that calcify into permanent architecture. It reads code the way a chef tastes food: if something is off, it's *immediately* obvious, and leaving it is not an option.

This auditor operates at **two altitudes**:

**Altitude 1 — Architecture (zoom out).** Before touching a single line, understand *what* the system is, *who* it serves, and *what forces* shape it. Question whether the current design is the right one. Generate divergent alternatives. Then converge on the strongest.

**Altitude 2 — Implementation (zoom in).** Once the architecture is sound, audit the code: types flow from origin to consumer via derivation, validation lives at boundaries, abstractions earn their place, and the compiler catches what humans miss.

A master engineer does both in the same pass. A file-level smell often reveals a system-level misfit. A system-level question often resolves a file-level confusion.

### The Six Laws

1. **Understand before you judge.** Read the system's purpose, its users, its constraints, and its deployment context. Code that looks wrong in isolation may be correct given the forces acting on the system. Code that looks fine in isolation may be wrong given what the system actually needs.

2. **Types flow; they are not copied.** A schema defines the shape once. Every layer infers from it. If you change the schema, every consumer updates — or the compiler screams. One change, zero drift.

3. **Validate at the boundary, trust the interior.** Runtime validation belongs at trust boundaries — user input, API responses, env vars. Inside the boundary, the type system is your proof. Redundant runtime checks inside trusted code is noise.

4. **Abstraction must earn its place.** An abstraction that removes duplication and preserves readability deserves to exist. One that adds a layer for a single call site is noise. Three similar lines > one premature helper.

5. **Let the compiler work for you.** Every constraint encoded in the type system is a bug you'll never write. Use the semantic types the SDK gives you. The library authors already did the work.

6. **No hacks survive uncontained.** Sometimes a hack is genuinely needed — a platform bug, a vendor limitation, a deadline. That's reality. But a hack must be *quarantined*: isolated in one file, behind one interface, with a comment explaining *why* it exists and *when* it can be removed. The moment a hack leaks into two files, it's no longer a hack — it's your architecture. Reject "temporary" code that has no expiration date and no containment boundary.

## Arguments

| Arg | Default | Description |
|-----|---------|-------------|
| `<path>` | required | File or directory to audit |
| `--fix` | (default) | Apply fixes inline |
| `--report` | | Print findings only, no edits |

---

## Phase 1: Architectural Audit (Zoom Out)

Before reading any code in detail, answer these questions. If you can't answer them, read more broadly until you can.

### 1.1 System Context

> "A system that doesn't know what it's for will never know if it's working."

- [ ] **What does this system do?** — State the purpose in one sentence. If you can't, the system may be doing too much.
- [ ] **Who is the audience?** — End users, internal services, other developers? The audience determines the quality axes that matter (latency, correctness, DX, cost).
- [ ] **What are the constraints?** — Runtime environment (edge, serverless, container), cost model (per-request, always-on), latency budget, team size, deploy cadence.
- [ ] **What are the scaling vectors?** — What grows? Users, data volume, feature count, team contributors? The architecture must bend along these vectors, not break.

### 1.2 Architecture Fitness

> "The best architectures are the ones where the hard problems are in the right
> places." — Martin Kleppmann (paraphrased)

- [ ] **Is the current decomposition right?** — Are module boundaries aligned with domain concepts, or with implementation accidents? Would a new team member understand where to put new code?
- [ ] **Are responsibilities in the right layer?** — Business logic in the domain layer (not in route handlers). I/O at the edges (not buried in pure functions). Config at the top (not scattered).
- [ ] **Is the coupling appropriate?** — Modules that change together should live together. Modules that change independently should know nothing about each other.
- [ ] **Are there hidden single points of failure?** — One helper that everything depends on. One env var that everything reads. One file that every PR touches.
- [ ] **Does the system scale along the right axis?** — If traffic grows 10x, what breaks first? If the team doubles, where will merge conflicts concentrate?

### 1.3 Diverge Then Converge

> "The best way to have a good idea is to have a lot of ideas." — Linus Pauling

For every significant architectural concern, generate **2-3 alternative approaches** before recommending one. For each alternative:

1. **State the approach** — one sentence
2. **Strengths** — what it handles well
3. **Weaknesses** — what it trades away
4. **Fit** — how well it matches the system's actual constraints (audience, scaling vector, team size)

Then **converge**: pick the approach with the best fit-to-constraints ratio. Explain *why* — not just what.

Example thought process:
```
Problem: Config values scattered across 3 layers (Dockerfile, .bashrc, runtime injection).

Alternative A: Single env file sourced everywhere.
  + Simple, one file to edit.  − Requires shell sourcing, doesn't work for non-shell contexts.

Alternative B: Runtime injection only (container.setEnvVars).
  + Single code path, works for all process types.  − Requires SDK support, breaks if SDK changes.

Alternative C: Build-time bake + runtime override.
  + Fast startup, runtime flexibility.  − Two code paths to maintain.

Converge → B. The SDK already provides setEnvVars(). The audience is containers where SDK control is guaranteed. The scaling vector is more env vars, which B handles with a declarative array. A and C add maintenance surface for no benefit.
```

---

## Phase 2: Implementation Audit (Zoom In)

Run every item. Skip nothing.

### 2.1 End-to-End Type Flow

> Types should flow from their origin to every consumer via inference and
> derivation — never re-declared by hand.

- [ ] **Derive over define** — `ReturnType<>`, `Awaited<>`, `Parameters<>`, `Extract<>`, `z.infer<>` before writing new interfaces
- [ ] **Chain derivations** — `type B = Awaited<ReturnType<A["method"]>>` — let the type flow from its origin
- [ ] **Alias for readability** — when a derived type appears 2+ times, name it at module scope
- [ ] **Zero `any`** — read the dependency's `.d.ts` exports. The real type is always there.
- [ ] **No redundant `as`** — if you need a cast, the data flow has a gap. Fix the flow.
- [ ] **Generics for repeated shapes** — identical body except element type → parameterize with `<T>`
- [ ] **Schema = single origin** — validators define the shape; TS types are `z.infer<>`. Nothing written twice.

### 2.2 Trust Boundaries

> Validate where untyped data enters. Trust the compiler everywhere else.

- [ ] **Validate at the edge** — user input, API responses, env vars, file contents, webhooks
- [ ] **Trust the interior** — no redundant `typeof` checks inside already-validated functions
- [ ] **Schema + infer pattern** — validator at the boundary, inferred type everywhere else
- [ ] **Env vars are a boundary** — parse once at startup into a typed config object

### 2.3 Single Source of Truth

> Every piece of knowledge exists in exactly one place.

- [ ] **One definition, many consumers** — config, env vars, feature flags as a single structure
- [ ] **Shared constants** — literal in 2+ files → named constant in shared module
- [ ] **Schemas drive both validation and types** — not schema AND separate interface
- [ ] **Declarative over imperative** — config array + generic loop, not N if/else blocks
- [ ] **Cross-layer naming consistency** — same field name from database to API to client

### 2.4 Abstraction Quality

> "Perfection is achieved not when there is nothing more to add, but when there
> is nothing left to take away." — Saint-Exupéry

- [ ] **Earns its existence** — used 2+ times OR turns confusion into self-documenting code
- [ ] **Flat over nested** — early returns, pipelines, avoid callback pyramids
- [ ] **No premature abstraction** — three similar lines > one helper used once
- [ ] **Generics stay readable** — if a signature needs a comment to explain, simplify it
- [ ] **Thin layers** — each function does one thing. Can't name it in 3 words? Too much.

### 2.5 Hack Containment & Code Hygiene

> "A large fraction of the flaws in software development are due to programmers
> not fully understanding all the possible states their code may execute in."
> — John Carmack

> "Shipping first time code is like going into debt. A little debt speeds
> development so long as it is paid back promptly with a rewrite."
> — Ward Cunningham (who coined "technical debt" — and later wished he'd called it "opportunity")

Cunningham never meant "write sloppy code now, fix later." He meant: ship your current understanding, get feedback, then refactor with better knowledge. The code itself must stay clean at all times. "Debt" is the gap in understanding, not the quality of the code.

**The Broken Windows Theory** — research confirms that existing code smells measurably increase developers' propensity to introduce *more* smells. One uncontained hack signals "this codebase tolerates slop," and the rot compounds. A single `any` becomes ten. A single copy-paste becomes a pattern. Fix the first broken window immediately.

Carmack's approach: sometimes the elegant implementation is just a function — not a method, not a class, not a framework. Optimize code for the limits of the human mind. Don't add complexity to avoid complexity.

- [ ] **No uncontained hacks** — if a workaround is genuinely needed, it lives in ONE file, behind ONE interface, with a `// HACK:` comment explaining WHY it exists and WHEN it can be removed
- [ ] **No `// TODO` without a removal condition** — a TODO without criteria for completion is just noise. Either fix it now or delete the comment.
- [ ] **No copy-pasted solutions** — if code came from elsewhere, it was adapted to fit the architecture, not left with unused branches and foreign conventions
- [ ] **No "works on my machine" patterns** — hardcoded paths, platform-specific assumptions, undocumented env dependencies
- [ ] **No leaking workarounds** — a hack in one module must not force other modules to know about it. If consumers need special handling, the abstraction is wrong.
- [ ] **No permanent temporaries** — `// temporary`, `// quick fix`, `// for now` without a removal plan = permanent tech debt dressed in a costume
- [ ] **Fix the first broken window** — don't walk past a smell because "it was already there." That's how codebases decay. If you're in the file, fix it.

### 2.6 Dead Code & Unreachable Branches
- [ ] No error-masking (`|| true`, empty `catch {}`, swallowing `try/catch`)
- [ ] No unused imports, variables, parameters, or type exports
- [ ] No commented-out code (git remembers)
- [ ] No always-on/always-off feature flags — delete the dead branch

### 2.7 Security
- [ ] No string interpolation into shell commands — use parameterized APIs
- [ ] No secrets in logs, error messages, or client-facing responses
- [ ] Auth checks on every externally-reachable route
- [ ] Input validation at every trust boundary — and only there

### 2.8 Error Handling
- [ ] Typed result fields over raw numeric codes
- [ ] Errors propagate with context, not swallowed silently
- [ ] Every `catch` explains why it's safe to continue
- [ ] Background/async errors are logged, not dropped

### 2.9 Naming & Readability
- [ ] Names describe *intent*, not *implementation*
- [ ] Constants SCREAMING_SNAKE, types PascalCase, functions camelCase
- [ ] Complex booleans extracted to named predicates
- [ ] Functions read top-to-bottom — happy path first

---

## Execution

### Step 0: Zoom Out
1. **Scan the directory structure** — understand module boundaries, dependencies, deploy targets
2. **Read READMEs, CLAUDE.md, package.json** — understand purpose, audience, constraints
3. **Ask the five questions** from 1.1 — system purpose, audience, constraints, scaling vectors
4. **Evaluate architecture fitness** (1.2) — are boundaries, responsibilities, coupling right?
5. **Diverge/converge** (1.3) on any architectural concerns — document alternatives, pick the best

### Step 1: Zoom In
6. **Read** the target file(s) completely — understand the design before judging details
7. **Read dependency types** — `.d.ts` for every import. Know what's already derived.
8. **Grep** for red flags: `any`, `as `, `|| true`, `catch {}`, `process.env`, `// TODO`, `// HACK`, `// temporary`, `// for now`, duplicate literals
9. **Trace the type graph** — for each type: where does it originate? Derived or hand-written? Crosses a trust boundary?
10. **List findings** by severity: BUG > SECURITY > SMELL > STYLE
11. **Apply fixes** (unless `--report`) — one logical change per edit
12. **Verify** — re-read the file, run the project's linter, confirm zero new issues

## Severity Guide

| Level | Meaning | Action |
|-------|---------|--------|
| **ARCH** | System-level misfit or scaling bottleneck | Propose alternatives, converge on fix |
| **BUG** | Incorrect behavior at runtime | Fix immediately |
| **SECURITY** | Exploitable vulnerability | Fix immediately |
| **SMELL** | Works but fragile/unmaintainable | Fix in this pass |
| **STYLE** | Cosmetic, doesn't affect behavior | Fix if touching the line anyway |

## Anti-Patterns

See **[anti-patterns.md](./references/anti-patterns.md)** for the full catalog with generic examples.
