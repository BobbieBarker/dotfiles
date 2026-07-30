---
name: erlang-nif
description: >
  Erlang/OTP NIF development -- hand-rolled C NIFs, resource management,
  dirty schedulers, process ownership, and idiomatic Erlang module design.
  ALWAYS use when writing Erlang NIF bindings or C code for BEAM integration.
  ALWAYS use when designing Erlang APIs that wrap native libraries.
  ALWAYS use when debugging NIF crashes, resource leaks, or scheduler blocking.
---

# Erlang NIF Development

## 1. Rules for Writing Erlang NIFs (LLM)

1. **ALWAYS keep NIF calls under 1ms on normal schedulers.** A NIF that blocks a normal scheduler degrades VM responsiveness, causes bad load balancing, and can trigger extreme memory usage. Use dirty schedulers for anything longer.
   ```erlang
   %% In the ErlNifFunc array:
   {"heavy_compute", 1, heavy_compute_nif, ERL_NIF_DIRTY_JOB_CPU_BOUND}
   {"blocking_io",   1, blocking_io_nif,   ERL_NIF_DIRTY_JOB_IO_BOUND}
   ```

2. **NEVER let a NIF crash.** A crash in NIF code brings down the entire BEAM VM. Every C function must handle all error paths and return a proper error term, never segfault.
   ```c
   // BAD: unchecked pointer
   MyStruct* s = (MyStruct*)enif_alloc_resource(my_type, sizeof(MyStruct));
   s->data = get_data(env, argv[0]);  // could fail

   // GOOD: check every allocation and conversion
   MyStruct* s = (MyStruct*)enif_alloc_resource(my_type, sizeof(MyStruct));
   if (s == NULL) return enif_make_badarg(env);
   if (!enif_get_resource(env, argv[0], my_type, (void**)&data))
       return enif_make_badarg(env);
   ```

3. **ALWAYS use resource objects for opaque native state.** Never pass raw pointers as integers. Resources get garbage-collected with destructors and are type-safe across the NIF boundary.
   ```c
   // Resource type registration in load callback
   static ErlNifResourceType* CONN_RESOURCE;

   static int load(ErlNifEnv* env, void** priv, ERL_NIF_TERM load_info) {
       CONN_RESOURCE = enif_open_resource_type(
           env, NULL, "quic_conn",
           conn_destructor,        // cleanup callback
           ERL_NIF_RT_CREATE,
           NULL);
       return CONN_RESOURCE ? 0 : -1;
   }
   ```

4. **ALWAYS release resources after making the Erlang term.** `enif_make_resource` does not transfer ownership. Release immediately to let the GC manage lifetime, or retain deliberately with documented reason.
   ```c
   ERL_NIF_TERM make_conn(ErlNifEnv* env, quiche_conn* conn) {
       ConnResource* r = enif_alloc_resource(CONN_RESOURCE, sizeof(ConnResource));
       r->conn = conn;
       r->closed = 0;
       ERL_NIF_TERM term = enif_make_resource(env, r);
       enif_release_resource(r);  // GC owns it now
       return term;
   }
   ```

5. **ALWAYS protect mutable shared state with mutexes.** A NIF is thread-safe only if it acts as a pure function on its arguments. Shared state (static variables, `priv_data`, mutable resources) needs `ErlNifMutex`.
   ```c
   typedef struct {
       ErlNifMutex* lock;
       quiche_conn* conn;
       int closed;
   } ConnResource;

   static void conn_destructor(ErlNifEnv* env, void* obj) {
       ConnResource* r = (ConnResource*)obj;
       enif_mutex_lock(r->lock);
       if (!r->closed && r->conn) {
           quiche_conn_free(r->conn);
           r->closed = 1;
       }
       enif_mutex_unlock(r->lock);
       enif_mutex_destroy(r->lock);
   }
   ```

6. **PREFER `enif_send` for async results from background threads.** Background threads must not call most NIF API functions directly. Create a message environment with `enif_alloc_env`, build the result, send via `enif_send`, then free the env.
   ```c
   // From a background thread:
   ErlNifEnv* msg_env = enif_alloc_env();
   ERL_NIF_TERM msg = enif_make_tuple2(msg_env,
       enif_make_atom(msg_env, "quic_data"),
       enif_make_binary(msg_env, &bin));
   enif_send(NULL, &owner_pid, msg_env, msg);
   enif_free_env(msg_env);
   ```

