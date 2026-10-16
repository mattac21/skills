# Coding standards

These apply in any language. The goal is code whose behavior can be tested through its
interfaces, without reaching into its internals.

## Modules

### Prefer deep modules

A module should hide a lot of behavior behind a small interface. Callers should not need to know
how it authenticates, paginates, retries, formats requests, or names its storage keys. A module
whose interface is about as complex as its implementation, such as a pass-through wrapper or a bag
of small helpers that callers must sequence correctly, adds cost without hiding anything.

- Give each external system (an API, a database, a queue, a file format) one owning module.
  Everything that talks to that system goes through it, so auth, errors and pagination are
  written once.
- If callers must call functions in a set order, or build keys, paths or IDs by hand to use a
  module, that knowledge belongs inside the module.
- Name operations after what they mean in the domain (`reserve_username(name)`), not after the
  mechanism (`put_if_absent(key)`).

### One responsibility per module

A module should have one reason to change. It has several if its name no longer describes half of
what it does, if one function both decides what should happen and carries it out across several
systems, or if unrelated callers use unrelated halves of it.

- Keep transport (HTTP handlers, CLI parsing, message consumers) separate from the work it triggers.
- Keep policy (what should happen, given this state) separate from orchestration (do this, then
  that).

### Abstractions must earn their place

Add an abstraction when it hides real complexity, or when a second implementation exists (a test
fake counts). Don't add one for a single caller with nothing to hide. The opposite failure is just
as common: the same concern written several times across modules because nothing owns it. Five ways
to build the same request headers means a module is missing.

A module that owns an external system usually earns a client class. When its functions share a
credential, an identity, a base URL or a cache, put them on an instance built once at startup and
passed in. The instance holds that state instead of module globals, and tests pass a fake client
instead of patching each function. The production client and the fake are the two adapters that
make the seam real.

## State

### Hide state and expose only what callers need

Each piece of state belongs to one module. Other modules reach it only through that module's
operations. Storage layout, key schemes, cache contents and serialization formats are private. If
two modules both know how a record's key is built, they will drift.

### Don't share mutable state across modules

Avoid module-level mutable globals such as caches, tokens, registries and locks. Keep that state on
an instance created at startup and passed to whatever needs it. Process-global state makes tests
order-dependent and stops two configurations from existing in one process.

### Separate mutation from access

A function that answers a question should not change anything. A function that changes state
should say so in its name and return little. Keep the places that write each piece of state few and
easy to find.

- Put decisions in pure functions that take the current state and return what to do. Put effects in
  a thin outer layer that gathers inputs, calls the decision, and carries it out (functional core,
  imperative shell).
- Don't store two concepts in one place for convenience, such as a lock and a result in the same
  record. Clearing one will silently clear the other.

### Lean toward immutable data where it pays off

Immutability is a preference, not a rule for every structure. Reach for it where it buys something:
values that several holders share, values that are cached or used as keys, and records that pass
between modules. In those places, a frozen record or a changed copy means nothing can change behind
a holder's back, and the places that write state stay few.

Mutable structures are fine where one owner holds them and mutation is the natural model: a list
built up before it's returned, an accumulator in a loop, or state that a single object owns and
changes through its own methods. Converting these adds ceremony and gains nothing.

- When going immutable would cost performance, look for a way to keep it before giving it up:
  - Memoize derived values computed from immutable inputs. This is safe because the inputs can't
    change.
  - Use structures that share structure between versions instead of copying.
  - As a last resort, mutate a private copy inside one function and freeze it on the way out.
- A memoization cache is mutable state too. Keep it on an instance or on a pure function of
  immutable arguments, never as a module-level dict that several callers write to.

### Write against current state, not a snapshot

When updating shared state after a long-running operation, re-read it first or use a conditional
write (compare-and-set). Writing back a copy taken before the operation overwrites everything that
changed in between, including changes made by people.

## Types

### Give each concept one owner, not many primitives

When a concept is handled as bare primitives (strings, tuples, raw API dicts), every caller has to
relearn its format, and values with different meanings can be swapped without anything noticing.
Give the concept one owner that parses it once and gives callers something they can't misuse.

That owner doesn't have to be a type named after the concept. Sometimes the right owner is a value
type for the concept itself, such as an email address that knows its domain. Other
times it's an internal type that handles the concept as part of its job, so callers never see the
concept at all. For example, a git working-directory type that owns the
`refs/remotes/<remote>/<branch>` layout is better than a ref type that every caller has to build.
Choose by what callers do with the concept. If they hold it and pass it around, give it a type. If
they only need something done with it, hide it inside the module that does it.

