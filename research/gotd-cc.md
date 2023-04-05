# PROJECT BRIEF: gotd/td

> **Change log**: 2026-04-24 — initial version.

---

## 1. TL;DR

**gotd/td** (`github.com/gotd/td`) is a production-grade **Telegram MTProto 2.0 API client**
written in pure Go. It targets bot and user-account automation on top of Telegram's binary
protocol. The stack is Go 1.25+, code-generated types from TL schemas, AES-IGE encryption
via a vendored `ige` library, and zero external C/C++ dependencies. It is open-source (MIT),
deployed as a library (not a standalone service), and tested against real Telegram servers in
CI. The main risk is **timing side-channels in RSA hash comparison** and
**incomplete DH parameter validation**.

---

## 2. Glossary

| Term | Meaning in this codebase |
|------|--------------------------|
| **Layer** | Telegram API schema version number; each schema update bumps the layer |
| **TL** | Type Language — Telegram's IDL for defining API types and RPC methods |
| **MTProto** | Mobile Transport Protocol — Telegram's encrypted binary RPC protocol |
| **DC** | Data Center — one of Telegram's geographically distributed server clusters |
| **Auth key** | 2048-bit key negotiated via DH; encrypts all session traffic |
| **PFS** | Perfect Forward Secrecy — mode where a temporary key wraps every session |
| **Salt** | 64-bit server-side nonce rotated periodically to prevent replay |
| **Invoker** | Interface `Invoke(ctx, encoder, decoder) error` — the core RPC call abstraction |
| **Middleware** | Wrapper around `Invoker` for cross-cutting concerns (flood-wait, tracing) |
| **Flood wait** | Telegram rate-limit response: retry after N seconds |
| **CDN** | Content Delivery Network nodes for media downloads |
| **SRP** | Secure Remote Password — used for 2FA password verification |
| **Obfuscated2** | Transport-layer obfuscation for MTProxy connections |
| **gotdgen** | Code generator: TL schema → Go source |
| **tg package** | ~2800 generated files representing all Telegram API types and methods |

---

## 3. Quick start

```bash
# Clone (already present in research/client/gotd)
cd research/client/gotd

# Dependencies
go mod download

# Run code generation (optional — generated code is committed)
go generate ./...

# Run all tests (unit only, no Telegram account needed)
go test --timeout 5m ./...

# Run a specific example (needs APP_ID, APP_HASH env vars)
export APP_ID=12345
export APP_HASH=abcdef0123456789abcdef0123456789
go run ./examples/bot-echo
```

The fastest path to a meaningful commit: pick any `TODO` in `crypto/` or `tgtest/`,
fix it, run `go test ./...`, and submit a PR following Conventional Commits.

---

## 4. C4: Context

```mermaid
C4Context
    title System Context — gotd/td

    Person(dev, "Developer", "Integrates gotd into their bot or user-automation app")
    System(gotd, "gotd/td library", "Pure-Go MTProto 2.0 client library")
    System_Ext(tg, "Telegram Servers", "DCs 1-5 + test DCs, CDN nodes")
    System_Ext(dns, "Telegram DNS Config", "TXT records with encrypted DC list")
    System_Ext(mtproxy, "MTProxy", "Optional obfuscation proxy")

    Rel(dev, gotd, "Imports as Go module")
    Rel(gotd, tg, "MTProto over TCP/WS, AES-IGE encrypted")
    Rel(gotd, dns, "DNS TXT lookup for DC config (RSA-signed)")
    Rel(gotd, mtproxy, "Obfuscated2 transport (optional)")
```

gotd is a **library**, not a service. Developers import it and build their own bots
or automation. All network traffic goes to Telegram DCs over MTProto.

---

## 5. C4: Containers

