# Effect V4 Checkpoints

Use this reference to select questions and remediation for the `effect-v4-tutor` skill. It is a map of concepts, common misconceptions, and question shapes. Prefer local repo docs and Effect source for exact API facts.

## Answer Formats

Vary formats across a checkpoint so the user practices recognition, discrimination, explanation, and implementation.

- Free text: best for mental models and tradeoffs.
- ABCD single-choice: best for distinguishing one correct pattern from plausible anti-patterns.
- Multi-select: best when several statements are true and you want to expose partial understanding.
- Code/repair: best for checking whether the user can apply a convention in implementation.

Ask for a short reason with ABCD and multi-select questions. Grade the reasoning, not just the selected letters.

Example ABCD:

```text
Which boundary is the right place to unwrap a `Config.redacted` API key for an SDK client?

A. Inside every domain function that needs to make a request
B. In the SDK adapter/layer where the concrete client is constructed
C. At module top level during import
D. In the logger so it can redact the value later

Reply with one letter and a one-sentence reason.
```

Example multi-select:

```text
Select all statements that fit this repo's Effect v4 style:

A. Expected domain failures should use typed errors in the Effect error channel.
B. `Effect.runPromise` belongs inside reusable service helpers whenever you need the value.
C. External JSON should be decoded at the boundary before entering domain code.
D. Tests should usually replace service layers rather than mocking deep imports.

Reply with the letters and a short reason for one choice you included and one choice you excluded.
```

## Core Mental Model

Effect code makes three things explicit:

- Success: `A`, the value produced by the computation.
- Failure: `E`, typed recoverable errors.
- Requirements: `R`, services or environment needed to run.

The implementation habit is to move uncertainty to boundaries, represent it in types, and keep runtime execution at entry points.

Useful first question:

```text
In `Effect.Effect<User, UserNotFoundError, UserRepository>`, what does each type parameter mean, and which one disappears when you provide a live repository layer?
```

## Typed Errors

Checks understanding of failures as data and defects as unexpected bugs.

Common misconceptions:

- Using `throw new Error` for domain failures.
- Catching broadly with `try/catch` inside Effect code.
- Treating all failures as strings or `unknown`.
- Splitting error classes before callers can recover differently.

Question shapes:

- Ask the user to choose between `Effect.fail`, `Effect.try`, and `Effect.tryPromise` for a concrete throwable or rejecting operation.
- Show a `throw new Error("not found")` snippet and ask for the Effect v4 replacement.
- Ask when an error deserves its own tag versus being part of a broader adapter error.

Remediation move:

- Explain that recoverable failures belong in the error channel, while defects are unexpected bugs.
- Show `Schema.TaggedErrorClass` for public or cross-module errors.
- Contrast `Effect.catchTag` with broad fallback handling at boundaries.

## Option And Absence

Checks whether the user separates absence from failure.

Common misconceptions:

- Returning `null` or `undefined` from domain code.
- Calling `Option.getOrThrow` to escape back to exceptions.
- Treating "not found" as always an exception instead of asking whether absence is expected.

Question shapes:

- Ask whether a missing optional profile image should be `Option` or a tagged error.
- Ask the user to consume an `Option` with `Option.match`.
- Show a nullable database field crossing a boundary and ask where to convert it.

Remediation move:

- Boundary code may receive nullish data; domain code should see `Option` when absence is expected.
- Failure is for cases the workflow cannot proceed through without recovery.

## Schema And Boundary Decoding

Checks source validation and wire/domain separation.

Common misconceptions:

- Trusting SDK, HTTP, or database payloads because TypeScript types exist.
- Using `JSON.parse` directly in application code.
- Keeping decoded data as loose `unknown` or raw SDK shapes.
- Adding a `Schema` suffix to every schema name.

Question shapes:

- Ask where `Schema.decodeUnknownEffect` belongs in an HTTP or SDK adapter.
- Ask the user to model a payload with `Schema.Class`.
- Show a `JSON.parse` snippet and ask for the local convention.

Remediation move:

- External data is unknown at runtime. Decode once at the boundary, then pass validated domain shapes inward.
- Prefer named `Schema.Class` models where local conventions call for them.