- Signs that a concept has no owner:
  - It is parsed or formatted in more than one place (a regex, an f-string key, a dict shape).
  - Several parameters of the same primitive type, with different meanings, are passed side by
    side.
  - Raw payloads from an external system get passed beyond the module that talks to that system.
  - A map, dict, untyped object or tuple whose keys or positions are read in more than one
    function, even inside the module that fetched it. A reader can't tell what it holds without
    tracing every producer. Parse an external payload into a typed record where it's fetched,
    keeping only the fields the code uses, and never use an untyped map for a shape the code builds
    itself.
- Preventing a mix-up is reason enough for a type, even one with no behaviour. A thin wrapper or
  `NewType` is often enough.
- Parse at the edge. Once a value has been accepted, nothing downstream should parse or validate
  it again.

### Make invalid states unrepresentable

Shape types so that a state that must never happen can't be built. Guarding against it at every
use is the alternative, and every guard is a place it can be forgotten.

- Fields that are only valid together belong in their own type, held as one optional field rather
  than as several optionals that must agree. Split into separate variants (a union of records, an
  enum with data) only when several fields depend on the mode or the code paths diverge. If a type
  only ever represents one mode, make the fields required and refuse the other mode where the
  value is parsed.
- A `None` that callers have to check again and again is a sign of a missing type. Resolve the
  absence once, at the edge: return a type that always has the value, or a variant that says why
  it doesn't. Then nothing downstream needs to check.
- Prefer an enum or variant over combinations of booleans, and over strings with a few allowed
  values.
- A constructor or parser should refuse invalid input, so that holding a value proves it's valid.
- Where the language can't express a constraint, enforce it once, in the constructor, rather than
  at every use.

## Dependencies and configuration

### Inject dependencies

A unit should receive its collaborators as parameters or constructor arguments, not construct or
look them up itself. Collaborators include clients, stores, executors, models and clocks. Default
wiring belongs in the outermost entrypoint; below that, pass things in.

- Time, randomness and "today" are dependencies. If behavior depends on the date, the date is an
  input.
- Don't add a `from_env`-style wrapper next to a function so that it can build its own
  collaborators. The entrypoint builds them and calls the function directly.

### Read configuration once, at the edge

Read environment variables, flags and config files in the entrypoint. Build a typed settings object
from them and pass it, or the clients built from it, down. Don't read the environment deep inside
logic. Never copy a config value into a module-level constant at load time: later changes won't
reach the copy, and tests can only change it by starting a new process.

### No side effects at load time

Loading a module should not open connections, construct clients, start subprocesses, read large
files, or require credentials. Do that in a factory called at startup. Otherwise the module can't be
loaded in a test without the real environment.

### Add seams to the product, not workarounds to tests

If a test can't drive a behavior through the product's public interface, change the product to
accept what the test needs. Examples are a CLI with no way to set the date, or a function that
builds its own client. A test that reaches around the interface, re-implements part of the product,
or patches internals is testing a different program.

## Errors

- Don't reduce a failure to a log line on a path that affects outcomes. If work must continue, record
  the failure somewhere observable, such as a returned error list or a status the caller can check,
  so both operators and tests can see it.
- Give errors enough context to act on, and never include secrets.

## Readability

### Names mean one thing

- Within a module, a name refers to one kind of value. Using `target` for a repo name in one
  function, a dict of branch heads in another and a ref string in a third makes every reader
  re-derive it.
- Name things after what they are in the domain, not after their mechanism or their place in a
  call chain. A family like `sync` / `_sync` / `_apply` doesn't say which level does what.
- No single-letter names: every variable, parameter, loop variable, lambda or closure parameter,
  and caught error says what it holds (`message`, not `m`; `error`, not `e`). The only exception is
  the language's discard name (`_`) for a value deliberately ignored.
- Prefer a clear longer name over a terse one. A one-word helper whose object is implicit
  (`_find`, `_record`, `_ls`) makes every call site look the same.

### Functions do one explainable thing

A function should be explainable in one sentence. Signs that it isn't:

- several try blocks, each with its own failure rule
- raising an exception only to catch it a few lines later
- variables that are only assigned on some branches and then read after the branches end
- a parameter whose `None` means "a different mode", or a boolean that switches between two
  behaviours (usually two functions)
- an argument the function fills in for its caller (return the value instead)

Keep cyclomatic complexity low. Every branch, loop, switch or match case, catch clause, conditional
expression, filter clause and short-circuit boolean operand is a decision point. Above about 10 a
function is too complex to read or test as a unit, whatever its length, so split it before adding
to it. The usual
split pulls each decision into a pure function (is this input valid? which record wins? what goes
in the reply?) and leaves the original as a short sequence of named steps, each with
one failure rule. Entrypoints and handlers are not exempt; they are the functions most likely to grow.

### Document every function that isn't trivial