```mermaid
C4Container
    title Container Diagram — gotd/td

    Container(telegram, "telegram", "Go package", "High-level client: auth, pools, DC migration, helpers")
    Container(mtproto, "mtproto", "Go package", "Single MTProto connection lifecycle, ping, ack, salts")
    Container(pool, "pool", "Go package", "Connection pooling per DC, session sync")
    Container(crypto, "crypto", "Go package", "AES-IGE, RSA, DH, SRP, PRNG")
    Container(exchange, "exchange", "Go package", "Diffie-Hellman key exchange flows")
    Container(transport, "transport", "Go package", "TCP codecs: abridged, full, padded, obfuscated, WS")
    Container(bin, "bin", "Go package", "TL binary serialization/deserialization")
    Container(tg, "tg", "Generated Go package", "~2800 files: all Telegram API types & RPC methods")
    Container(session, "session", "Go package", "Pluggable session storage: file, memory, tdesktop")
    Container(gen, "gen + gotdgen", "Go package + CLI", "TL schema → Go code generator")
    Container(updates, "telegram/updates", "Go package", "Update sequencing, gap detection, state machine")
    Container(downloader, "telegram/downloader", "Go package", "Multi-stream file download with CDN support")
    Container(uploader, "telegram/uploader", "Go package", "Multi-stream file upload")

    Rel(telegram, mtproto, "Creates and manages connections")
    Rel(telegram, pool, "Acquires/releases connections per DC")
    Rel(telegram, updates, "Delegates update handling")
    Rel(telegram, downloader, "File downloads")
    Rel(telegram, uploader, "File uploads")
    Rel(mtproto, crypto, "Encrypt/decrypt messages")
    Rel(mtproto, exchange, "Key exchange on connect")
    Rel(mtproto, transport, "Sends/receives framed bytes")
    Rel(mtproto, bin, "Serialize/deserialize TL objects")
    Rel(pool, session, "Persist/restore session state")
    Rel(gen, tg, "Generates")
```

| Container | Technology | Purpose | Owner |
|-----------|-----------|---------|-------|
| `telegram` | Go | High-level API, auth, DC migration, reconnects | gotd team |
| `mtproto` | Go | Single-connection MTProto lifecycle | gotd team |
| `pool` | Go | Connection pooling, session sync | gotd team |
| `crypto` | Go + `gotd/ige` | All cryptographic primitives | gotd team |
| `exchange` | Go | DH key exchange (client & server flows) | gotd team |
| `transport` | Go | TCP/WS codec layer | gotd team |
| `bin` | Go | Binary (de)serialization, zero-alloc | gotd team |
| `tg` | Generated Go | Telegram API types (layer-versioned) | auto-generated |
| `session` | Go | Pluggable session persistence | gotd team |
| `gen`/`gotdgen` | Go + text/template | Code generation pipeline | gotd team |
| `updates` | Go | Sequenced update delivery, gap fill | gotd team |
| `downloader` | Go | CDN-aware file downloads | gotd team |
| `uploader` | Go | Chunked file uploads | gotd team |

---

## 6. C4: Components

### 6.1 telegram package

```mermaid
C4Component
    title Components — telegram package

    Component(client, "Client", "Struct", "Main entry point: NewClient(), Run(), Invoke()")
    Component(invoker, "Invoker pipeline", "Interface chain", "Middleware → invokeDirect → invokeConn/invokeMigrate")
    Component(auth, "auth", "Sub-package", "Bot auth, user auth (phone+code+2FA), QR login")
    Component(updmgr, "updates.Manager", "Sub-package", "Gap detection, state machine, sequencing")
    Component(dl, "downloader", "Sub-package", "CDN redirect, multi-part download")
    Component(ul, "uploader", "Sub-package", "Multi-part chunked upload")
    Component(msg, "message", "Sub-package", "Message builders, HTML/Markdown styling")
    Component(peers, "peers", "Sub-package", "Peer resolution, caching, access hash management")
    Component(dcs, "dcs", "Sub-package", "Hardcoded DC list, DNS config, prod/test/staging")

    Rel(client, invoker, "Delegates RPC calls")
    Rel(client, auth, "Authentication flows")
    Rel(client, updmgr, "Update handling")
    Rel(client, dl, "Downloads")
    Rel(client, ul, "Uploads")
    Rel(client, dcs, "DC resolution")
    Rel(invoker, peers, "Peer lookups in middleware")
```

### 6.2 mtproto package

