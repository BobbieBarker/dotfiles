---
name: quiche-http3
description: >
  Cloudflare quiche QUIC/HTTP3 library integration -- C API surface, connection
  lifecycle, I/O model, timer management, HTTP/3 event processing, and BEAM
  integration patterns. ALWAYS use when working on quiche_erl or plug_quiche.
  ALWAYS use when writing NIF bindings for quiche. ALWAYS use when implementing
  QUIC connection management or HTTP/3 request handling.
---

# Cloudflare quiche: QUIC and HTTP/3

## 1. Core Architecture

quiche is a **caller-driven, poll-based** QUIC and HTTP/3 library. The application owns:
- The socket (UDP)
- The event loop
- All timers

quiche never spawns threads, never opens sockets, never sets timers. The application calls quiche functions to process packets and query state. This is the key property that makes it a good fit for BEAM integration: an Erlang process naturally fills the role of "application event loop."

### Integration loop (pseudocode)

```
loop:
  packet = socket.recv()
  quiche_conn_recv(conn, packet)

  if quiche_conn_is_established(conn):
    process_h3_events(h3_conn, conn)

  while quiche_conn_send(conn, out) != DONE:
    socket.send(out)

  timeout = quiche_conn_timeout_as_millis(conn)
  schedule_timer(timeout)

on_timer:
  quiche_conn_on_timeout(conn)
  goto loop  // may generate new packets
```

### BEAM mapping

| quiche concept | BEAM equivalent |
|---------------|----------------|
| Application event loop | Erlang process (gen_server or gen_statem) |
| UDP socket | `:gen_udp` socket owned by listener process |
| Timer | `erlang:send_after/3` driven by `quiche_conn_timeout_as_millis` |
| Connection state | NIF resource (`ErlNifResourceType`) |
| Multiple connections | One process per connection, demuxed by Connection ID |
| HTTP/3 events | Messages to the connection-owning process |


## 2. quiche C API Reference

### Opaque types

```c
quiche_config         // Shared config (TLS certs, transport params, ALPN)
quiche_conn           // Single QUIC connection
quiche_stream_iter    // Iterator over readable/writable stream IDs
quiche_h3_config      // HTTP/3 specific config (QPACK settings)
quiche_h3_conn        // HTTP/3 layer on top of a QUIC connection
quiche_h3_event       // HTTP/3 event (headers, data, finished, goaway, reset)
quiche_path_event     // Multipath event (new, validated, failed, closed)
```

### Error codes

QUIC transport errors (`quiche_error`):
- `QUICHE_ERR_DONE` (-1): No more work to do (not an error, signals loop exit)
- `QUICHE_ERR_BUFFER_TOO_SHORT` (-2): Output buffer too small
- `QUICHE_ERR_INVALID_STATE` (-6): Operation not valid in current state
- `QUICHE_ERR_INVALID_STREAM_STATE` (-7): Stream not in correct state
- `QUICHE_ERR_FLOW_CONTROL` (-11): Flow control limit exceeded
- `QUICHE_ERR_STREAM_LIMIT` (-12): Stream count limit exceeded
- `QUICHE_ERR_STREAM_STOPPED` (-15): Peer sent STOP_SENDING
- `QUICHE_ERR_STREAM_RESET` (-16): Peer sent RESET_STREAM
- `QUICHE_ERR_TLS_FAIL` (-10): TLS handshake failure
- `QUICHE_ERR_CRYPTO_FAIL` (-9): Cryptographic operation failed

HTTP/3 errors (`quiche_h3_error`):
- `QUICHE_H3_ERR_DONE` (-1): No more events to process
- `QUICHE_H3_ERR_STREAM_BLOCKED` (-13): Stream blocked by flow control
- `QUICHE_H3_ERR_FRAME_UNEXPECTED` (-9): Unexpected HTTP/3 frame
- Transport errors are offset by -1000 (e.g., `QUICHE_H3_TRANSPORT_ERR_DONE` = -1001)

### HTTP/3 event types

