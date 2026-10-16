---
name: quality-review
description: Review a diff, branch, PR, or existing code (a path or the whole repo) for code quality against ~/.claude/CODING_STANDARDS.md, using the deep-module vocabulary (module, interface, seam, adapter, depth, leverage, locality). Covers module depth and responsibility, state ownership and immutability, concept ownership and types (including making invalid states unrepresentable), dependency injection and config, error visibility, naming and function readability, and test quality; on a whole codebase it also questions the product's major design decisions against alternative directions. Use when the user says "quality review", "review for code quality", "review against my coding standards", "is this well designed", or invokes /quality-review. Not a bug hunt; use /code-review for correctness.
argument-hint: "[PR number | branch | path | .]  (default: current branch vs its base)"
---

# Quality review

Target: $ARGUMENTS

Review the target for design and test quality. Report findings; don't edit code.

## 1. Load the rubric

Read both before looking at code:

- `~/.claude/CODING_STANDARDS.md` is the rubric. Every finding cites one of its section headings.
- [design-vocabulary.md](design-vocabulary.md) holds the terms to phrase findings in, and the tests used to verify them below.

Also read the repo's `AGENTS.md` / `CLAUDE.md` if present. Where those files say the repo deliberately does something the standards discourage, say so once in the report rather than flagging every instance. A habit you only observe in the code (every module reads the environment, every test monkeypatches) is not a documented choice: new code that follows it is still a finding.

## 2. Resolve the target

- No argument: the current branch against its base. `git merge-base HEAD origin/main` (or `master`), then `git diff <base>...HEAD` plus uncommitted changes from `git diff HEAD`.
- PR number or URL: `gh pr diff <n>` and `gh pr view <n> --json baseRefName,headRefName,files`.
- Branch: `git diff <base>...<branch>`.
- Path, or `.` for the whole repo: review the existing code under it as it stands, with no diff. This is codebase mode, below.

## 3. Read enough to judge design

### Change mode (diff, branch, PR)

A diff alone can't show whether a module is deep. For each changed module, read the whole file, and grep for its callers and its tests. Depth, ownership and duplication are judgments about the call sites, so look at them.

Scope findings to what the change introduces or makes worse. A pre-existing problem in code the diff touches is in scope if the change was the natural moment to fix it; one in untouched code is not.

New modules get module-scale review, as in codebase mode: walk Types and Readability as fully as the other sections. For each new or substantially rewritten module, list every function with its parameter and return types before judging it, and note each untyped map, dict or tuple it accepts, returns or reads keys from, and each single-letter name.