```mermaid
C4Component
    title Components — mtproto package

    Component(conn, "Conn", "Struct", "Single MTProto connection: connect, readLoop, pingLoop")
    Component(rpc, "rpc.Engine", "Internal", "Request-response matching, retries, ack tracking")
    Component(handler, "MessageHandler", "Interface", "Routes decoded messages by type")
    Component(salts, "salts.Salts", "Internal", "Server salt rotation and caching")
    Component(msgid, "MessageID", "Internal", "Monotonic message ID generation")

    Rel(conn, rpc, "Dispatches requests")
    Rel(conn, handler, "Dispatches incoming messages")
    Rel(conn, salts, "Gets current salt for encryption")
    Rel(conn, msgid, "Generates message IDs")
```

---

## 7. Data flows

### 7.1 Client initialization and connection

```mermaid
sequenceDiagram
    participant App as Application
    participant C as telegram.Client
    participant P as pool.Pool
    participant M as mtproto.Conn
    participant E as exchange
    participant T as Telegram DC

    App->>C: NewClient(appID, appHash, opts)
    App->>C: Run(ctx, callback)
    C->>C: restoreConnection() [load session]
    C->>C: reconnectUntilClosed()

    loop Until context cancelled
        C->>P: acquire(dcID)
        P->>M: new Conn (or reuse free)
        M->>T: TCP connect
        alt No auth key (first time)
            M->>E: Client key exchange
            E->>T: req_pq_multi
            T-->>E: ResPQ (pq, server_nonce)
            Note over E: PQ factorization, DH
            E->>T: req_DH_params
            T-->>E: server_DH_params
            E->>T: set_client_DH_params
            T-->>E: dh_gen_ok → auth_key
        end
        M->>M: start readLoop, pingLoop, ackLoop, saltLoop
        alt PFS enabled
            M->>T: auth.bindTempAuthKey
        end
        M-->>C: ready
        C->>App: callback(ctx)
    end
```

**Trust boundary**: all data after key exchange is AES-IGE encrypted.
The auth key is the trust root — its compromise exposes the entire session.

_Source_: `telegram/connect.go:130-208`, `mtproto/conn.go:207-255`, `exchange/client_flow.go`.

### 7.2 Authentication (user account)

```mermaid
sequenceDiagram
    participant App as Application
    participant Auth as telegram/auth
    participant C as telegram.Client
    participant T as Telegram DC

    App->>Auth: auth.NewFlow(UserAuthenticator, opts)
    Auth->>App: PhoneNumber() prompt
    App-->>Auth: "+1234567890"
    Auth->>C: auth.SendCode(phone)
    C->>T: MTProto: auth.sendCode
    T-->>C: auth.SentCode {phone_code_hash}
    Auth->>App: Code() prompt
    App-->>Auth: "12345"
    Auth->>C: auth.SignIn(phone, code, hash)
    C->>T: MTProto: auth.signIn
    alt 2FA required
        T-->>C: SESSION_PASSWORD_NEEDED
        Auth->>App: Password() prompt
        App-->>Auth: "mypassword"
        Note over Auth: SRP: PBKDF2-SHA512 + SHA256
        Auth->>C: auth.CheckPassword(SRP input)
        C->>T: MTProto: auth.checkPassword
    end
    T-->>C: auth.Authorization {user}
    Auth-->>App: Authenticated
```

**Trust boundary**: phone code and password never leave the device unencrypted.
SRP ensures the password itself is never sent — only a zero-knowledge proof.

_Source_: `telegram/auth/flow.go:50-113`, `telegram/auth/user.go`, `crypto/srp/srp.go`.

### 7.3 RPC invocation pipeline

```mermaid
sequenceDiagram
    participant App as Application
    participant Inv as Invoke pipeline
    participant MW as Middleware chain
    participant Conn as mtproto.Conn
    participant RPC as rpc.Engine
    participant T as Telegram DC

    App->>Inv: client.API().MessagesSendMessage(ctx, req)
    Inv->>MW: middleware[0].Invoke()
    MW->>MW: middleware[1..N].Invoke()
    MW->>Inv: invokeDirect()
    alt DC migration error
        Inv->>Inv: invokeMigrate(targetDC)
        Note over Inv: acquires connection to new DC
    end
    Inv->>Conn: invokeConn(primaryConn)
    Conn->>RPC: rpc.Do(req)
    RPC->>Conn: writeContentMessage(encoded)
    Conn->>T: encrypted MTProto frame
    T-->>Conn: encrypted response
    Conn->>RPC: handleResult(msgID, response)
    RPC-->>App: decoded response
```

