# Global Elixir/OTP Conventions

These conventions are distilled from production Elixir codebases and must be followed in all Elixir projects.

## Project Structure

- Prefer umbrella apps when the system has distinct bounded contexts. Each app should have a single responsibility.
- Separate the web layer, data layer, and shared utilities into distinct umbrella apps.
- Schema modules nest under their context: `MyApp.Users.User`, `MyApp.Groups.Group`.

## Context Modules

- Context modules are the public API for a domain. All external callers go through the context, never directly to schemas or repos.
- Keep context functions focused — orchestrate calls, don't embed query logic.
- Delegate CRUD to `EctoShorts.Actions` where it fits. Write explicit queries only when needed.

## Result Tuples

- Every function that can fail returns `{:ok, value} | {:error, reason}`.
- Define type aliases for common result shapes: `t_res(schema)`, `find_res(schema)`.
- Use `ErrorMessage` structs for domain errors: `%{code: atom, message: String.t(), details: any()}`.

## `with` Expressions

- Use `with` for sequential operations where each step depends on the previous — it replaces nested `case` with a flat chain.
- **`<-` is a sibling to `=`:** `=` raises `MatchError` on non-match, `<-` aborts the chain and returns the non-matched value as-is. This "fall-through" behavior is the key design property — a bare `with` (no `else`) naturally propagates the first failure.
- **Never put plain `=` bindings inside a `with` chain.** Every line in `with` must use `<-` and participate in the fall-through control flow. An `=` line is not doing any pattern-match-or-abort work, it's just a local binding being smuggled into the chain to set up arguments for the next `<-` step. That's a misuse of the macro — the `=` line is not taking advantage of the control flow `with` exists to provide. Fix it by extracting a private composing function that bundles the infallible transform with the fallible step that consumes it, returning a single `{:ok, _} | {:error, _}` for the `with` to bind via `<-`:

```elixir
# BAD — argv = ... is a smuggled local binding, not a with step.
with :ok <- verify(),
     argv = normalize_argv(raw_argv),
     {:ok, opts} <- parse_options(argv) do
  ...
end

# GOOD — the composing helper does the normalize + parse together and
# returns a single fallible result the with can consume via <-.
with :ok <- verify(),
     {:ok, opts} <- parse_options(raw_argv) do
  ...
end

defp parse_options(raw_argv) do
  raw_argv |> normalize_argv() |> OptionParser.parse(strict: @spec) |> ...
end
```

The same rule applies to infallible *computations* threaded through the chain (e.g., `config = build_config(lib, op, opts)`): either bundle the compute into a preceding fallible helper so it emerges already wrapped in `{:ok, config}`, or pull it into the `do` block where plain `=` bindings are fine (the `do` block is normal Elixir code, not a `with` step).
- **Avoid `else` clauses.** The pitfall of `else` is that all failure clauses from every `<-` are flattened into a single block, obscuring which step actually failed. Prefer letting non-matches fall through.
- **Normalize return values in helpers, not in `else`.** When steps return structurally different shapes, wrap them in private functions that return a consistent shape (e.g., `:ok | {:error, reason}`). This keeps the `with` chain clean and makes each step's failure self-describing without needing `else` to sort it out:

```elixir
with :ok <- validate_extension(path),
     :ok <- validate_exists(path) do
  {:ok, Path.expand(path)}
end

defp validate_extension(path) do
  if Path.extname(path) == ".ex", do: :ok, else: {:error, :invalid_extension}
end

defp validate_exists(path) do
  if File.exists?(path), do: :ok, else: {:error, :missing_file}
end
```

- When wrapping in a transaction, use the project's transaction wrapper (e.g., `Repo.transact` if available) to get automatic rollback on `{:error, _}`. If using raw `Repo.transaction`, let the `with` fall through and call `Repo.rollback/1` on mismatch — this is one of the few acceptable uses of `else`.

## Changesets

- Declare `@required` and `@allowed` module attributes for field lists.
- Use `cast_embed` with dedicated changeset functions for nested data.
- Write custom validations as private functions.
- Separate create and update changesets when their rules diverge.

## Query Composition