Measure cyclomatic complexity for every changed source file before judging Readability, with the language's usual tool: for example `radon cc -s` for Python, `gocyclo` for Go, eslint's `complexity` rule for JavaScript/TypeScript, clippy's `cognitive_complexity` for Rust, PMD for Java. Run it through the repo's own tooling if it has some, or a one-off runner (`uvx`, `npx`, `go run`) if not. Prefer a tool that counts short-circuit boolean operands; some (ruff's C901, for one) skip them and underscore the worst functions. With no tool available, count decision points by hand as the rubric defines them. A function the diff adds that scores above 10, or an existing one the diff pushes past 10 or raises further, is a candidate. For an existing function, score the base version too (write `git show <base>:<file>` to a scratchpad file and measure that) so the finding can say what the change added.

When the diff adds more than about 150 lines of source, launch two `general-purpose` subagents in parallel while you review the design sections, each given the diff, the changed file list and the step 4 verification rules for its sections: one for Types, Readability and "Lean toward immutable data where it pays off", the other for Testing, applying the assertion checks to every new test. Verify what each returns before keeping it.

### Codebase mode (path or `.`)

Everything under the path is in scope, but the bar for a finding rises with the size of the scope. A single module or small directory (roughly under 1,500 lines of source) gets module-, function- and naming-level findings: walk Types and Readability as fully as the other sections. A package or the whole repo gets only the big-ticket problems: the design decisions and architectural patterns that hold the codebase back. The same applies when the user names a feature or tool (e.g. "the whole CLI") that turns out to be one module: judge it at module scale. Ask of each candidate at package or repo scale:

- How many modules, tests or future changes pay for it? A cost confined to one function is a nit at this scale.
- Would fixing it change how new code in this repo gets written? If the fix is a few local lines with no effect on callers, drop it.
- Is it a pattern or a one-off? Individual instances count only as evidence for a pattern.

Examples of big-ticket problems: an external system with no owning module, so every caller re-does auth and pagination; policy and orchestration fused in the entrypoints, so decisions can only be tested end to end; state ownership split across modules that each know the storage layout; clients and config built at import time, so nothing can be tested without the real environment; a test suite that patches internals throughout because no seams exist. A long function, a misnamed variable, a single swallowed exception or one weak assertion is not big-ticket on its own.

These problems span modules, so build a map before reading files in depth:

- the modules (files or packages) and roughly what each does, from `git ls-files <path>` and a skim of each module's top
- how the main pieces connect: what calls what, and how state and data move between entrypoints, modules and external systems
- each external system (API, database, queue, file format, model) and every module that talks to it directly
- every place environment or config is read (`os.environ`, `getenv`, `process.env`, flag parsing)
- module-level mutable state and import-time effects (clients, connections, caches, locks built at module scope)
- the entrypoints, and the tests directory with which modules each test file exercises

Grep is enough for most of this. Keep the map short; it's a working index, not part of the report.

If the scope is small (roughly under 3,000 lines of source), review it directly from the map, and in parallel launch two `general-purpose` subagents, each given the file list and the step 4 verification rules for its sections. One gets only the Types, Readability and "Lean toward immutable data where it pays off" sections of the rubric. The other gets only the Testing section and must apply the assertion checks to every test. That way those sections get full attention rather than what's left after the earlier ones. Verify what each returns before keeping it. Otherwise split it into areas of related modules, each with its tests, and give each area to a `general-purpose` subagent, launched in parallel. Give each one the rubric path, the vocabulary path, the map, its file list, the step 4 verification rules and the big-ticket bar above, and tell it not to edit anything. A subagent that only sees its own area drifts toward local nits, so ask it for the design patterns it sees in its area and the evidence for each (file:line), not a list of line-level findings. Cross-module problems come from the map, so work those out yourself, then combine them with the subagents' patterns, merge the ones that are the same underlying decision seen from different areas, and re-check the evidence for each before reporting.

## 4. Review against each section

Walk the standards in order: Modules, State, Types, Dependencies and configuration, Errors, Readability, Testing. For each candidate finding, verify it before keeping it:

- Shallow module or pass-through: apply the deletion test. Name the callers and say what complexity would reappear in them, or show that nothing would.
- Abstraction that hasn't earned its place: count the adapters at the seam. A test fake counts. One adapter with nothing hidden behind it is a finding; two is a real seam.
- Missing abstraction: check the opposite direction just as hard. Look for a concern handled inline in three or more places, inside one module as well as across modules.
- Missing owner / duplicated concern: list each place the concern is written (file:line), not "this appears in several places".
- Concept with no owner: list every place it is represented, parsed or built (file:line), and any signature where same-typed parameters with different meanings sit side by side. Identifiers count as concepts, not just structured data. Names and IDs of different kinds passed as the same primitive (a source and a target repo name, a branch name and a ref, two kinds of ID) are the main case to catch. Then say what kind of owner fits. If callers hold and pass the concept, recommend a type for it. If they only need something done with it, recommend hiding it inside the module that does the work. Don't default to a domain type.
- Ordering constraints or hand-built keys/paths/IDs: show the caller doing the sequencing or construction.
- Shared mutable state, load-time effects, config read deep: name the global or the import-time call, and the test or second configuration it blocks.
- Snapshot writes: show the read, the long-running step, and the write-back.
- Mutable data that would be better immutable: report it only when the mutability has a concrete cost. Show the second holder that sees an unexpected change, the cache or key it makes unsafe, or the change made far from where the value was created. A mutable structure with one owner that changes it through its own code isn't a finding. If the immutable version would cost performance, say what to memoize or share, and where that cache lives.
- Invalid state that can be represented: name the state that must never happen and show the type that allows it (optional fields that must be set together, combinations of booleans, a string with a few allowed values). List the guards or `None` checks that exist only because of it (file:line). Propose the shape that makes it impossible, and say which guards that change deletes.
- Dict or tuple as a concept: list the keys read and every function that reads them (file:line), and every place the dict is built. Propose the record type in the repo's existing style (a pydantic model or dataclass, a TypeScript interface, a Go struct, whatever the repo already uses), holding only the fields the code uses, and say where the payload gets parsed into it.
- Module that owns an external system: check whether its functions share a credential, identity, base URL or cache through module globals or per-call environment reads, and whether tests patch several of its functions. If so, recommend a client class with domain-named methods, built at startup and passed in, and name the patches the fake would replace.
- Missing doc comments: list each non-trivial function without one (file:line) and what its doc comment would need to say that the signature doesn't (a fallback, an error mode, state it touches, an input shape). Group them into one Readability finding per module.
- Single-letter names: report them as one grouped Readability finding listing each (file:line) with a proposed name. They count toward the Readability cap as one finding, never as a reason to drop them.
- Swallowed errors: show the path from the failure to an outcome that changes while only a log line records it.
- Name with several meanings: grep it and list what it holds at each site.
- Name that hides its level or object: show the call chain or call sites where it reads ambiguously, and propose the replacement.
- Function that does more than one thing: try to write its one-sentence summary. If you can't, quote the branch, try block or mode parameter that stops you.
- Function too complex: for each candidate from the complexity measurement, give its score, then list the phases it runs in order (filter the input, fetch, claim, validate, build, create, record, notify, ...) with their line ranges and each phase's failure rule. Propose the split as named functions, one per phase or decision, with signatures, and say what each extracted piece makes testable without patching; usually the decisions become pure functions and the original reads as a short sequence of named steps. One finding per function. Above 20 it ranks with the design findings; 11 to 20 is a Readability finding that doesn't count toward the Readability cap. Don't drop a split because the function is an entrypoint or because the repo has larger ones.
- Test assertions: name the product bug that would make the assertion fail. If you can't, that's the finding. For parametrized or table-driven cases, trace each input through the code and say which part of it reaches the behaviour the case is named for. A case whose distinguishing input is consumed by an earlier branch doesn't cover that behaviour, even though a test for it exists. Likewise, check that the fixture or fake can actually produce the input that would trigger each guard or refusal path. If it never can, the guard is untested.
- Test doubles: say what the test is about, then whether the double sits outside that boundary or re-implements product logic.
- Tests that patch internals: point to the decision that should be pulled out or the dependency that should be passed in, since that is the real fix.
- Proposed seam or port, in any finding's fix: give it operations named in the domain (`close_ticket(id)`), not transport verbs (`get`/`post`/`put`). A port that only forwards requests fails the deletion test, and its fake would have to re-implement the routing. Check every proposed seam against the deletion test before reporting it.

Give Testing the same depth as the sections before it, even when the earlier sections produce many findings. Run every Testing check above against each test file: for every assertion, name the product bug that would make it fail. A security-relevant behaviour with no failing assertion (redaction, an auth check, an approval gate) ranks with the design findings.

Drop anything you can't make concrete. Drop style nits the standards don't cover. Don't report correctness bugs here unless they follow directly from a quality issue (e.g. a snapshot write that loses data); mention them in one line and point to /code-review.

## 5. Question the design decisions (codebase mode, package or repo scale)

Step 4 takes the system's shape as given. This step doesn't. It asks whether the decisions the product is built around still hold, and what a different direction would solve differently or remove entirely.

Reconstruct the problem first. From the README, `AGENTS.md`/`CLAUDE.md`, docs, the entrypoints and `git log` (the early commits and the ones that introduced each major piece), write a few sentences on what the product is for, what it serves, and its constraints. Split the constraints into hard ones (security, compliance, platforms the org is committed to, scale) and choices, and mark which you found documented and which you inferred.

Then list the major decisions as "to solve Y, chose X". These are the three to six choices everything else is built around: where it runs, how work is triggered and sequenced, where state lives, the unit of decomposition, which external systems are load-bearing, and build versus buy. Where commit messages or docs say why a decision was made, note it.

Having read the code, the current design will look inevitable, so get alternatives from somewhere that hasn't. Launch two or three `general-purpose` subagents in parallel. Give each only the problem statement and the hard constraints, not the map or the current design, and tell it not to read the repo. Give each a different priority, for example: the fewest moving parts and no infrastructure we operate; make the problem the hardest current subsystem exists for go away; buy instead of build wherever a mature product fits. Ask each for a sketch of the system it would build: components, how work flows, where state lives, and what it gives up.

Compare each major decision against the alternatives from the subagents and from your own reasoning. An alternative counts if it solves Y a different way, or changes the product so Y never comes up. For each one worth reporting, give:

- the current decision and the problem it solves
- the alternative, concretely
- what it would remove: which step 4 findings disappear, which modules stop needing to exist
- what it costs: the new trade-offs, what it can't do that today's design can, and the migration effort
- what would have to be true for it to win, and a cheap way to find out (a spike, a measurement, a question for the team)
- a verdict: keep, revisit when a named condition changes, or worth moving toward now

Keep is a valid verdict, and saying briefly why a decision holds up is useful. Don't propose an alternative that breaks a hard constraint, and don't treat migration as free. When an alternative depends on what a third-party product can do today, check it with a web search rather than recalling it.

Many step 4 findings are symptoms of one of these decisions. Link each finding to the decision behind it, if any, so the report shows which problems a change of direction would remove and which need fixing either way.

## 6. Report

Rank by how much the issue costs future changes and tests, highest first. At equal cost, put design findings above Readability findings, and keep Readability to about ten findings so it doesn't bury the rest. Group related renames into one finding. For each finding give:

- location (`file:line`)
- the standards section it falls under
- what's wrong, in the vocabulary terms
- the concrete consequence: the change, test or caller that pays for it
- the smallest fix that resolves it

When the `ReportFindings` tool is available, report through it once, with `category` set to the kebab-cased standards section (`deep-modules`, `inject-dependencies`, `every-assertion-must-fail`, ...), `summary` as the defect and `failure_scenario` as the consequence. Otherwise write the findings as a numbered list.

In codebase mode at package or repo scale, write the report as prose rather than `ReportFindings`, since these problems don't live on one line. Order it as:

1. Anything urgent found along the way (a security hole, data loss), briefly, pointing to /code-review.
2. The problem statement and constraints from step 5, in a short paragraph, so the reader can disagree with them.
3. The design decisions from step 5, the ones worth revisiting first, then a line or two on those that hold up.
4. The architectural findings from step 4, each noting the decision it stems from, if any.

For the architectural findings, give the handful that matter most, usually three to seven, ranked. For each:

- the design decision or pattern, named in the vocabulary terms
- where it shows: a few representative locations, not every instance
- what it costs: the kinds of change that touch too many files, the tests that can't be written or have to patch internals, the bugs it invites
- the direction to move, and a first step small enough to land on its own

Leave out everything below the bar, including a count of it. If the codebase has no big-ticket problems, say so rather than filling the list.

Close with at most two lines: sections that came up clean, and anything you couldn't judge without more context. If nothing survived verification, say so plainly.