```c
QUICHE_H3_EVENT_HEADERS        // Request/response headers received
QUICHE_H3_EVENT_DATA           // Body data available on stream
QUICHE_H3_EVENT_FINISHED       // Stream fully received (FIN)
QUICHE_H3_EVENT_GOAWAY         // Peer is shutting down gracefully
QUICHE_H3_EVENT_RESET          // Stream was reset by peer
QUICHE_H3_EVENT_PRIORITY_UPDATE // Peer changed stream priority
```


## 3. Connection Lifecycle

### Server-side: accept flow

```
1. Receive UDP packet
2. quiche_header_info()          -- parse DCID, SCID, version, token
3. quiche_version_is_supported() -- check QUIC version
4. If new connection:
   a. quiche_retry()             -- send stateless retry (anti-amplification)
   b. On retry response:
      quiche_accept()            -- create quiche_conn with validated token
5. quiche_conn_recv()            -- feed packet to connection
6. quiche_conn_is_established()  -- check if handshake complete
7. If established and no h3:
   quiche_h3_conn_new_with_transport() -- create HTTP/3 layer
```

### Client-side: connect flow

```
1. quiche_connect(server_name, scid, local, peer, config)
2. quiche_conn_send() in loop    -- generates Initial packet
3. Send via UDP
4. Receive response, feed to quiche_conn_recv()
5. Repeat send/recv until quiche_conn_is_established()
6. quiche_h3_conn_new_with_transport() -- create HTTP/3 layer
```

### Shutdown

```
quiche_conn_close(conn, app_error, error_code, reason, reason_len)
// Then continue send loop until DONE -- sends CONNECTION_CLOSE frame
// Eventually quiche_conn_is_closed() returns true
```

### Resource ownership chain

```
quiche_config (shared, long-lived)
  +-- quiche_conn (per-connection)
       |-- streams (identified by uint64_t stream_id, no separate handle)
       +-- quiche_h3_conn (optional, per-connection HTTP/3 layer)
            +-- quiche_h3_event (transient, freed after processing)
```

**NIF mapping:** `quiche_config` and `quiche_conn` become `ErlNifResourceType` with destructors. `quiche_h3_conn` either shares the connection resource or gets its own. Stream IDs are plain integers, not resources. Events are transient and processed immediately in the NIF.


## 4. Configuration API

### QUIC transport config (quiche_config)

```c
quiche_config *quiche_config_new(QUICHE_PROTOCOL_VERSION);

// TLS
quiche_config_load_cert_chain_from_pem_file(config, path);
quiche_config_load_priv_key_from_pem_file(config, path);
quiche_config_load_verify_locations_from_file(config, ca_path);
quiche_config_verify_peer(config, true);

// ALPN (MUST set for HTTP/3)
quiche_config_set_application_protos(config,
    (uint8_t *)QUICHE_H3_APPLICATION_PROTOCOL,
    sizeof(QUICHE_H3_APPLICATION_PROTOCOL) - 1);

// Transport parameters
quiche_config_set_max_idle_timeout(config, 30000);          // ms
quiche_config_set_initial_max_data(config, 10000000);       // bytes
quiche_config_set_initial_max_stream_data_bidi_local(config, 1000000);
quiche_config_set_initial_max_stream_data_bidi_remote(config, 1000000);
quiche_config_set_initial_max_streams_bidi(config, 100);
quiche_config_set_initial_max_streams_uni(config, 100);

// Congestion control
quiche_config_set_cc_algorithm(config, QUICHE_CC_BBR2_GCONGESTION);

// Datagrams (RFC 9221, foundation for WebTransport)
quiche_config_enable_dgram(config, true, recv_queue, send_queue);

// PMTU discovery
quiche_config_discover_pmtu(config, true);

quiche_config_free(config);  // when done
```

### HTTP/3 config (quiche_h3_config)

```c
quiche_h3_config *quiche_h3_config_new();
quiche_h3_config_set_max_field_section_size(h3_config, 16384);
quiche_h3_config_set_qpack_max_table_capacity(h3_config, 4096);
quiche_h3_config_set_qpack_blocked_streams(h3_config, 100);
quiche_h3_config_free(h3_config);
```


## 5. Packet I/O

### Receiving packets