_Source_: `telegram/invoke.go:24-90`, `mtproto/rpc.go:14-48`.

### 7.4 File download with CDN

```mermaid
sequenceDiagram
    participant App as Application
    participant DL as downloader.Downloader
    participant C as telegram.Client
    participant DC as Telegram DC
    participant CDN as Telegram CDN

    App->>DL: Download(ctx, location, writer)
    loop For each part (default 512KB)
        DL->>C: upload.getFile(location, offset, limit)
        C->>DC: MTProto request
        alt CDN redirect
            DC-->>C: upload.fileCdnRedirect {cdn_token, key, iv}
            C->>CDN: upload.getCdnFile(cdn_token, offset, limit)
            CDN-->>C: upload.cdnFile {bytes}
            Note over DL: Decrypt with AES-CTR using cdn key/iv
            Note over DL: Verify hash from DC
        else Direct
            DC-->>C: upload.file {bytes}
        end
        DL->>App: write part to writer
    end
```

**Trust boundary**: CDN nodes are semi-trusted — files are AES-CTR encrypted,
and hashes are verified against the origin DC.

_Source_: `telegram/download.go:64-94`, `telegram/downloader/downloader.go`.

### 7.5 Update processing

```mermaid
sequenceDiagram
    participant T as Telegram DC
    participant Conn as mtproto.Conn
    participant Mgr as updates.Manager
    participant State as state machine
    participant App as Application handler

    T->>Conn: encrypted update message
    Conn->>Conn: handleMessage() → route by type
    Conn->>Mgr: handler.OnMessage(update)
    Mgr->>State: state.Push(pts, qts, seq, update)
    alt Gap detected (missing seq)
        State->>T: updates.getDifference / channels.getChannelDifference
        T-->>State: missed updates
    end
    State->>App: handler.Handle(ctx, updates)
```

**Important invariant** (`telegram/updates/manager.go:22-27`):
updates may contain **negative Pts/Qts/Seq** values. These must NOT be used in handlers.

_Source_: `mtproto/handle_message.go`, `telegram/updates/manager.go:84-162`.

---

## 8. Deployment / runtime topology

gotd/td is a **library**, not a deployable service. The runtime topology depends on the
consuming application. A typical deployment:

```mermaid
graph LR
    subgraph "User's Infrastructure"
        App["Application process<br/>(imports gotd/td)"]
        SessionFile["Session file<br/>(0600 perms)"]
    end

    subgraph "Telegram Infrastructure"
        DC1["DC 1<br/>149.154.175.53:443"]
        DC2["DC 2<br/>149.154.167.50:443"]
        DC3["DC 3<br/>149.154.175.100:443"]
        DC4["DC 4<br/>149.154.167.91:443"]
        DC5["DC 5<br/>91.108.56.128:443"]
        CDN["CDN nodes"]
    end

    subgraph "Optional"
        MTProxy["MTProxy"]
    end

    App -->|"MTProto/TCP"| DC2
    App -.->|"DC migration"| DC1
    App -.->|"DC migration"| DC4
    App -.->|"CDN download"| CDN
    App -.->|"Obfuscated2"| MTProxy
    MTProxy -->|"TCP"| DC2
    App -->|"read/write"| SessionFile
```

DC endpoints are **hardcoded** in `telegram/dcs/prod.go`. DNS-based config is RSA-signed.

---

## 9. Dependencies and integrations

| Dependency | Version | Purpose | Criticality | Fallback |
|-----------|---------|---------|-------------|----------|
| `gotd/ige` | v0.2.2 | AES-IGE block cipher | **Critical** — core encryption | None (protocol-mandated) |
| `gotd/tl` | v0.4.0 | TL schema parser | Build-time only | None |
| `gotd/getdoc` | v0.52.0 | Doc extraction for codegen | Build-time only | Generate without docs |
| `cenkalti/backoff/v4` | v4.3.0 | Exponential backoff | High — reconnection logic | Replace with custom |
| `go-faster/errors` | v0.7.1 | Error wrapping with frames | Medium | `fmt.Errorf` |
| `go-faster/xor` | v1.0.0 | Fast XOR for crypto | High — encryption path | `crypto/subtle.XORBytes` |
| `coder/websocket` | v1.8.14 | WebSocket transport | Medium — WASM support | TCP-only |
| `klauspost/compress` | v1.18.5 | GZIP for message compression | Medium | `compress/gzip` |
| `otel/*` | v1.43.0 | OpenTelemetry tracing | Low — observability | Disable tracing |
| `uber/zap` | v1.27.1 | Structured logging | Low — debug | `log/slog` |
| `golang.org/x/crypto` | v0.49.0 | SHA, PBKDF2, SRP primitives | **Critical** | None |
| `golang.org/x/sync` | v0.20.0 | Concurrency primitives | High | Custom sync |
| `stretchr/testify` | v1.11.1 | Test assertions | Test-only | `testing` stdlib |