7. **USE `enif_consume_timeslice` for medium-length work.** If work is 0.1-1ms, call this to hint at reduction consumption. Returns 1 when the timeslice is exhausted; yield promptly.
   ```c
   for (int i = 0; i < n; i++) {
       process_item(items[i]);
       if (enif_consume_timeslice(env, 5)) {  // 5% per item
           // reschedule remaining work via enif_schedule_nif
           break;
       }
   }
   ```

8. **USE yielding NIFs for divisible long-running work.** Break into chunks with `enif_schedule_nif` to return control to the VM between iterations. Preferred over dirty NIFs when work is naturally divisible.

9. **ALWAYS design the Erlang API first.** Write the `-spec` and `-type` definitions, decide the message shapes and ownership model, then implement the C to match. The Erlang API is the product; C is the implementation detail.

10. **ALWAYS provide Erlang stub functions.** Every NIF needs an Erlang fallback that calls `erlang:nif_error/1`. This is mandatory for the module to compile before the NIF loads.
    ```erlang
    -spec connect(config_handle(), binary()) ->
        {ok, conn_handle()} | {error, atom()}.
    connect(_Config, _ServerName) ->
        erlang:nif_error(nif_library_not_loaded).
    ```

11. **PREFER tagged tuples for NIF return values.** Follow the `{ok, Value} | {error, Reason}` convention. Use `enif_make_tuple2` with atom tags. For void operations that cannot fail, return a bare `ok` atom.
    ```c
    // Success with value
    return enif_make_tuple2(env,
        enif_make_atom(env, "ok"),
        make_conn(env, conn));

    // Error with reason
    return enif_make_tuple2(env,
        enif_make_atom(env, "error"),
        enif_make_atom(env, "handshake_failed"));

    // Void success
    return enif_make_atom(env, "ok");
    ```

12. **NEVER use `-import`.** Always use fully qualified `module:function` calls. Imports hide the call origin and break xref analysis.

13. **USE process-based resource ownership.** Monitor the owning process from the NIF. When the owner dies, clean up native resources. This is the quicer pattern and the correct BEAM-native lifecycle model.

14. **PREFER maps over records for public API option types.** Records are compile-time constructs tied to header files. Maps are runtime, composable, and don't create cross-module coupling. Use records only for internal state.

15. **CLASSIFY dirty NIF types correctly.** CPU-bound work (crypto, parsing, compression) uses `ERL_NIF_DIRTY_JOB_CPU_BOUND`. I/O-bound work (socket ops, file I/O, blocking FFI) uses `ERL_NIF_DIRTY_JOB_IO_BOUND`. Misclassification starves schedulers.


## 2. Erlang Module Structure for NIF Libraries

### Module Organization Pattern (from quicer)

```
src/
  mylib.erl              % Public API (facade module)
  mylib_nif.erl          % NIF stubs + on_load + library loading
  mylib_connection.erl   % Connection process / behaviour
  mylib_stream.erl       % Stream process / behaviour
  mylib_lib.erl          % Internal helpers, option normalization
  mylib_app.erl          % Application behaviour
  mylib_sup.erl          % Top-level supervisor

c_src/
  mylib_nif.c            % NIF entry points, resource types, load/upgrade
  mylib_connection.c     % Connection-specific NIF implementations
  mylib_stream.c         % Stream-specific NIF implementations
  mylib_config.c         % Configuration NIF implementations
```

### NIF Module Pattern

```erlang
-module(mylib_nif).
-on_load(init/0).

%% Public NIF entry points
-export([connect/3, send/3, recv/2, close/1]).

%% Types
-export_type([conn_handle/0, stream_handle/0, config_handle/0]).
-opaque conn_handle()   :: reference().
-opaque stream_handle() :: reference().
-opaque config_handle() :: reference().

-type conn_opts() :: #{
    server_name := binary(),
    alpn_protos := [binary()],
    idle_timeout => non_neg_integer(),
    max_streams_bidi => non_neg_integer()
}.

init() ->
    PrivDir = case code:priv_dir(mylib) of
        {error, bad_name} ->
            filename:join(filename:dirname(filename:dirname(
                code:which(?MODULE))), "priv");
        Dir -> Dir
    end,
    erlang:load_nif(filename:join(PrivDir, "mylib_nif"), 0).

%% Stubs -- replaced when NIF loads
-spec connect(config_handle(), binary(), conn_opts()) ->
    {ok, conn_handle()} | {error, atom()}.
connect(_Config, _ServerName, _Opts) ->
    erlang:nif_error(nif_library_not_loaded).
```

### Facade Module Pattern

