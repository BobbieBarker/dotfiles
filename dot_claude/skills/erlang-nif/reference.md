# Erlang NIF Reference

## OTP Behaviour Patterns for NIF Libraries

### gen_server Wrapping a NIF Resource

The standard pattern for stateful NIF resources: a `gen_server` owns the resource handle, encapsulates all calls, and ensures cleanup on termination.

```erlang
-module(mylib_connection).
-behaviour(gen_server).

-export([start_link/2, send/2, recv/2, close/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-record(state, {
    conn_handle :: mylib_nif:conn_handle(),
    owner       :: pid(),
    active      :: boolean() | once | non_neg_integer(),
    buffer      :: queue:queue(binary())
}).
-type state() :: #state{}.

%%% Client API (encapsulated -- callers never use gen_server:call directly)

-spec start_link(mylib_nif:conn_handle(), map()) -> {ok, pid()} | {error, term()}.
start_link(ConnHandle, Opts) ->
    gen_server:start_link(?MODULE, {ConnHandle, Opts, self()}, []).

-spec send(pid(), iodata()) -> ok | {error, term()}.
send(Pid, Data) ->
    gen_server:call(Pid, {send, Data}).

-spec recv(pid(), timeout()) -> {ok, binary()} | {error, term()}.
recv(Pid, Timeout) ->
    gen_server:call(Pid, recv, Timeout).

-spec close(pid()) -> ok.
close(Pid) ->
    gen_server:stop(Pid, normal, 5000).

%%% gen_server callbacks

init({ConnHandle, Opts, Owner}) ->
    _MonRef = monitor(process, Owner),
    Active = maps:get(active, Opts, true),
    {ok, #state{conn_handle = ConnHandle,
                owner = Owner,
                active = Active,
                buffer = queue:new()}}.

handle_call({send, Data}, _From, #state{conn_handle = H} = State) ->
    Reply = mylib_nif:stream_send(H, iolist_to_binary(Data)),
    {reply, Reply, State};

handle_call(recv, _From, #state{buffer = Buf} = State) ->
    case queue:out(Buf) of
        {{value, Data}, Buf2} ->
            {reply, {ok, Data}, State#state{buffer = Buf2}};
        {empty, _} ->
            {reply, {error, empty}, State}
    end.

handle_cast(_Msg, State) ->
    {noreply, State}.

%% Active mode: forward NIF data messages to owner
handle_info({mylib, data, _Handle, #{data := Data}},
            #state{active = true, owner = Owner} = State) ->
    Owner ! {mylib_data, self(), Data},
    {noreply, State};

handle_info({mylib, data, _Handle, #{data := Data}},
            #state{active = once, owner = Owner} = State) ->
    Owner ! {mylib_data, self(), Data},
    {noreply, State#state{active = false}};

%% Passive mode: buffer incoming data
handle_info({mylib, data, _Handle, #{data := Data}},
            #state{active = false, buffer = Buf} = State) ->
    {noreply, State#state{buffer = queue:in(Data, Buf)}};

%% Owner died: clean up
handle_info({'DOWN', _Ref, process, Owner, _Reason},
            #state{owner = Owner} = State) ->
    {stop, owner_down, State}.

terminate(_Reason, #state{conn_handle = H}) ->
    mylib_nif:close(H),
    ok.
```

### Supervisor Tree for a NIF Library Application

```erlang
-module(mylib_sup).
-behaviour(supervisor).

-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    SupFlags = #{
        strategy  => one_for_one,
        intensity => 10,
        period    => 60
    },
    Children = [
        %% Listener supervisor: one child per listening port
        #{id       => mylib_listener_sup,
          start    => {mylib_listener_sup, start_link, []},
          type     => supervisor,
          shutdown => infinity},
        %% Connection supervisor: dynamic, one child per accepted connection
        #{id       => mylib_conn_sup,
          start    => {mylib_conn_sup, start_link, []},
          type     => supervisor,
          shutdown => infinity}
    ],
    {ok, {SupFlags, Children}}.
```

```erlang
-module(mylib_conn_sup).
-behaviour(supervisor).

-export([start_link/0, start_connection/2, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

start_connection(ConnHandle, Opts) ->
    supervisor:start_child(?MODULE, [ConnHandle, Opts]).

init([]) ->
    SupFlags = #{
        strategy  => simple_one_for_one,
        intensity => 0,
        period    => 1
    },
    ChildSpec = #{
        id      => mylib_connection,
        start   => {mylib_connection, start_link, []},
        restart => temporary,
        type    => worker
    },
    {ok, {SupFlags, [ChildSpec]}}.
```