---

## 10. Hot files map

Based on git history, these files are touched by nearly every significant change:

| File | Description |
|------|-------------|
| `telegram/client.go` | Core Client struct, NewClient(), Run() |
| `telegram/options.go` | All configurable client options |
| `telegram/connect.go` | Reconnection loop, session restore |
| `telegram/invoke.go` | RPC invocation pipeline, DC migration |
| `telegram/pool.go` | DC sub-connection management |
| `mtproto/conn.go` | MTProto connection lifecycle |
| `mtproto/rpc.go` | RPC dispatch, sequence numbers |
| `mtproto/handle_message.go` | Message type routing |
| `pool/pool.go` | Connection pool acquire/release/dead |
| `pool/session.go` | Session state, DC migration |
| `crypto/cipher_encrypt.go` | Message encryption |
| `crypto/cipher_decrypt.go` | Message decryption + validation |
| `exchange/client_flow.go` | DH key exchange (client side) |
| `bin/buffer.go` | Core binary buffer |
| `telegram/auth/flow.go` | Authentication state machine |
| `telegram/updates/manager.go` | Update sequencing engine |
| `tg/tl_update_gen.go` | Generated update types (changes every layer) |
| `go.mod` | Dependency versions |

---

## 11. Reading order

For a new engineer to understand the system in ~1 day:

1. **`ARCHITECTURE.md`** — 4-layer mental model (2 min)
2. **`telegram/client.go`** — Client struct, fields, NewClient() (15 min)
3. **`telegram/options.go`** — What's configurable (10 min)
4. **`telegram/connect.go`** — Connection lifecycle, Run(), reconnects (15 min)
5. **`telegram/invoke.go`** — RPC pipeline, DC migration (10 min)
6. **`telegram/middleware.go`** — Middleware wrapping pattern (5 min)
7. **`pool/pool.go`** — Connection pooling strategy (15 min)
8. **`pool/session.go`** — Session state management (10 min)
9. **`mtproto/conn.go`** — Single connection lifecycle (20 min)
10. **`mtproto/rpc.go`** — Request dispatch, sequence numbers (10 min)
11. **`mtproto/handle_message.go`** — Message type routing (5 min)
12. **`exchange/client_flow.go`** — DH key exchange (20 min)
13. **`crypto/cipher_encrypt.go` + `cipher_decrypt.go`** — AES-IGE envelope (15 min)
14. **`telegram/auth/flow.go`** — Auth state machine (10 min)
15. **`telegram/updates/manager.go`** — Update engine (15 min)
16. **`bin/buffer.go` + `bin/decode.go`** — Binary protocol (10 min)
17. **`examples/bot-echo/main.go`** — Simplest working example (5 min)
18. **`gen/make_structures.go`** (skim) — How types are generated (10 min)
19. **`telegram/downloader/downloader.go`** — CDN download flow (10 min)
20. **`tgtest/`** (skim) — How the mock Telegram server works (10 min)

---

## 12. Invariants and gotchas

1. **64-bit alignment**: `connsCounter` in `telegram/client.go:58-60` must be
   the first field in the struct for atomic operations on 32-bit platforms.
   Moving it will cause panics.

2. **PFS ordering**: In PFS mode, the read loop **must** start before the callback
   because `auth.bindTempAuthKey` needs the read loop to process the response
   (`mtproto/conn.go:234-239`).

3. **Content messages increment seqno**: Every `Invoke()` assumes the call is a
   content message and increments the sequence number (`mtproto/rpc.go:16`).
   Non-content messages would break sequence tracking.