- Define composable query functions on each schema module: `from/0`, `by_field/2`, `filter_by_x/2`.
- These return `Ecto.Query.t()` and compose via pipes.
- For multi-step transactions, prefer the project's transaction wrapper (e.g., `Repo.transact`) with `with` pipelines. Use `Ecto.Multi` when steps need named references for later steps or when you need partial rollback visibility.
- Reserve raw SQL (`Repo.query!/2`) for genuinely complex queries that Ecto's DSL cannot express.

## Schemas

- Use a project-level `use MyApp` macro that bundles `Ecto.Schema`, `Ecto.Changeset`, `Ecto.Query`, and any custom query function imports.
- Define `@type t :: %__MODULE__{}` on every schema.
- Use `@derive {Jason.Encoder, except: [...]}` to control serialization and prevent leaking sensitive fields.
- Use embedded schemas for structured data that doesn't need its own table.

## Aliases and Imports

- Group aliases at the top of the module, using multi-alias syntax:

```elixir
alias MyApp.{
  Repo,
  Users,
  Users.User,
  Groups.Group
}
```

- Import sparingly. Prefer explicit aliases over imports when the source module matters for readability.
- Use `require` only for macros (`require Logger`).

## Typespecs and Documentation

- Every public function gets a `@spec`.
- Use `@impl` on all behaviour callbacks.
- Write `@moduledoc` on all public modules. Use `@moduledoc false` on internal ones.
- Define `@type` and `@typedoc` for domain-specific types.
- Do not add `@doc` to every function — only where the name and spec aren't self-explanatory.

## GenServer and Agent Patterns

- Use Agents for simple state holders (feature flags, license data, error accumulators).
- Use GenServers for stateful services with message handling, timers, or PubSub subscriptions.
- Always define a client API in the same module — callers should never use `GenServer.call/cast` directly.
- Use `Task.Supervisor` for fire-and-forget async work. Name each supervisor by purpose.

## Supervision

- Default to `:one_for_one` strategy unless children have genuine dependencies.
- Use `DynamicSupervisor` for processes created at runtime (e.g., per-integration supervisors).
- Use `Registry` for named process lookup when the set of processes is dynamic.

## Caching

- Use a behaviour-based cache abstraction with adapters (Redis for prod, local/simple for test).
- Implement read-through caching: check cache first, fall back to DB, populate cache on miss.
- On mutations, evict affected cache keys — don't try to update them in place.
- Define TTLs explicitly. Never cache without an expiry.

## Error Handling

- Let processes crash and restart via supervision — don't over-rescue.
- Use `rescue` / `catch` only at system boundaries (plugs, channel handlers, GenServer callbacks).
- Never silently swallow errors. Log and re-raise, or return a result tuple.
- Validate at system boundaries (user input, external API responses). Trust internal code.

## Web Layer

- Use `action_fallback` to a centralized fallback controller for error rendering.
- Custom plugs for cross-cutting concerns: auth, request logging, validation, serialization.
- Keep controllers thin — pattern match assigns, call context, render.
- Use WebSocket channels for real-time features, not polling.

## Testing

- Use `ExUnit.CaseTemplate` for reusable setup (`DataCase`, `ConnCase`).
- SQL Sandbox for database isolation. Mark tests `async: true` where safe.
- Use Req's built-in test adapters/plugs for HTTP boundary testing. Use Mox for non-HTTP behaviour mocking.
- Test through the public context API, not internal functions.

## Observability

- Emit telemetry events for key operations (DB queries, HTTP calls, cache hits/misses).
- Use structured JSON logging in production.
- Integrate Sentry or equivalent for error tracking with stacktraces.
- Expose Prometheus metrics for infrastructure monitoring.

## Configuration

- Compile-time config in `config.exs` for static values.
- Runtime config in `runtime.exs` for environment-dependent values (DB URLs, secrets, feature flags).
- Access env vars through a utility function with defaults and type coercion — never raw `System.get_env` scattered through business logic.
- Per-app `Config` modules for accessing app-specific settings.

## Code Quality

- Enforce Credo for linting and Dialyzer for type checking.
- Use the Elixir formatter. No debates about style.
- Warnings as errors in CI.
