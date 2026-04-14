# Ruuk

*Pronounced "rook" — like the bird, or the chess piece.*

A statically typed, functional language for enterprise applications.

Ruuk makes enterprise workflows -- state transitions, failure modes, multi-step processes -- into compiler-verified structures. The type system catches the classes of bugs that arise most often in complex domains: invalid state sequences, unhandled outcomes, and silently dropped fields.

## Why a New Language?

I've built enterprise applications in many problem domains. Working in that space for years, I kept running into the same patterns -- state machine logic scattered across if/else chains with no compiler enforcement of valid transitions, methods that return on the happy path but fold several distinct failures into a single exception to "handle more specifically later," DTOs with manual field mappings between related types and no structural guarantee that every field was accounted for.

Any developer who has traced a production bug to a state transition that was never guarded, spent an afternoon auditing DTO mappings to find a silently dropped field, or written a catch block that collapses three legally distinct failure modes into a single log line -- knows this feeling. The discipline exists. The patterns are understood. But they live in code review comments, team wikis, and the institutional memory of whoever has been on the team longest. The compiler offered no help.

F# and Ionide -- its editor integration -- changed how I thought about development. As I typed, the compiler responded in real time: not just catching type mismatches, but making entire categories of errors impossible by construction. Unhandled cases, impossible states -- gone, structurally, before the code ran. It felt less like a tool checking my work and more like a collaborator who understood what I was trying to build.

That raised an obvious question: how far could this go? F# is powerful, but it doesn't know about enterprise domain patterns. It can verify that a function receives the right type -- it can't verify that a workflow handles every distinct outcome, or that an operation runs in a valid state. Those properties still lived in convention and code review. I wanted the same experience -- structural, compile-time, impossible-to-ignore -- for the patterns I kept enforcing by hand.

Ruuk is the result. It's built around the enterprise patterns I kept wishing I could express more directly, with verification that falls out of declarations you'd write anyway for clarity. When the code compiles, the structural properties hold -- not by convention, not by coverage targets, but by construction.

That turns out to matter beyond individual developer clarity. More code is written agentically today than ever before. A compiler that understands state transitions, exhaustive outcomes, and field accounting doesn't care how the code was produced -- it catches the same structural gaps whether the author was a person or an agent. The properties that help me write clearer code are the same properties that make agentic code safer to ship.

## How It Works

Ruuk is in the ML/F# tradition: statically typed, expression-oriented, immutable by default, with type inference and exhaustive pattern matching. If you've spent time with Elm, OCaml, or F#, you'll be productive quickly. On top of that foundation, Ruuk adds constructs that make enterprise domain modeling a first-class concern of the compiler.

### Operations with Exhaustive Outcomes

An operation declares every possible result. The compiler rejects any call site that doesn't handle all of them.

```ruuk
pub op createUser =
    payload req: CreateUserRequest
    to store: UserStore
    outcomes =
        | Created of user: User
        | AlreadyExists
        | ValidationFailed of errors: List<String>
```

At the call site:

```ruuk
createUserReq
|> createUser to userStore
|> on Created user           -> sendWelcomeEmail user
|> on AlreadyExists          -> returnConflict
|> on ValidationFailed errors -> returnBadRequest errors
```

Every outcome must be handled. A wildcard catch-all is a compile error. You cannot silently drop the `AlreadyExists` case.

This replaces the `Result<T, E>` pattern where failure is a single opaque type. In regulated domains -- healthcare, finance, government -- different failures have different legal and operational consequences. "Denied" and "denied with appeal rights" are not the same outcome. The compiler enforces that distinction.

### Projections and Field Accounting

Enterprise code is full of type variations: a `User` becomes a `CreateUserRequest` (without server-assigned fields), a `UserProfile` (without the password hash), a `UserSummary` (just name and role). Every manual mapping is a place where a field can be silently dropped or included when it shouldn't be. It's the software equivalent of double-entry bookkeeping -- except most languages don't require the books to balance.

```ruuk
pub type CreateUserRequest = User without { id, createdAt, passwordHash }
pub type UserProfile = User without { passwordHash }
pub type UserSummary = User only { id, name, role }
```

Projections are traced -- the compiler knows `UserProfile` derives from `User`. When you convert between related types, the compiler enforces **field accounting**: every field difference must be explicitly accounted for. Add a `referralCode` field to `User`, and the compiler tells you that `CreateUserRequest` hasn't decided whether to include or exclude it. No field is silently dropped. No field is silently added.

This is lightweight formal methods -- not theorem proving, but structural verification that falls out of declarations you'd write anyway for clarity.

### State Machines as Types

Resources move through typed states. The compiler tracks the current state and rejects invalid transitions.

```ruuk
pub resource PatientCase =
    state Initial
    state HistoryGathered
    state StatusAssessed
    state RadiologyComplete
    state ReadyForReport

pub op assessPatientStatus =
    payload history: PatientHistory
    performs PatientCase Initial -> StatusAssessed
    outcomes =
        | StatusComplete of ClinicalStatus
        | AssessmentFailed of reason: String

pub op analyzeRadiology =
    payload status: ClinicalStatus
    performs PatientCase StatusAssessed -> RadiologyComplete
    outcomes =
        | RadiologyAnalyzed of RadiologyReport
        | NoImagingAvailable
        | AnalysisFailed of reason: String
```

`analyzeRadiology` cannot run unless `PatientCase` is in `StatusAssessed` state. This is a compile error, not a runtime check. Code that calls operations out of order gets a compiler rejection -- not a silent bug that surfaces during an oncology review.

The same mechanism handles I/O resources: a database connection's state machine encapsulates the async lifecycle so it never leaks into your application code.

### Sagas with Declared Compensation

Multi-step workflows declare what happens when a step fails. Compensation isn't a try/catch someone might forget to write -- it's declared alongside the step, the way a database transaction declares its rollback behavior.

```ruuk
pub saga TumorBoardReview =
    step gatherPatientHistory
    step assessPatientStatus
    step analyzeRadiology        compensate archiveIncompleteCase
    step retrieveGuidelines
    step searchClinicalTrials
    step generateReport          compensate deletePartialReport
```

If `generateReport` fails after five steps have completed, `deletePartialReport` runs automatically. The compiler verifies that every step with side effects has a compensation path. No orphaned partial state. No "we'll add error handling later."

## What the Compiler Checks

These constructs address patterns that appear repeatedly in enterprise software and are difficult to enforce with conventional language tooling:

| Domain Pattern | How Ruuk Expresses It |
| --- | --- |
| Enforcing workflow step ordering | State machines reject invalid transitions at compile time |
| Exhaustive outcome handling | All outcomes declared; unhandled cases don't compile |
| Structural field mappings | Field accounting verifies every difference is intentional |
| Multi-step rollback behavior | Sagas declare compensation alongside each step |
| Domain intent as structure | Operations declare inputs, outputs, and state changes explicitly |

The compiler doesn't replace domain knowledge or careful design. It makes the structural properties of that design verifiable. The right types and declarations become the documentation. The compiler becomes the second pair of eyes.

## Status

Ruuk is in active development. The language design is stable. The compiler is pre-alpha and not yet public.

I write about the design decisions behind Ruuk and the broader case for language-level correctness in enterprise software:

- [Dev.to articles](https://dev.to/matweiss) -- deep dives into language design topics
- [Mat Weiss on LinkedIn](https://www.linkedin.com/in/matweiss)

## License

[TBD]