4. **Negative Pts/Qts/Seq in updates**: Telegram may send negative values in
   update objects. These are **protocol artifacts** and must not be used in
   application logic (`telegram/updates/manager.go:22-27`).

5. **Upload part size constraints**: `partSize % 1024 == 0` AND
   `MaxPartSize % partSize == 0` (`telegram/uploader/part.go:19-22`).
   Violating this causes silent upload corruption.

6. **Session file is single-process**: The mutex in `session/storage_file.go`
   protects in-process concurrency only. Two processes sharing the same session
   file will corrupt it. `[ASSUMPTION]`

7. **DC endpoints are hardcoded**: `telegram/dcs/prod.go` contains static IP
   addresses. DNS-based discovery exists but is RSA-signature-verified.

8. **Salt expiration**: If server salt expires mid-session, MTProto returns
   `BadServerSalt`. The client automatically fetches a new salt and retries
   (`mtproto/rpc.go`). Do not treat this as a fatal error.

9. **Flood wait is retriable**: `FLOOD_WAIT_X` errors include a retry-after
   duration. The standard middleware handles this automatically. Custom invokers
   must respect it or risk account bans.

10. **Generated code is committed**: The `tg/` package (~2800 files) is checked
    into git. Running `go generate` is only needed after schema updates.
    Do not manually edit `tl_*_gen.go` files.

11. **Non-PFS dial timeout**: In non-PFS mode, the dial timeout covers both
    TCP connect AND key exchange (`mtproto/connect.go:19-23`). In PFS mode,
    key exchange has its own timeout.

12. **CDN connections are caller-managed**: `telegram/download.go:64-78` returns
    a closer for CDN pools. The downloader must NOT close CDN connections —
    the caller controls their lifecycle.

---

## 13. Security findings

### Critical: None found

### High: None found

### Medium

#### S-01: Timing side-channel in RSADecryptHashed
- **Category**: STRIDE/Information Disclosure; CWE-208
- **Severity**: Medium (impact: key material leakage; likelihood: low — requires
  network timing precision)
- **File**: `crypto/rsa_hashed.go:50-58`
- **Evidence**: Loop iterates `for i := 0; i <= len(paddedData); i++` with
  early `return` on `bytes.Equal(h[:], hash)` — variable iteration count
  leaks padding length through timing.
- **Exploit**: Attacker measures decryption time across many attempts to determine
  valid padding boundaries, reducing key exchange search space.
- **Recommendation**: Always iterate all positions; use constant-time comparison
  from `crypto/subtle.ConstantTimeCompare`.

#### S-02: Timing side-channel in RSAPad hash verification
- **Category**: STRIDE/Information Disclosure; CWE-208
- **Severity**: Medium (same rationale as S-01)
- **File**: `crypto/rsa_pad.go:155`
- **Evidence**: `bytes.Equal(hash, h.Sum(nil))` — while Go's `bytes.Equal` is
  constant-time for equal lengths, the surrounding control flow may leak timing.
- **Recommendation**: Use `subtle.ConstantTimeCompare` explicitly for clarity
  and defense-in-depth.

#### S-03: Incomplete DH parameter range check (documented FIXME)
- **Category**: STRIDE/Tampering; OWASP A02 (Cryptographic Failures)
- **Severity**: Medium (impact: weakened key exchange; likelihood: very low —
  server-controlled parameters)
- **File**: `crypto/check_dh.go:26-27`
- **Evidence**: `// FIXME(tdakkota): we check that 2^2047 <= p < 2^2048 but docs
  says to check 2^2047 < p < 2^2048.` Implementation uses `p.BitLen() != 2048`
  which accepts `p = 2^2047` (exactly 2048 bits, but spec requires strict `>`).
  Note: TDLib uses the same check (linked in comment on line 30).
- **Exploit**: A malicious server could supply `p = 2^2047` (exact boundary),
  which Telegram spec explicitly excludes.
- **Recommendation**: Add explicit check `p.Cmp(two2047) > 0` in addition to bit length.

#### S-04: Missing gB range validation in server-side DH flow
- **Category**: STRIDE/Tampering; OWASP A02
- **Severity**: Medium (impact: weak shared secret; likelihood: low — affects
  `tgtest` server, not production client path)
