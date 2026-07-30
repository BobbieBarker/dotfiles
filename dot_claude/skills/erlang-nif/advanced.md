# Advanced Erlang Patterns for NIF Library Development

## OTP Behaviour Integration

### gen_statem for Connection State Machines

QUIC connections have natural state machine semantics. `gen_statem` (not `gen_server`) is the right behaviour when connection lifecycle has distinct states with different valid operations per state.

```erlang
-module(mylib_conn).
-behaviour(gen_statem).

-export([start_link/2, send/2, recv/2, close/1]).
-export([init/1, callback_mode/0, terminate/3]).
-export([connecting/3, handshaking/3, established/3, closing/3]).

-record(data, {
    conn_handle  :: mylib_nif:conn_handle(),
    owner        :: pid(),
    owner_mon    :: reference(),
    streams      :: #{mylib_nif:stream_handle() => pid()},
    send_queue   :: queue:queue(),
    timer_ref    :: reference() | undefined
}).

%%% Client API

start_link(ConnHandle, Opts) ->
    gen_statem:start_link(?MODULE, {ConnHandle, Opts, self()}, []).

send(Pid, Data) ->
    gen_statem:call(Pid, {send, Data}).

recv(Pid, Timeout) ->
    gen_statem:call(Pid, recv, Timeout).

close(Pid) ->
    gen_statem:call(Pid, close).

%%% Callbacks

callback_mode() -> [state_functions, state_enter].

init({ConnHandle, _Opts, Owner}) ->
    MonRef = monitor(process, Owner),
    Data = #data{
        conn_handle = ConnHandle,
        owner       = Owner,
        owner_mon   = MonRef,
        streams     = #{},
        send_queue  = queue:new()
    },
    {ok, connecting, Data}.

%% State: connecting
connecting(enter, _OldState, Data) ->
    TimerRef = erlang:send_after(5000, self(), connect_timeout),
    {keep_state, Data#data{timer_ref = TimerRef}};

connecting(info, {mylib, connected, _Handle, Props}, Data) ->
    cancel_timer(Data),
    {next_state, established, Data};

connecting(info, connect_timeout, Data) ->
    {stop, {shutdown, connect_timeout}, Data};

connecting({call, From}, {send, _Data}, _Data) ->
    {keep_state_and_data, [{reply, From, {error, not_connected}}]}.

%% State: established
established(enter, _OldState, Data) ->
    flush_send_queue(Data),
    {keep_state, Data};

established({call, From}, {send, Payload}, #data{conn_handle = H} = Data) ->
    Result = mylib_nif:stream_send(H, iolist_to_binary(Payload)),
    {keep_state, Data, [{reply, From, Result}]};

established(info, {mylib, data, StreamHandle, #{data := Bin}},
            #data{owner = Owner} = Data) ->
    Owner ! {mylib_data, self(), StreamHandle, Bin},
    {keep_state, Data};

established(info, {mylib, peer_stream, _ConnHandle, #{stream := SH}},
            #data{streams = Streams} = Data) ->
    {ok, Pid} = mylib_stream_sup:start_stream(SH),
    {keep_state, Data#data{streams = Streams#{SH => Pid}}};

established({call, From}, close, #data{conn_handle = H} = Data) ->
    mylib_nif:close(H),
    {next_state, closing, Data, [{reply, From, ok}]}.

%% State: closing
closing(enter, _OldState, Data) ->
    TimerRef = erlang:send_after(3000, self(), close_timeout),
    {keep_state, Data#data{timer_ref = TimerRef}};

closing(info, {mylib, closed, _Handle, _Props}, Data) ->
    {stop, normal, Data};

closing(info, close_timeout, Data) ->
    {stop, normal, Data}.

%%% Internal

cancel_timer(#data{timer_ref = undefined}) -> ok;
cancel_timer(#data{timer_ref = Ref}) -> erlang:cancel_timer(Ref).

flush_send_queue(#data{send_queue = Q, conn_handle = H}) ->
    lists:foreach(
        fun(Payload) -> mylib_nif:stream_send(H, Payload) end,
        queue:to_list(Q)).

terminate(_Reason, _State, #data{conn_handle = H}) ->
    mylib_nif:close(H),
    ok.
```

### When gen_server vs gen_statem

| Use case | Behaviour | Why |
|----------|-----------|-----|
| Simple resource wrapper (config, stats) | `gen_server` | No state transitions, just CRUD |
| Connection lifecycle | `gen_statem` | Distinct states with different valid operations |
| Stream data pump | `gen_server` | Mostly steady-state data flow |
| Listener accept loop | `gen_server` | Single state: accepting |
| Protocol negotiation | `gen_statem` | Multi-phase handshake with timeouts per phase |


## Concurrency Patterns for NIF Libraries

### Process-per-Entity Model