## Erlang Concurrency Patterns for NIF Libraries

### Process-per-Connection Model

Each QUIC connection gets its own Erlang process. The process:
1. Owns the NIF resource handle
2. Drives the I/O loop (poll `gen_udp`, feed packets to quiche via NIF)
3. Demultiplexes streams to child processes
4. Handles timers (idle timeout, keep-alive, loss detection)

```
                    mylib_sup
                       |
           +-----------+-----------+
           |                       |
    mylib_listener_sup      mylib_conn_sup
           |                  (simple_one_for_one)
    mylib_listener            |     |     |
    (gen_server,           conn_1 conn_2 conn_3
     owns UDP socket,      (gen_server, each owns
     demuxes by CID)        a quiche_conn resource)
                               |
                          stream processes
                          (one per stream)
```

### UDP Demultiplexing (Mode A: gen_udp-driven)

The listener process owns the UDP socket. Inbound packets are routed to the correct connection process by QUIC Connection ID.

```erlang
-module(mylib_listener).
-behaviour(gen_server).

-record(state, {
    socket      :: gen_udp:socket(),
    config      :: mylib_nif:config_handle(),
    conn_table  :: ets:tid()  % CID -> pid mapping
}).

handle_info({udp, Socket, IP, Port, Packet},
            #state{socket = Socket, conn_table = Tab, config = Config} = State) ->
    case extract_dcid(Packet) of
        {ok, DCID} ->
            case ets:lookup(Tab, DCID) of
                [{DCID, ConnPid}] ->
                    ConnPid ! {quic_packet, IP, Port, Packet};
                [] ->
                    handle_new_connection(IP, Port, Packet, Config, Tab)
            end;
        {error, _} ->
            ok  % Drop malformed packets silently
    end,
    {noreply, State}.
```

### Timer Management

QUIC requires precise timer management for retransmission, idle timeout, and loss detection. Use `erlang:send_after/3` driven by quiche's `quiche_conn_timeout` API.

```erlang
handle_info(quic_timeout, #state{conn_handle = H} = State) ->
    mylib_nif:on_timeout(H),
    schedule_next_timeout(H),
    maybe_send_pending(H, State);

handle_info({quic_packet, _IP, _Port, Packet},
            #state{conn_handle = H} = State) ->
    case mylib_nif:recv(H, Packet) of
        ok ->
            cancel_timer(State),
            schedule_next_timeout(H),
            maybe_send_pending(H, State);
        {error, done} ->
            {stop, normal, State}
    end.

schedule_next_timeout(ConnHandle) ->
    case mylib_nif:timeout_as_millis(ConnHandle) of
        infinity -> ok;
        Ms       -> erlang:send_after(Ms, self(), quic_timeout)
    end.
```

### Registry for Dynamic Connection Lookup

```erlang
%% In application start:
{ok, _} = Registry:start_link(keys: :unique, name: mylib_conn_registry).

%% When a connection is established, register by CID:
Registry:register(mylib_conn_registry, DCID, ConnPid).

%% Lookup:
case Registry:lookup(mylib_conn_registry, DCID) of
    [{ConnPid, _}] -> ConnPid;
    []             -> undefined
end.
```


## C NIF Implementation Patterns

### Standard NIF Entry Point File