```erlang
-module(mylib).

%% The public API. All external callers use this module.
%% Delegates to mylib_nif for native calls, wraps with
%% Erlang-side option normalization and message reception.

-export([connect/2, connect/3, send/2, recv/2, close/1]).

-spec connect(binary(), conn_opts()) ->
    {ok, conn_handle()} | {error, atom()}.
connect(ServerName, Opts) ->
    connect(ServerName, Opts, 5000).

-spec connect(binary(), conn_opts(), timeout()) ->
    {ok, conn_handle()} | {error, atom()}.
connect(ServerName, Opts0, Timeout) ->
    Opts = mylib_lib:normalize_conn_opts(Opts0),
    case mylib_nif:async_connect(ServerName, Opts) of
        {ok, ConnHandle} ->
            receive
                {mylib, connected, ConnHandle, Props} ->
                    {ok, ConnHandle};
                {mylib, connect_failed, ConnHandle, Reason} ->
                    {error, Reason}
            after Timeout ->
                mylib_nif:close(ConnHandle),
                {error, timeout}
            end;
        {error, _} = Err ->
            Err
    end.
```


## 3. Erlang Coding Conventions (Inaka Style Guide)

### Source Layout
- 2-space indentation, spaces over tabs
- 100 characters per line maximum
- No trailing whitespace
- Group exported functions first, unexported after
- Types at the beginning of the module

### Naming
- **Modules:** lowercase with underscores: `mylib_connection`
- **Functions:** lowercase with underscores: `start_stream`
- **Variables:** CamelCase, no underscores: `ConnHandle`, `StreamOpts`
- **Atoms:** lowercase with underscores: `handshake_failed`
- **Macros:** ALL_UPPER_CASE (use sparingly, prefer functions)
- **Records:** lowercase, prefixed with module context: `#conn_state{}`
- **State records:** Named `#mod_state{}` with `-type state() :: #mod_state{}` for OTP behaviours

### Control Flow
- **Prefer function clauses over `case`:** pattern-match in clause heads
- **Avoid `if`:** use `case` or function clauses instead
- **Avoid nested `try...catch`:** isolate error handling at boundaries
- **Avoid `throw`/`catch` for flow control:** use tagged tuples
- **Avoid deep nesting:** max 3 levels; extract helper functions
- **Prefer higher-order functions over manual recursion:** `lists:map`, `lists:foldl`, list comprehensions

### Module Design
- No god modules: one responsibility per module
- No `-import`: always fully qualify external calls
- No `-compile(export_all)`: export only the public API
- No macros for module or function names
- No type definitions in header files: use `-export_type` from the defining module
- No record definitions in header files: keep records internal, expose opaque types
- Encapsulate OTP server APIs: never `gen_server:call` across module boundaries

### Types and Specs
- `-spec` on every exported function, unexported when it adds value
- `-type` and `-opaque` for all domain types
- `-export_type` for types used outside the module
- Use `-callback` attributes, not `behaviour_info/1`
- Define `-type state() :: #mod_state{}` in every OTP behaviour implementation

### Strings and Data
- Prefer iolists over string concatenation
- Use tagged tuples or atoms for inter-process messages: `{set_connection_pid, Pid}`
- Element 1 of message tuples is always a human-readable atom

### Records
- Record and field names: lowercase with underscores
- Define before function bodies
- Never share via header files: provide opaque types and accessors
- Always add type definitions to record fields

### Error Handling
- Let processes crash: supervision handles recovery
- `rescue`/`catch` only at system boundaries
- Never silently swallow errors
- Validate at system boundaries, trust internal code
- Defensive checks on the client side (before `gen_server:call`)

### Testing
- One or two asserts per test function
- Use Common Test for integration, EUnit for unit
- Mark property-based tests clearly

### Dependencies
- Lock to tags or commits in `rebar.config`, never `master`/`main`

### Logging
- No `io:format` in production code
- Use appropriate log levels: `debug` (verbose), `info` (lifecycle), `notice` (meaningful events), `warning` (handled errors), `error` (unexpected failures with stack traces)


## 4. NIF Resource Lifecycle Patterns

### Pattern: Owner-Monitored Resources

The gold standard for BEAM-native resource management. Each native resource is owned by an Erlang process. The NIF monitors the owner; when the owner exits, the resource is cleaned up.