Each QUIC entity (connection, stream) maps to one Erlang process. This gives:
- Isolated failure domains (one bad stream doesn't crash the connection)
- Natural mailbox-based buffering
- Per-entity GC (short-lived stream processes GC independently)
- Familiar OTP supervision semantics

```
Listener process (gen_server)
  |-- owns UDP socket
  |-- demuxes packets by Connection ID
  |
  +-- Connection process 1 (gen_statem)
  |     |-- owns quiche_conn resource
  |     |-- drives I/O loop for this connection
  |     |
  |     +-- Stream process A (gen_server)
  |     |     |-- owns stream handle
  |     |     |-- active/passive recv
  |     |
  |     +-- Stream process B (gen_server)
  |
  +-- Connection process 2 (gen_statem)
        |-- ...
```

### Selective Receive for Protocol Sequencing

Erlang's selective receive is powerful for protocol handshakes where messages must be processed in order.

```erlang
%% Wait for handshake completion, ignoring data messages
%% that arrive before handshake finishes (they stay in mailbox)
wait_handshake(ConnHandle, Timeout) ->
    receive
        {mylib, connected, ConnHandle, Props} ->
            {ok, Props};
        {mylib, connect_failed, ConnHandle, Reason} ->
            {error, Reason}
    after Timeout ->
        {error, timeout}
    end.
%% After handshake, data messages that arrived early are still
%% in the mailbox and will be picked up by the next receive.
```

### Monitor vs Link Decision Matrix

| Scenario | Use | Why |
|----------|-----|-----|
| Stream owned by connection | Monitor | Connection should clean up, not crash |
| Connection owned by supervisor | Link (via start_link) | Supervisor restarts on crash |
| Owner process watching resource | Monitor | Owner should handle cleanup gracefully |
| Paired processes (must die together) | Link | Bidirectional fate-sharing |

### Avoiding Mailbox Overflow

NIF libraries that push data via `enif_send` can overwhelm a process mailbox. Mitigation:

```erlang
%% Pattern 1: Bounded active mode (like {active, N} in gen_tcp)
ok = mylib:setopt(StreamHandle, active, 100),
%% After 100 messages, mode flips to passive and owner gets:
%% {mylib_passive, StreamHandle}

%% Pattern 2: Flow control in the NIF
%% Connection process tells NIF to stop sending when buffer > threshold
handle_info({mylib, data, SH, #{data := Bin}}, #data{buffer_size = BS} = Data)
  when BS > ?MAX_BUFFER ->
    mylib_nif:stream_set_readable(SH, false),  % pause NIF-side delivery
    {noreply, Data#data{buffer_size = BS + byte_size(Bin)}};
```


## Distributed Erlang Considerations for NIF Libraries

### Resource Handles are Node-Local

NIF resource handles (opaque references) are only valid on the node that created them. They cannot be sent to remote nodes.

```erlang
%% BAD: sending a NIF resource handle to a remote node
RemotePid ! {use_this, ConnHandle},  % ConnHandle is garbage on remote node

%% GOOD: proxy through a local process
%% Remote callers send requests to a local process that owns the resource
mylib_proxy:send(RemoteNode, ConnId, Data)
```

### Cluster-Aware Connection Routing

For distributed deployments, use `pg` (process groups) to discover which node owns a particular connection.

```erlang
%% On connection establishment:
pg:join(mylib_connections, {server_name, <<"example.com">>}, self()).

%% From any node, find who handles a particular server:
case pg:get_members(mylib_connections, {server_name, ServerName}) of
    [Pid | _] -> gen_statem:call(Pid, {send, Data});
    []        -> {error, no_connection}
end.
```


## Error Handling Philosophy

### Let It Crash (with nuance for NIFs)

Standard Erlang "let it crash" applies to the Erlang process layer. The NIF layer is different: a crash in C brings down the entire VM.

**Erlang layer:** let processes crash, supervisors restart them. A stream process dying is fine.

**NIF layer:** defensive programming. Check every pointer, every return code, every allocation. Never crash.

```erlang
%% Erlang side: crash-tolerant
handle_info({mylib, stream_error, SH, Reason}, Data) ->
    %% Stream process crashes, supervisor restarts it
    exit(stream_processes(SH, Data), {stream_error, Reason}),
    {noreply, remove_stream(SH, Data)}.
```

```c
/* C side: paranoid */
static ERL_NIF_TERM
nif_stream_send(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    ConnResource* r;
    ErlNifBinary bin;

    if (!enif_get_resource(env, argv[0], CONN_RESOURCE, (void**)&r))
        return enif_make_badarg(env);

    if (!enif_inspect_binary(env, argv[1], &bin))
        return enif_make_badarg(env);

    enif_mutex_lock(r->lock);
    if (r->closed) {
        enif_mutex_unlock(r->lock);
        return error_tuple(env, "closed");
    }

    ssize_t written = quiche_conn_stream_send(
        r->conn, r->stream_id, bin.data, bin.size, false);

    enif_mutex_unlock(r->lock);

    if (written < 0)
        return error_tuple(env, quiche_error_str(written));

    return ok_tuple(env, enif_make_long(env, written));
}
```

### Graceful Degradation Under Load

```erlang
%% Connection process monitors its own mailbox depth
handle_info(check_pressure, #data{} = Data) ->
    {message_queue_len, Len} = process_info(self(), message_queue_len),
    NewData = case Len > ?PRESSURE_THRESHOLD of
        true  -> apply_backpressure(Data);
        false -> release_backpressure(Data)
    end,
    erlang:send_after(?PRESSURE_CHECK_INTERVAL, self(), check_pressure),
    {noreply, NewData}.

apply_backpressure(#data{conn_handle = H} = Data) ->
    %% Tell NIF to stop reading from socket temporarily
    mylib_nif:pause_recv(H),
    Data#data{backpressure = true}.
```


## Property-Based Testing with PropEr

### No-Crash Properties (Critical for NIFs)

```erlang
-module(mylib_prop_SUITE).
-include_lib("proper/include/proper.hrl").
-include_lib("common_test/include/ct.hrl").

%% The NIF must never crash the VM regardless of input
prop_config_no_crash() ->
    ?FORALL(Opts, config_opts_gen(),
        begin
            Res = mylib_nif:config_new(Opts),
            is_tuple(Res)
        end).

%% Garbage packets must not crash or leak
prop_recv_no_crash() ->
    ?FORALL(Bin, binary(),
        begin
            {ok, Config} = mylib:config(test_config()),
            {ok, Conn} = mylib_nif:accept(Config),
            Res = mylib_nif:recv(Conn, Bin),
            mylib_nif:close(Conn),
            is_tuple(Res) orelse is_atom(Res)
        end).

%% Resource count must be bounded (no leaks)
prop_resource_bounded() ->
    ?FORALL(Cmds, commands(?MODULE),
        begin
            {_H, _S, Res} = run_commands(?MODULE, Cmds),
            cleanup_all(),
            %% After cleanup, check no leaked resources
            Res =:= ok
        end).

%% Generators
config_opts_gen() ->
    #{alpn_protos => list(binary()),
      idle_timeout => non_neg_integer(),
      max_streams => non_neg_integer()}.
```

### Stateful Testing with PropEr State Machine

```erlang
%% Model the connection lifecycle as a state machine
%% PropEr generates random sequences of operations and
%% verifies postconditions hold

-record(model, {
    conns = [] :: [reference()],
    streams = [] :: [{reference(), reference()}]
}).

initial_state() -> #model{}.

command(#model{conns = []}) ->
    {call, mylib, connect, [<<"localhost">>, test_opts()]};
command(#model{conns = Conns}) ->
    oneof([
        {call, mylib, connect, [<<"localhost">>, test_opts()]},
        {call, mylib, send, [elements(Conns), binary()]},
        {call, mylib, close, [elements(Conns)]}
    ]).

postcondition(#model{}, {call, _, connect, _}, {ok, _Conn}) -> true;
postcondition(#model{}, {call, _, connect, _}, {error, _}) -> true;
postcondition(#model{}, {call, _, send, _}, ok) -> true;
postcondition(#model{}, {call, _, send, _}, {error, closed}) -> true;
postcondition(_, _, _) -> false.  % unexpected result = test failure
```


## Memory and Performance Considerations

### Binary Reference Counting

The BEAM uses reference-counted binaries for data > 64 bytes. This matters for NIF data transfer:

```
Packet flow: gen_udp -> NIF -> quiche -> NIF -> Erlang process

- gen_udp delivers a refc binary (>64B packets)
- NIF receives it via enif_inspect_binary (zero-copy reference)
- quiche processes in-place (no copy if possible)
- NIF output via enif_make_new_binary (new refc binary)
- Erlang process receives refc binary (pointer, not copy)

Total copies for a typical 1400B QUIC packet: 1 (NIF output)
Compare to Port-based approach: 2+ copies minimum
```

### ETS for Hot-Path Lookups

Connection ID to process mapping must be fast. ETS provides concurrent read access without process bottleneck.

```erlang
%% Created by listener, read by any process
Tab = ets:new(mylib_conn_table, [
    set,
    public,              % any process can read
    {read_concurrency, true},  % optimized for concurrent reads
    named_table
]),

%% Lookup on packet arrival (hot path)
case ets:lookup(mylib_conn_table, DCID) of
    [{DCID, ConnPid}] -> ConnPid ! {quic_packet, Packet};
    [] -> handle_new_connection(Packet)
end.
```

### Reduction Budgeting

Each NIF call consumes reductions. Budget accordingly:

| Operation | Typical time | Scheduler impact |
|-----------|-------------|-----------------|
| config_new | ~10us | Normal scheduler, no issue |
| conn_recv (1 packet) | ~50us | Normal, call enif_consume_timeslice |
| conn_send (1 packet) | ~30us | Normal, call enif_consume_timeslice |
| stream_recv (bulk) | ~500us | Borderline, use dirty IO if >1ms |
| handshake (crypto) | ~2-5ms | Dirty CPU scheduler |
| key_update | ~1ms | Dirty CPU scheduler |