```c
#include <erl_nif.h>
#include <quiche.h>
#include <string.h>

// Resource types
static ErlNifResourceType* CONFIG_RESOURCE  = NULL;
static ErlNifResourceType* CONN_RESOURCE    = NULL;

// Resource structs
typedef struct {
    quiche_config* config;
} ConfigResource;

typedef struct {
    ErlNifMutex*    lock;
    ErlNifPid       owner;
    ErlNifMonitor   owner_mon;
    quiche_conn*    conn;
    int             closed;
} ConnResource;

// Resource callbacks
static ErlNifResourceTypeInit conn_resource_init = {
    .dtor    = conn_destructor,
    .stop    = NULL,
    .down    = conn_owner_down,  // monitor callback
    .members = 3
};

// Load callback: register resource types
static int
load(ErlNifEnv* env, void** priv, ERL_NIF_TERM load_info) {
    CONFIG_RESOURCE = enif_open_resource_type(
        env, NULL, "quic_config",
        config_destructor, ERL_NIF_RT_CREATE, NULL);

    CONN_RESOURCE = enif_open_resource_type_x(
        env, "quic_conn",
        &conn_resource_init,
        ERL_NIF_RT_CREATE, NULL);

    if (!CONFIG_RESOURCE || !CONN_RESOURCE)
        return -1;

    return 0;
}

// Upgrade callback (for hot code loading)
static int
upgrade(ErlNifEnv* env, void** priv, void** old_priv, ERL_NIF_TERM load_info) {
    return load(env, priv, load_info);
}

// NIF function table
static ErlNifFunc nif_funcs[] = {
    // name,         arity, function_ptr,              flags
    {"config_new",   1,     nif_config_new,            0},
    {"connect",      3,     nif_connect,               0},
    {"recv",         2,     nif_recv,                   0},
    {"send",         2,     nif_send,                   0},
    {"close",        1,     nif_close,                  0},
    {"timeout",      1,     nif_timeout,                0},
    {"on_timeout",   1,     nif_on_timeout,             0},
    // Dirty NIF for potentially long operations
    {"stream_recv",  3,     nif_stream_recv,            ERL_NIF_DIRTY_JOB_IO_BOUND},
};

ERL_NIF_INIT(mylib_nif, nif_funcs, load, NULL, upgrade, NULL)
```

### Argument Extraction Helpers

```c
// Extract binary from Erlang term
static int
get_binary(ErlNifEnv* env, ERL_NIF_TERM term, ErlNifBinary* bin) {
    return enif_inspect_binary(env, term, bin)
        || enif_inspect_iolist_as_binary(env, term, bin);
}

// Extract map value by atom key
static ERL_NIF_TERM
get_map_val(ErlNifEnv* env, ERL_NIF_TERM map, const char* key) {
    ERL_NIF_TERM atom = enif_make_atom(env, key);
    ERL_NIF_TERM val;
    if (enif_get_map_value(env, map, atom, &val))
        return val;
    return 0;  // caller must check
}

// Build an {ok, Resource} tuple
static ERL_NIF_TERM
ok_tuple(ErlNifEnv* env, ERL_NIF_TERM value) {
    return enif_make_tuple2(env,
        enif_make_atom(env, "ok"), value);
}

// Build an {error, Reason} tuple
static ERL_NIF_TERM
error_tuple(ErlNifEnv* env, const char* reason) {
    return enif_make_tuple2(env,
        enif_make_atom(env, "error"),
        enif_make_atom(env, reason));
}
```

### Binary Data Handling

```c
// Return data as an Erlang binary (zero-copy for large data via refc binaries)
static ERL_NIF_TERM
make_binary_term(ErlNifEnv* env, const uint8_t* data, size_t len) {
    ERL_NIF_TERM bin_term;
    unsigned char* buf = enif_make_new_binary(env, len, &bin_term);
    if (buf == NULL)
        return enif_make_badarg(env);
    memcpy(buf, data, len);
    return bin_term;
}

// For large data (>64 bytes), BEAM uses refc binaries automatically.
// enif_make_new_binary handles this transparently.
// Below 64 bytes, data is copied onto the process heap.
```


## Dialyzer and Xref Configuration

```erlang
%% rebar.config
{dialyzer, [
    {warnings, [
        error_handling,
        underspecs,
        unmatched_returns,
        unknown
    ]},
    {plt_extra_apps, [crypto, ssl]}
]}.

{xref_checks, [
    undefined_function_calls,
    undefined_functions,
    locals_not_used,
    deprecated_function_calls
]}.
```

Always run both before committing:
```shell
rebar3 dialyzer
rebar3 xref
```


## Common Test Suite Structure

```
test/
  mylib_SUITE.erl           % Integration: handshake, data transfer, reconnect
  mylib_config_SUITE.erl    % Config creation, validation, edge cases
  mylib_stream_SUITE.erl    % Stream lifecycle, flow control, reset
  mylib_property_SUITE.erl  % PropEr: no-crash on garbage, memory bounds
  mylib_interop_SUITE.erl   % QUIC Interop Runner harness

%% Test certificates for TLS
test/certs/
  ca.pem
  server.pem
  server.key
  client.pem
  client.key
```

Generate test certificates:
```shell
# Self-signed CA + server cert for testing
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 \
  -keyout test/certs/server.key -out test/certs/server.pem \
  -days 365 -nodes -subj "/CN=localhost"
```
