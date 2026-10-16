# Design vocabulary

Condensed from Matt Pocock's codebase-design skill
(https://github.com/mattpocock/skills/tree/main/skills/engineering/codebase-design), reconciled
with `~/.claude/CODING_STANDARDS.md`. Where the two differ, the standards win; the differences are
listed at the end.

## Terms

Use these exactly in findings. Don't substitute "component", "service", "API" or "boundary".

- **Module**: anything with an interface and an implementation, at any scale: a function, class, package or a slice across tiers.
- **Interface**: everything a caller must know to use the module correctly. That includes the type signature, but also invariants, ordering constraints, error modes, required configuration and performance characteristics. A module whose callers must call things in a set order has that order in its interface.
- **Implementation**: the code inside the module.
- **Depth**: leverage at the interface, i.e. how much behaviour a caller or test can exercise per unit of interface it has to learn. Deep means a lot of behaviour behind a small interface; shallow means the interface is nearly as complex as the implementation. Depth is not a ratio of line counts.
- **Seam**: the place where a module's interface lives, where behaviour can be altered without editing that place. Where to put the seam is its own decision, separate from what goes behind it.
- **Adapter**: a concrete thing that satisfies an interface at a seam (the production client, an in-memory fake).
- **Leverage**: what callers get from depth. One implementation pays back across every call site and test.
- **Locality**: what maintainers get from depth. Change, bugs and verification concentrate in one place.

## Tests for a finding

- **Deletion test.** Imagine deleting the module and inlining it. If complexity vanishes, it was a pass-through. If complexity reappears across its callers, it was earning its keep.
- **Adapter count.** One adapter means a hypothetical seam; two means a real one. A production client plus a test fake is two.
- **The interface is the test surface.** Callers and tests cross the same seam. A test that has to reach past the interface means the module is the wrong shape.
- **Internal vs external seams.** A deep module can have private internal seams used by its own tests. Exposing them through the interface just because a test wants them is a finding.

## Dependency categories

Use these to recommend how a deepened module should be tested.

1. In-process (pure computation, in-memory state): merge and test through the interface directly. No adapter.
2. Local-substitutable (a local stand-in exists, e.g. SQLite for a DB, a temp dir for a filesystem): test with the stand-in running. The seam stays internal.
3. Remote but owned (your own services over a network): define a port at the seam; production uses the network adapter, tests use an in-memory one.
4. True external (third-party APIs, an LLM): inject it as a port; tests pass a fake.

## Where this differs from the standards

- Pocock says "mock adapter" for true externals. The standards want a fake passed in, with tests asserting on outcomes or the state left in the fake, never on the sequence of calls.
- Pocock's "replace, don't layer" says to delete unit tests of shallow modules once tests exist at the deepened interface. That applies to tests of modules that were merged away. It does not remove unit tests of pure decision functions, which the standards keep as the cheapest level that can catch the bug.