```c
typedef struct {
    ErlNifMutex*    lock;
    ErlNifPid       owner;
    ErlNifMonitor   owner_mon;
    quiche_conn*    conn;
    int             closed;
} ConnResource;

static ERL_NIF_TERM
nif_accept(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    ConnResource* r = enif_alloc_resource(CONN_RESOURCE, sizeof(ConnResource));
    r->lock = enif_mutex_create("conn_lock");
    r->closed = 0;
    r->conn = do_accept(/* ... */);

    // The calling process becomes the owner
    enif_self(env, &r->owner);
    enif_monitor_process(env, r, &r->owner, &r->owner_mon);

    ERL_NIF_TERM term = enif_make_resource(env, r);
    enif_release_resource(r);
    return enif_make_tuple2(env, enif_make_atom(env, "ok"), term);
}

// Called when the monitored owner process exits
static void
conn_owner_down(ErlNifEnv* env, void* obj,
                ErlNifPid* pid, ErlNifMonitor* mon) {
    ConnResource* r = (ConnResource*)obj;
    enif_mutex_lock(r->lock);
    if (!r->closed && r->conn) {
        quiche_conn_close(r->conn, true, 0, NULL, 0);
        r->closed = 1;
    }
    enif_mutex_unlock(r->lock);
}
```

### Pattern: Hierarchical Resource Contexts

From quicer: resources form a tree (Registration > Listener > Connection > Stream). Child contexts hold references to parents, preventing premature parent cleanup.

```c
typedef struct {
    ErlNifMutex*      lock;
    quiche_conn*      conn;
    ConfigResource*   config;   // ref-counted parent reference
    int               closed;
} ConnResource;

// In destructor: release parent reference
static void conn_destructor(ErlNifEnv* env, void* obj) {
    ConnResource* r = (ConnResource*)obj;
    if (r->conn) quiche_conn_free(r->conn);
    if (r->config) enif_release_resource(r->config);
    if (r->lock) enif_mutex_destroy(r->lock);
}

// When creating a connection from a config:
ConnResource* conn = enif_alloc_resource(CONN_RESOURCE, sizeof(ConnResource));
conn->config = config;
enif_keep_resource(config);  // prevent GC of parent
```

### Pattern: Active/Passive Message Delivery

Mirror `:gen_tcp` semantics. Active mode pushes data as messages to the owner; passive mode requires explicit `recv` calls.

```erlang
%% Active mode (default): NIF sends messages to owner
%% Owner receives: {mylib, data, StreamHandle, Data}
%%                 {mylib, stream_closed, StreamHandle, Flags}

%% Passive mode: explicit receive
{ok, Data} = mylib:recv(StreamHandle, 1000),

%% Switching modes (like inet:setopts)
ok = mylib:setopt(StreamHandle, active, false),
ok = mylib:setopt(StreamHandle, active, once),
ok = mylib:setopt(StreamHandle, active, 10),  % N messages then passive
```


## 5. Build System (rebar3)

### rebar.config for C NIF with Rust Static Library

```erlang
{erl_opts, [debug_info, warnings_as_errors]}.

{pre_hooks, [
    {"(linux|darwin)", compile, "make -C c_src"},
    {"(linux|darwin)", clean,   "make -C c_src clean"}
]}.

{post_hooks, []}.

{port_specs, [
    {"priv/mylib_nif.so", ["c_src/*.c"]}
]}.

{port_env, [
    {"CFLAGS",  "$CFLAGS -I c_src/include -Wall -Werror -std=c11"},
    {"LDFLAGS", "$LDFLAGS -L c_src/lib -lquiche -lm -lpthread"},
    %% macOS: no -lrt
    {"darwin", "LDFLAGS", "$LDFLAGS -L c_src/lib -lquiche -lm -lpthread"},
    %% Linux: needs -lrt for clock_gettime on older glibc
    {"linux",  "LDFLAGS", "$LDFLAGS -L c_src/lib -lquiche -lm -lpthread -lrt"}
]}.

{dialyzer, [
    {warnings, [error_handling, underspecs, unmatched_returns]}
]}.

{xref_checks, [undefined_function_calls, undefined_functions]}.
```

### c_src/Makefile Pattern

```makefile
QUICHE_DIR ?= $(CURDIR)/deps/quiche
QUICHE_LIB  = $(QUICHE_DIR)/target/release/libquiche.a

# Build quiche static library (first build pulls BoringSSL, ~5min)
$(QUICHE_LIB):
	cd $(QUICHE_DIR) && cargo build --release --features ffi

# Ensure quiche headers are available
include: $(QUICHE_DIR)/quiche/include/quiche.h

clean:
	rm -f ../priv/mylib_nif.so
```


## 6. Event Message Format Convention

Follow the quicer convention. All messages from NIF to Erlang use a 4-tuple:

