---
name: effect-client-wrapper
description: Use when wrapping third-party SDK clients such as Stripe, Resend, AWS, Supabase, Prisma, Drizzle, or vendor packages with Effect v4 services. Provides the SDK callback use pattern, typed errors, tracing, retries, config, and layer-based dependency injection. Based on RhysSullivan/effect-client-wrapper-skill skill.
---

# Effect Client Wrapper Pattern

Wrap third-party SDK clients with Effect v4 using the callback `use` pattern. This skill is for SDK clients with their own client objects, not raw HTTP client wrappers.

## When To Use

Use this pattern for libraries that expose Promise-based APIs with many methods, need construction-time config or credentials, and do not already have a native Effect wrapper.

Examples: Stripe, Resend, AWS SDK, Google Cloud APIs, Supabase, Prisma, Drizzle, vendor exchange clients, generated SDK packages.

For one or two calls, an explicit service method may be enough. For broad SDKs, expose `use` plus a small set of convenience methods.

## Effect V4 Rules

- Define services with `Context.Service`, not `Context.Tag` or `Context.GenericTag`.
- Define public recoverable errors with `Schema.TaggedErrorClass`.
- Use `Effect.try` for synchronous SDK construction that can throw.
- Use `Effect.tryPromise` for SDK Promise calls that can reject.
- Thread `AbortSignal` from `Effect.tryPromise` into SDK calls when supported.
- Use `Config.redacted` for secrets; unwrap with `Redacted.value` only at the SDK boundary.
- Use `Effect.fn("Service.method")` for reusable service methods.
- Provide implementations with `Layer.effect` or `Layer.succeed`.
- Decode weak SDK output with `Schema.decodeUnknownEffect` before it enters domain code.
- Do not use `async` / `await`, `try` / `catch`, bare `throw`, or `new Error` in Effect application code.

## Canonical Pattern

```typescript
import { Config, Effect, Layer, Redacted, Schema } from "effect";
import * as Context from "effect/Context";
import { ThirdPartyClient } from "third-party-sdk";

export class MyClientError extends Schema.TaggedErrorClass<MyClientError>()(
  "MyClientError",
  { cause: Schema.Defect },
) {}

export class MyClientInstantiationError extends Schema.TaggedErrorClass<MyClientInstantiationError>()(
  "MyClientInstantiationError",
  { cause: Schema.Defect },
) {}

export class MyClient extends Context.Service<
  MyClient,
  {
    readonly use: <A>(
      fn: (client: ThirdPartyClient, signal: AbortSignal) => PromiseLike<A>,
    ) => Effect.Effect<A, MyClientError>;
  }
>()("MyClient") {
  static readonly layer = Layer.effect(
    MyClient,
    Effect.gen(function* () {
      const apiKey = yield* Config.redacted("MY_CLIENT_API_KEY");

      const client = yield* Effect.try({
        try: () => new ThirdPartyClient({ apiKey: Redacted.value(apiKey) }),
        catch: (cause) => new MyClientInstantiationError({ cause }),
      });

      const use = <A>(
        fn: (client: ThirdPartyClient, signal: AbortSignal) => PromiseLike<A>,
      ): Effect.Effect<A, MyClientError> =>
        Effect.tryPromise({
          try: (signal) => fn(client, signal),
          catch: (cause) => new MyClientError({ cause }),
        }).pipe(Effect.withSpan("MyClient.use"));

      return MyClient.of({ use });
    }),
  );
}
```

Usage:

```typescript
const program = Effect.gen(function* () {
  const myClient = yield* MyClient;
  return yield* myClient.use((client, signal) =>
    client.someMethod({ param: "value", signal }),
  );
}).pipe(Effect.provide(MyClient.layer));
```

The callback receives the SDK client and the Effect-owned `AbortSignal`. If the SDK does not accept a signal, intentionally ignore it and document that the underlying operation may continue after Effect interruption.

## Convenience Methods

Expose typed methods alongside `use` for operations your app calls frequently. Keep `use` for less common SDK calls.

```typescript
const createCustomer = Effect.fn("MyClient.createCustomer")(function* (
  input: CreateCustomerInput,
) {
  return yield* use((client, signal) =>
    client.customers.create({ ...input, signal }),
  );
});

return MyClient.of({ createCustomer, use });
```

Convenience methods should speak application language. Do not mirror the whole SDK method-by-method unless you actually need that surface.

## Decoding Weak SDK Output

If the SDK returns `unknown`, `any`, generated loose types, or source payloads that still need validation, decode at the adapter boundary.

```typescript
export class Customer extends Schema.Class<Customer>("Customer")({
  id: CustomerId,
  email: Schema.String,
  name: Schema.NullOr(Schema.String),
}) {}

const getCustomer = Effect.fn("MyClient.getCustomer")(function* (id: CustomerId) {
  const raw = yield* use((client, signal) => client.customers.get(id, { signal }));
  return yield* Schema.decodeUnknownEffect(Customer)(raw).pipe(
    Effect.mapError((cause) => new MyClientError({ cause })),
  );
});
```

