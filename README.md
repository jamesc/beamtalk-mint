# mint

A pure-Beamtalk HTTP client — a port of [Mint](https://hexdocs.pm/mint)
(HTTP/1.1 today, HTTP/2 next) with no native dependencies. It replaces
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