```c
// recv_info tells quiche where the packet came from/to
quiche_recv_info recv_info = {
    .from = (struct sockaddr *)&peer_addr,
    .from_len = peer_addr_len,
    .to = (struct sockaddr *)&local_addr,
    .to_len = local_addr_len,
};

ssize_t done = quiche_conn_recv(conn, buf, buf_len, &recv_info);
// done < 0: error (check quiche_error)
// done >= 0: bytes consumed
```

### Sending packets

```c
quiche_send_info send_info;

// Call in a loop until QUICHE_ERR_DONE
while (1) {
    ssize_t written = quiche_conn_send(conn, out, sizeof(out), &send_info);
    if (written == QUICHE_ERR_DONE) break;
    if (written < 0) { /* handle error */ break; }

    // send_info.at tells you WHEN to send (for pacing)
    // send_info.from/to tell you the path
    sendto(sock, out, written, 0,
           (struct sockaddr *)&send_info.to, send_info.to_len);
}
```

### Send quantum (GSO batching hint)

```c
size_t quantum = quiche_conn_send_quantum(conn);
// Allocate output buffer of at least `quantum` bytes for GSO
// efficiency. On Linux with GSO, this enables sendmmsg batching.
```


## 6. Timer Management

quiche does NOT manage timers internally. The application MUST:

```c
// After every recv/send cycle, check the next timeout
uint64_t timeout_ms = quiche_conn_timeout_as_millis(conn);
// Schedule a timer for timeout_ms from now

// When the timer fires:
quiche_conn_on_timeout(conn);
// Then call send loop again -- on_timeout may generate packets
// (retransmissions, keepalives, loss probes)
```

**BEAM pattern:** Use `erlang:send_after(TimeoutMs, self(), quic_timeout)` after each recv/send cycle. On receiving `quic_timeout`, call `quiche_conn_on_timeout` via NIF, then flush the send loop.

**Critical:** If the application fails to call `on_timeout`, the connection will never retransmit lost packets, never detect idle timeout, and never complete handshakes that need retries.


## 7. HTTP/3 Event Processing

### Event poll loop

```c
quiche_h3_event *ev;
while (1) {
    int64_t stream_id = quiche_h3_conn_poll(h3_conn, conn, &ev);
    if (stream_id < 0) break;  // QUICHE_H3_ERR_DONE

    switch (quiche_h3_event_type(ev)) {
        case QUICHE_H3_EVENT_HEADERS:
            // Extract headers via callback
            quiche_h3_event_for_each_header(ev, header_cb, userdata);
            break;

        case QUICHE_H3_EVENT_DATA:
            // Read body data
            quiche_h3_recv_body(h3_conn, conn, stream_id, buf, buf_len);
            break;

        case QUICHE_H3_EVENT_FINISHED:
            // Request fully received, send response
            break;

        case QUICHE_H3_EVENT_GOAWAY:
            // Peer shutting down, stop sending new requests
            break;

        case QUICHE_H3_EVENT_RESET:
            // Stream was reset by peer
            break;
    }
    quiche_h3_event_free(ev);
}
```

### Sending HTTP/3 responses

```c
quiche_h3_header headers[] = {
    {.name = (uint8_t *)":status", .name_len = 7,
     .value = (uint8_t *)"200",    .value_len = 3},
    {.name = (uint8_t *)"content-type", .name_len = 12,
     .value = (uint8_t *)"text/plain",  .value_len = 10},
};

// Send response headers
quiche_h3_send_response(h3_conn, conn, stream_id,
                        headers, 2, false);  // fin=false: body follows

// Send response body
quiche_h3_send_body(h3_conn, conn, stream_id,
                    body, body_len, true);  // fin=true: end of response
```

### Sending HTTP/3 requests (client)

```c
quiche_h3_header headers[] = {
    {.name = (uint8_t *)":method",    .name_len = 7,
     .value = (uint8_t *)"GET",       .value_len = 3},
    {.name = (uint8_t *)":path",      .name_len = 5,
     .value = (uint8_t *)"/index.html", .value_len = 11},
    {.name = (uint8_t *)":authority", .name_len = 10,
     .value = (uint8_t *)"example.com", .value_len = 11},
    {.name = (uint8_t *)":scheme",    .name_len = 7,
     .value = (uint8_t *)"https",     .value_len = 5},
};

int64_t stream_id = quiche_h3_send_request(h3_conn, conn,
                                            headers, 4, true);
```