- **File**: `exchange/server_flow.go:261-266`
- **Evidence**: `gB` from client is used directly in `Exp(gB, a, dhPrime)` without
  checking `1 < gB < dhPrime - 1`.
- **Recommendation**: Add `crypto.CheckDHParams(gB, dhPrime)` before computing shared secret.

#### S-05: Session file symlink attack surface
- **Category**: STRIDE/Elevation of Privilege; CWE-59
- **Severity**: Medium (impact: session hijack; likelihood: low — requires local access)
- **File**: `session/storage_file.go:47`
- **Evidence**: `os.WriteFile(f.Path, data, 0600)` with `// TODO(tdakkota): use
  robustio/renameio?` on line 46. Permissions are correct, but no symlink
  resolution or parent directory ownership check.
- **Exploit**: Attacker with local access creates symlink at session path pointing
  to attacker-controlled file.
- **Recommendation**: Use `os.O_NOFOLLOW` or resolve symlinks before write.
  Consider atomic writes via temp file + rename.

#### S-06: DNS-based DC config (mitigated)
- **Category**: STRIDE/Spoofing; OWASP A08 (Software and Data Integrity)
- **Severity**: Medium (impact: DC redirection; likelihood: very low — RSA-signed)
- **File**: `telegram/dcs/dns.go`
- **Evidence**: DNS TXT record lookup for DC configuration. Mitigated by RSA
  signature verification with vendored public key.
- **Recommendation**: Document DNSSEC recommendation. Ensure RSA verification
  cannot be bypassed.

### Low

#### S-07: `Buffer.Skip()` lacks bounds checking
- **Category**: OWASP A03 (Injection — panic via OOB)
- **Severity**: Low (causes panic, not exploitation; callers generally check first)
- **File**: `bin/buffer.go` — `Skip(n)` does `b.Buf = b.Buf[n:]` without bounds check.
- **Recommendation**: Add `if n > len(b.Buf) { return io.ErrUnexpectedEOF }`.

#### S-08: SHA-1 usage in legacy RSA path
- **Category**: OWASP A02 (Cryptographic Failures)
- **Severity**: Low (protocol-mandated for MTProto v1 compatibility; SHA-256 used
  in v2 path)
- **File**: `crypto/rsa_hashed.go:6,27,53` — `#nosec G505` annotations present.
- **Recommendation**: No action needed. Protocol requirement. Document rationale.

#### S-09: Probabilistic primality test TODO
- **Category**: Informational
- **Severity**: Low
- **File**: `crypto/prime.go:7-12` — `// TODO: maybe it should be smaller?` for
  `probabilityN = 64` (Miller-Rabin rounds).
- **Recommendation**: 64 rounds gives ~2^-128 error probability. Remove TODO or
  document rationale. Current value is safe.

---

## 14. Open questions

1. **No formal security audit**: The project states it follows Telegram Security
   Guidelines and uses Full Disclosure, but no third-party audit has been performed.
   The crypto implementation may have subtle issues not caught by code review alone.

2. **E2E test reliability**: CI config shows `TEST_ACCOUNTS_BROKEN=1` — unclear
   whether E2E tests against real Telegram servers are currently passing.

3. **WASM/JS crypto PRNG**: `crypto/rand.go` has a JS-specific path for WASM.
   The quality of the PRNG in that environment is unknown.

4. **Secret chat implementation completeness**: `tg/e2e` package exists with
   generated types, but the level of secret chat support in the high-level API
   is unclear from code inspection alone.

5. **Multi-process session safety**: File-based session storage uses in-process
   mutexes only. Whether multi-process deployment is explicitly unsupported or
   just undocumented is unknown.

6. **CDN hash verification completeness**: The downloader supports CDN with
   hash verification, but whether all edge cases (partial chunks, retries after
   hash failure) are correctly handled requires deeper review.

7. **`go 1.25.0` requirement**: The go.mod specifies Go 1.25.0 which is
   a future/pre-release version. This may affect compatibility for consumers
   on stable Go releases.

---

## 15. Change log

| Date | Change |
|------|--------|
| 2026-04-24 | Initial document created from source analysis of gotd/td at commit `4131a53fe` |