Every function gets a doc comment in the language's convention (a docstring, JSDoc, a Go doc
comment, `///`), private helpers included, unless it is a one-liner or so simple that its name and
signature say everything: a straight-line body with no branches, loops, error handling or side
effects beyond the obvious. Anything with a branch, a loop, error handling, a fallback, an external
call or a side effect qualifies. The doc comment says what the function returns or changes
and anything a caller can't see from the signature: its fallbacks, its error modes, the state it
touches, and the shape of any input it expects.

## Testing

### Test behavior through public interfaces

A test should describe what the system does from the outside: given these inputs or events, these
observable results. It should survive any refactor that keeps the behavior. Asserting which internal
functions were called, with which arguments, or in what order ties the test to the current
implementation. Such tests fail on harmless changes and pass on real bugs.

### Fake at the boundary of what the test is about

Decide what the test is checking, then replace only what lies outside that.

- An end-to-end or integration test checks the whole system, so replace only external systems and
  sources of nondeterminism (remote services, the clock, randomness, a language model). Everything
  you own runs for real. If a test double there has to re-implement product logic or conventions,
  such as naming rules, formats or branching, the boundary is in the wrong place. Move it out to the
  external dependency.
- A unit test scoped to one module's business logic checks that module, so faking its internal
  collaborators is fine, even many of them. Pass the fakes in through the interfaces the module
  already depends on, and keep each fake as simple as the test needs. Assert on the module's outcome
  (its return value, or the state it left in the fakes), not on the sequence of calls it made.
- A fake replaces only the external system, never code you own on the far side of it. If work you
  own runs inside the external system (a job inside a build service), the fake calls that real code
  with an injected stage, rather than re-implementing it.
- Fakes produce what the real system would send, such as webhook payloads and API responses, and
  the product parses it with its own parser. A fake that hands the product already-parsed objects
  skips the edge the test should cover.
- Give a fake a typed configuration API, with one method per operation it stands in for and taking
  the domain type that operation returns. A single catch-all type, or a raw dict of outcomes keyed
  by name, lets a test configure answers that can't happen.
- Fakes are code. Keep each one as small as its actual usage, delete parts nothing exercises, and
  check fakes of external systems against the real thing where possible (contract tests).

### Drive end-to-end tests one event at a time

Don't hide the run behind a helper that pumps every event until the system goes quiet. The test
delivers each event itself, at the point in the story where it happens. Before each delivery it sets
up only what the external systems will answer to that event. After it, it asserts the state the
system left behind. A failure then points at the step that broke. A reader can see which input
caused which result, and the test reads as the story it checks.

- Each test owns its data and its answers. It creates its own records (the ticket, the repo state)
  and says what each external call returns. There are no shared defaults that a test silently
  inherits. Repeating a few lines of setup across tests is better than a helper that hides what the
  story depends on.
- Helpers are for queries and assertions named in domain terms (`launched()`, `labels(ticket)`),
  not for steps of the story.
- A short comment on each step is welcome here. It says what is being mocked, which event is being
  processed and what is being ensured, and why. This is the one place comments narrate. The test is
  the spec, and the why of each step isn't in the code.

### Every assertion must be able to fail for a real bug

Before keeping an assertion, name the product bug that would make it fail. Delete or rewrite it if:

- it checks a value the test produced itself, such as text a fake wrote or an ID the test chose;
- the fixture already guarantees it, such as a permission the environment never grants or a path
  that can't emit what is being checked for;
- it checks that something exists when the behavior is about what it contains;
- it pins incidental detail (formatting, internal names, message wording) instead of behavior;
- it repeats a cheaper test's check at a more expensive level;
- it asserts on the fake's own record of what happened (a transcript of deliveries or calls) rather
  than on the state the product left in the external system. That is the fake asserting the fake.

Use mutation testing to find assertions that never fail: break the product on purpose and confirm a
test catches it.

### Test at the cheapest level that can catch the bug

Pure decision logic gets fast unit tests with no doubles. Orchestration gets in-process tests with
in-memory fakes of its injected dependencies. End-to-end tests cover a few full stories across real
process boundaries, plus the integrations nothing smaller can reach. If a unit test has to patch
internals to reach the logic it wants, the code needs its decisions separated from its effects, or
its dependencies passed in. More patches won't fix that.

### Don't sleep; wait for observable conditions

Tests should wait for a specific condition, with a timeout, such as a record existing or a status
changing. A sleep either wastes time or hides a race. If a test only passes with a sleep, there is an
ordering bug in the product or the test. "Nothing happened for N seconds" does not prove work has
finished; give the system a way to report that it is idle.

### Keep tests readable as specifications

Name tests after the behavior they check. Wrap repeated queries in helpers named in domain terms,
so a test reads as a sequence of events and expectations rather than raw API calls. Keep the setup
explicit and close to the assertions it supports.