```erlang
{mylib, EventName :: atom(), ResourceHandle :: reference(), Props :: map()}
```

Examples:
```erlang
{mylib, connected,      ConnHandle,   #{is_resumed => false}}
{mylib, data,           StreamHandle, #{data => Binary, flags => Flags}}
{mylib, stream_closed,  StreamHandle, #{error_code => 0}}
{mylib, peer_initiated, ConnHandle,   #{stream => StreamHandle}}
{mylib, dgram_recv,     ConnHandle,   #{data => Binary}}
```

This convention enables:
- Pattern matching on event name in `receive` blocks and `handle_info` clauses
- Resource handle correlation (which connection/stream)
- Extensible properties without breaking existing pattern matches


## 7. Testing Strategy for NIF Libraries

### Unit Tests (EUnit in Erlang, cargo test in C/Rust)

```erlang
-module(mylib_tests).
-include_lib("eunit/include/eunit.hrl").

config_create_test() ->
    {ok, Config} = mylib:config(#{alpn_protos => [<<"h3">>]}),
    ?assertMatch({ok, _}, {ok, Config}).

bad_args_test() ->
    ?assertError(badarg, mylib_nif:connect(not_a_config, <<"example.com">>)).
```

### Integration Tests (Common Test)

```erlang
-module(mylib_SUITE).
-include_lib("common_test/include/ct.hrl").
-export([all/0, init_per_suite/1, end_per_suite/1]).
-export([bidirectional_handshake/1, garbage_packet_survives/1]).

all() -> [bidirectional_handshake, garbage_packet_survives].

init_per_suite(Config) ->
    {ok, _} = application:ensure_all_started(mylib),
    Config.

bidirectional_handshake(Config) ->
    %% Start listener
    {ok, L} = mylib:listen(#{port => 0, certfile => cert(), keyfile => key()}),
    {ok, Port} = mylib:listener_port(L),
    %% Connect client
    {ok, Conn} = mylib:connect(<<"localhost">>,
                               #{port => Port, alpn_protos => [<<"h3">>]}),
    receive {mylib, connected, Conn, _} -> ok
    after 5000 -> ct:fail(timeout)
    end.
```

### Property-Based Tests (PropEr)

```erlang
%% No crash on any input: the NIF must never segfault
prop_recv_no_crash() ->
    ?FORALL(Bin, binary(),
        begin
            %% Feed garbage to the recv path
            Res = mylib_nif:feed_data(make_conn(), Bin),
            is_tuple(Res) orelse is_atom(Res)
        end).
```


## 8. Prior Art Reference

### quicer (emqx/quic) -- Erlang NIF wrapping MsQuic

The most mature Erlang QUIC NIF binding. Key architectural patterns to study:

- **4-layer architecture:** Erlang API > NIF stubs > C implementation > MsQuic
- **Per-component C files:** `quicer_connection.c`, `quicer_stream.c`, `quicer_listener.c`
- **Process ownership model:** each resource owned by an Erlang process, monitored for cleanup
- **Active/passive mode:** mirrors `:gen_tcp` semantics
- **Event tuple format:** `{quic, EventName, Handle, Props}`
- **Hierarchical contexts:** Global > Registration > Config > Listener > Connection > Stream
- **Mutex per context:** `ErlNifMutex` on every mutable resource
- **Kill switch:** `QUICER_SKIP_NIF_LOAD` env var for testing without native library
- **ABI versioning:** persistent_term override for version compat

### OTP crypto -- Erlang NIF wrapping OpenSSL/BoringSSL

OTP's own NIF patterns at `lib/crypto/src/crypto.erl`:
- `-on_load` for NIF initialization
- Stub functions raising `nif_error`
- Resource types for EVP contexts
- Dirty NIF flags for heavy crypto operations

### Key Differences: quiche vs msquic for NIF Design

| Aspect | quiche (our choice) | msquic (quicer uses) |
|--------|-------------------|---------------------|
| C API stability | Stable `quiche.h`, documented | Stable `msquic.h` |
| Callback model | Poll-based (caller drives) | Event-driven (callbacks fire) |
| Thread model | Caller's thread | Internal thread pool |
| NIF complexity | Simpler: NIF calls quiche, gets result | Complex: must handle async callbacks |
| I/O ownership | Caller owns socket | msquic owns socket by default |

quiche's poll-based model is a better fit for BEAM integration because the Erlang process naturally drives the I/O loop via `gen_udp`, and quiche just processes packets. No callback registration, no thread pool management, no socket ownership fights.