The rest of the application should see validated domain shapes, not raw SDK payloads.

## Error Splitting

Keep one broad SDK error when callers cannot recover differently. Split errors only when `catchTag` branches are meaningful.

```typescript
export class MyClientRateLimited extends Schema.TaggedErrorClass<MyClientRateLimited>()(
  "MyClientRateLimited",
  { retryAfterSeconds: Schema.optionalKey(Schema.Int) },
) {}

const use = <A>(
  fn: (client: ThirdPartyClient, signal: AbortSignal) => PromiseLike<A>,
) =>
  Effect.tryPromise({
    try: (signal) => fn(client, signal),
    catch: (cause) =>
      isRateLimitError(cause)
        ? new MyClientRateLimited({ retryAfterSeconds: cause.retryAfterSeconds })
        : new MyClientError({ cause }),
  });
```

Only inspect SDK-specific error classes inside the adapter. Do not leak SDK error types into domain code.

## Retry

Retries belong around operations that are safe to retry. Do not retry non-idempotent writes unless the SDK/API provides idempotency keys or the operation is otherwise safe.

```typescript
import { Schedule } from "effect";

const retryPolicy = Schedule.exponential("100 millis").pipe(
  Schedule.both(Schedule.recurs(3)),
  Schedule.jittered,
);

const retryable = myClient.use((client, signal) =>
  client.readOnlyLookup({ id, signal }),
).pipe(Effect.retry(retryPolicy));
```

Prefer method-specific retries over retrying every SDK call globally.

## Raw Client Access

Do not expose the raw SDK client by default. It lets callers bypass typed errors, tracing, and interruption. Prefer `use` plus convenience methods. Expose `client` only for a concrete escape-hatch requirement.

## Stripe Sketch

```typescript
import Stripe from "stripe";
import { Config, Effect, Layer, Redacted, Schema } from "effect";
import * as Context from "effect/Context";

export class StripeClientError extends Schema.TaggedErrorClass<StripeClientError>()(
  "StripeClientError",
  { cause: Schema.Defect },
) {}

export class StripeClientInstantiationError extends Schema.TaggedErrorClass<StripeClientInstantiationError>()(
  "StripeClientInstantiationError",
  { cause: Schema.Defect },
) {}

export class StripeClient extends Context.Service<
  StripeClient,
  {
    readonly use: <A>(
      fn: (stripe: Stripe, signal: AbortSignal) => PromiseLike<A>,
    ) => Effect.Effect<A, StripeClientError>;
  }
>()("StripeClient") {
  static readonly layer = Layer.effect(
    StripeClient,
    Effect.gen(function* () {
      const secretKey = yield* Config.redacted("STRIPE_SECRET_KEY");
      const stripe = yield* Effect.try({
        try: () => new Stripe(Redacted.value(secretKey)),
        catch: (cause) => new StripeClientInstantiationError({ cause }),
      });

      const use = <A>(fn: (stripe: Stripe, signal: AbortSignal) => PromiseLike<A>) =>
        Effect.tryPromise({
          try: (signal) => fn(stripe, signal),
          catch: (cause) => new StripeClientError({ cause }),
        });

      return StripeClient.of({ use });
    }),
  );
}
```

Build app-facing services above the SDK wrapper, for example `Billing.createCustomer`, and return validated application shapes rather than raw `Stripe.Customer` where practical.

## Testing

Provide replacement layers instead of mocking SDK imports. Prefer replacing the application-facing service directly.

```typescript
export const BillingTest = Layer.succeed(Billing, {
  createCustomer: (email) =>
    Effect.succeed(new BillingCustomer({ id: BillingCustomerId.make("cus_test"), email })),
});
```

Do not create double-cast fake SDK instances. If you need to test a lower-level wrapper without the live SDK, introduce a narrow port for the part of the SDK you use and fake that port.

```typescript
type StripeBillingPort = {
  readonly createCustomer: (email: string, signal: AbortSignal) => PromiseLike<unknown>;
};

const makeStripeBillingPort = (stripe: Stripe): StripeBillingPort => ({
  createCustomer: (email, signal) => stripe.customers.create({ email }, { signal }),
});

const fakeStripeBillingPort: StripeBillingPort = {
  createCustomer: (email) => Promise.resolve({ id: "cus_test", email }),
};
```

Use SDK-backed fixture or sandbox tests only around adapter behavior you need to verify. Ordinary domain tests should not depend on a live SDK.

## Checklist

- Service uses `Context.Service`.
- SDK construction happens in `Layer.effect`.
- Secrets use `Config.redacted`.
- Construction errors are distinct from operation errors.
- Promise calls are wrapped with `Effect.tryPromise`.
- `AbortSignal` is threaded into SDK calls when supported.
- Public errors use `Schema.TaggedErrorClass`.
- Weak SDK output is decoded before entering domain code.
- Frequently used operations are exposed as named `Effect.fn` methods.
- Raw SDK client is not exposed unless there is a concrete need.
- Tests use replacement layers, high-level service fakes, or narrow ports.
