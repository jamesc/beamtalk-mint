# mint

A pure-Beamtalk HTTP client — a port of [Mint](https://hexdocs.pm/mint)
(HTTP/1.1 and HTTP/2) with no native dependencies beyond `hpack`. It replaces
`gun` under `beamtalk-http` ([BT-4088](https://linear.app/beamtalk/issue/BT-4088/pure-beamtalk-http-client-mint-port-under-beamtalk-http)).

The connection is a processless `Value`: every call threads it through
and answers an updated copy, exactly like Mint threads its `conn`.

```beamtalk
conn := (MintHTTP1 connect: "127.0.0.1" port: 8080) unwrap.
{live, ref} := (conn
  request: "GET"
  target: "/"
  headers: #("x-echo: ping")
  body: "") unwrap.
{done, parts} := (live stream: ref timeout: 5000) unwrap.
// parts: {#status, ref, 200}, {#headers, ref, headers},
//        {#data, ref, bytes}, {#done, ref}
done close.
```

## API

- `MintHTTP1 connect:host port:port [options:]` — cleartext; options: `#timeout` (ms, default 5000).
- `MintHTTP1 connectTls:host port:port options:` — TLS via `ssl`
  (`#verify`, default `#verify_none`; `#alpn`, default `#("http/1.1")`).
  A missing ALPN negotiation proceeds as HTTP/1.1; a *different*
  protocol is an error.
- `request:method target:target headers:headers body:body` — one request
  in flight; answers `{conn, ref}`.
- `stream:ref timeout:ms` — the full response as Mint parts; the
  connection stays open for reuse unless the response says
  `connection: close` (HTTP/1.0 without `keep-alive` too).
- `close`, `isOpen`, `openRequestCount`.
- `MintHeader` — `name` (lowercase String) and `value`
  (String, or opaque Binary for non-UTF-8 bytes).
- `MintHTTP1 buffered:` + `withInFlightRef:`/`withInFlightMethod:` —
  test-only parsing over preset bytes.

Bodies are framed per RFC 9112: method/status rules, chunked (extensions
ignored, trailers consumed), `content-length` (conflicts rejected), or
close-delimited.

`MintHTTP2` speaks prior-knowledge cleartext `h2c` (`connect:host
port:`) and TLS with ALPN `h2` (`connectTls:host port:options:`, where
a missing or different protocol is an error). Requests multiplex —
each answers its stream id as the reference, and `stream:ref` reads
until that stream completes while buffering the others. Connection and
per-stream flow-control windows start at 65535 (or the server's
`initial_window_size`); request bodies that do not fit wait for
`WINDOW_UPDATE`s, and receive windows are replenished once
half-consumed. Peer `SETTINGS` are applied (`initial_window_size`
re-scales open streams, `header_table_size` resizes the HPACK encoder,
`max_frame_size` caps sends, `max_concurrent_streams` caps new
requests); `PING`s are acknowledged and `RST_STREAM` on the streamed
reference answers `{#reset, code}`.

## Building

```bash
just build
```

## Testing

```bash
just clean test
```

`MintParseTest` parses preset buffers (chunked, pipelining, HEAD/204,
interim `1xx`, malformed input, ALPN checks). `MintConnectionTest` and
`MintTlsTest` run against a local Buccaneer server (plain and HTTPS).
`MintH2FrameTest` covers the frame codec hermetically;
`MintHTTP2Test` drives settings negotiation, multiplexed streams,
flow control, and TLS ALPN against the `MintH2TestServer` loopback
H2 peer.