## Services And Layers

Checks dependency modeling and layer wiring.

Common misconceptions:

- Using globals, singletons, or direct constructors inside domain functions.
- Using older `Context.Tag` patterns when local v4 code expects `Context.Service`.
- Confusing the service interface with the live implementation layer.
- Running effects inside service methods instead of returning Effects.

Question shapes:

- Ask what `R` means before and after `Effect.provide`.
- Ask where SDK client construction belongs.
- Show a function that imports a concrete repository directly and ask how to invert it into a service.
- Ask why replacement layers make tests simpler.

Remediation move:

- `Context.Service` names a dependency. `Layer` constructs or provides it. Business logic uses `yield* ServiceName` and does not know the live implementation.

## Boundaries And Runtime

Checks where native APIs and Effect runtime calls belong.

Common misconceptions:

- Calling `Effect.runPromise` deep inside application services.
- Reading `process.env`, `Date.now`, or `Math.random` directly in domain code.
- Using raw `fetch` where repo conventions call for Effect HTTP/platform services.

Question shapes:

- Ask where runtime execution should happen in a CLI/API/worker.
- Ask how to read secret config without leaking redacted values.
- Ask why `Clock` or `Random` helps tests.

Remediation move:

- Runtime edges run effects. Interior code returns Effects and receives platform capabilities through services.

## Control Flow And Collections

Checks expression-oriented branching and explicit concurrency.

Common misconceptions:

- Imperative loops in domain transformations.
- Native `switch` or if/else ladders over modeled states.
- Using `Promise.all` inside Effect code.
- Forgetting to make concurrency policy explicit.

Question shapes:

- Ask how to traverse an array effectfully and limit concurrency.
- Ask how to branch over a tagged union with `Match`.
- Ask why `Arr.match` can be clearer than manual length checks.

Remediation move:

- Prefer Effect and Effect data modules so control flow, failure, and concurrency remain visible in the type and code shape.

## Testing

Checks whether the user can test Effect code without breaking architecture.

Common misconceptions:

- Mocking deep imports instead of providing replacement layers.
- Running effects manually in the middle of tests where Effect test helpers exist.
- Depending on live SDKs for ordinary domain tests.

Question shapes:

- Ask how to test a service that needs `UserRepository` and `Clock`.
- Ask where a fake layer should be provided.
- Ask which tests should hit a sandbox SDK and which should use replacement layers.

Remediation move:

- Test the domain with fake or in-memory service layers. Reserve live SDK/API tests for narrow adapter behavior.

## Observability

Checks tracing and structured logging habits.

Common misconceptions:

- Using `console.log` inside application code.
- Logging broad failures without span context.
- Losing operation names by wrapping everything in anonymous helpers.

Question shapes:

- Ask where to add `Effect.withSpan` or `Effect.fn("Name")`.
- Ask what information belongs in structured log fields.
- Ask why broad boundary catches should be logged.

Remediation move:

- Named effects, spans, and structured logs make typed control flow visible at runtime.

## SDK And Adapter Wrappers

Checks wrapping Promise-based clients and preserving interruption, errors, config, and decoding.

Common misconceptions:

- Exposing a raw SDK client throughout the app.
- Ignoring `AbortSignal` even when the SDK supports it.
- Mirroring every SDK method instead of writing app-facing operations.
- Letting raw SDK output cross into domain code.

Question shapes:

- Ask where to construct the SDK client and where to unwrap secrets.
- Ask why a callback `use` method can be safer than raw client access.
- Ask when to expose convenience methods over `use`.

Remediation move:

- Keep the SDK at the boundary. Wrap construction and calls in typed Effects, thread interruption, decode weak output, and expose application language above the SDK.

## Retake Design

Retakes should test the same missed concept through a different scenario.

Examples:

- If the user missed typed errors, switch from HTTP not found to config parse failure.
- If the user missed layers, switch from repository service to clock/config service.
- If the user missed Schema, switch from HTTP JSON to SDK response payload.

End retakes when the user can explain the why and apply the pattern to a new example.