## 8. Stream Management

QUIC streams are identified by `uint64_t` stream IDs. No separate handle type.

### Stream ID semantics (RFC 9000)

- Bit 0: initiator (0 = client-initiated, 1 = server-initiated)
- Bit 1: directionality (0 = bidirectional, 1 = unidirectional)
- Client bidi: 0, 4, 8, 12, ...
- Server bidi: 1, 5, 9, 13, ...
- Client uni: 2, 6, 10, 14, ...
- Server uni: 3, 7, 11, 15, ...

### Reading from streams

```c
// Get iterator of readable streams
quiche_stream_iter *iter = quiche_conn_readable(conn);
uint64_t stream_id;
while (quiche_stream_iter_next(iter, &stream_id)) {
    bool fin = false;
    ssize_t len = quiche_conn_stream_recv(conn, stream_id,
                                          buf, sizeof(buf), &fin, NULL);
    if (fin) { /* stream fully received */ }
}
quiche_stream_iter_free(iter);
```

### Writing to streams

```c
ssize_t written = quiche_conn_stream_send(conn, stream_id,
                                          data, data_len, fin, NULL);
// written < 0: error
// written >= 0: bytes accepted (may be less than data_len due to flow control)
```

### Flow control queries

```c
ssize_t cap = quiche_conn_stream_capacity(conn, stream_id);
// cap = how many bytes can be written now before flow control blocks
```


## 9. Datagram API (RFC 9221)

Foundation for WebTransport (phase 2).

```c
// Check if datagrams are enabled
ssize_t max_len = quiche_conn_dgram_max_writable_len(conn);

// Send
quiche_conn_dgram_send(conn, data, data_len);

// Receive
ssize_t len = quiche_conn_dgram_recv(conn, buf, sizeof(buf));
```


## 10. Connection Demultiplexing

A single UDP socket serves all connections. Inbound packets must be routed to the correct `quiche_conn` by Destination Connection ID (DCID).

```c
// Parse packet header to extract DCID
quiche_header_info(buf, buf_len, LOCAL_CONN_ID_LEN,
                   &version, &type,
                   scid, &scid_len,
                   dcid, &dcid_len,
                   token, &token_len);

// Look up connection by DCID in your connection table
conn = lookup_by_dcid(dcid, dcid_len);
```

**BEAM pattern:** The listener process owns the UDP socket, parses DCID with a thin NIF call to `quiche_header_info`, looks up the connection process via ETS or Registry, and forwards the raw packet. The connection process then calls `quiche_conn_recv`.


## 11. Anti-Amplification and Stateless Retry

Servers MUST implement stateless retry to prevent amplification attacks:

```
1. Receive Initial packet from unknown DCID
2. Generate token = encrypt(client_addr, odcid)
3. quiche_retry(scid, dcid, new_scid, token, version, out)
4. Send retry packet
5. Client resends Initial with token
6. Server validates token, calls quiche_accept(scid, odcid, ...)
```

The token binds the client's address to the original DCID, preventing IP spoofing amplification.


## 12. Key Constants

```c
QUICHE_PROTOCOL_VERSION     0x00000001   // QUIC v1 (RFC 9000)
QUICHE_MAX_CONN_ID_LEN      20           // Max Connection ID length
QUICHE_MIN_CLIENT_INITIAL_LEN 1200       // Min Initial packet size
QUICHE_H3_APPLICATION_PROTOCOL "\x02h3"  // ALPN for HTTP/3
```


## 13. Build Requirements

- Rust 1.88+ (quiche is a Rust library with C FFI)
- `--features ffi` flag to cargo to enable the C API
- BoringSSL bundled (first build compiles it, needs cmake + Go)
- Produces `libquiche.a` static library + `quiche.h` header
- Link with: `-lquiche -lm -lpthread` (Linux: add `-lrt`)
- quiche version: 0.24.5 (latest as of August 2025)
