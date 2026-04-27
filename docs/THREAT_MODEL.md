# Threat Model — Tributary v0.1

This is a working subset of `DESIGN.md §2`, expanded with implementation
specifics. It defines what the v0.1 daemon defends against, what it
delegates to the platform, and what it explicitly does not address.

---

## Trust boundaries

```
┌────────────────────────────────────────────────────────────┐
│ Operator host (trusted)                                    │
│                                                            │
│   ┌─────────────┐    ┌──────────────┐    ┌─────────────┐  │
│   │ tributaryd  │───→│ SQLite       │    │ trib / UI   │  │
│   │   (Zig)     │    └──────────────┘    └──────┬──────┘  │
│   │             │                                │         │
│   │             │←── UDS (0600) / 127.0.0.1 ────┘         │
│   │             │      capability-token auth               │
│   └──────┬──────┘                                          │
│          │ TLS                                             │
└──────────┼─────────────────────────────────────────────────┘
           ▼
   ┌────────────────┐
   │ Feed servers   │  (untrusted)
   └────────────────┘
```

---

## Assets, in priority order

1. **Subscription list.** Discloses operator interests, targets, tradecraft.
2. **Saved queries / alert rules.** Discloses intelligence requirements.
3. **Annotations + extracted entities.** Analytical product.
4. **Item corpus.** Bulk content; lower sensitivity but can re-derive (1).
5. **Inference outputs.** Reasoning artifacts (v0.2+).

---

## Adversaries and v0.1 mitigations

| Adversary | Capability | v0.1 mitigation |
|---|---|---|
| Local non-owner user | Reads home directory, `/tmp` | Token file, UDS, port file all `0600`. Parent dirs `0700`. `umask 0077` before bind. |
| Other process on the host | Connects to UDS / loopback | Bearer-token gate on every protected route. Token never echoed, never logged, never on argv. |
| Compromised feed publisher | Hostile XML/JSON, oversized payloads, parser exploits | Streaming SAX reader, no DTD, predefined entities only, depth/element/attr/text caps, allowlist HTML sanitizer, JS scheme rejected in UI. Persistent fuzz corpus + harnesses. |
| Active network adversary | MITM, downgrade | TLS via stdlib defaults. (TLS-1.3-only enforcement is v0.4 hardening.) |
| Slow / hostile HTTP server | Slowloris, oversized body, redirect loop, content-encoding tricks | Body cap 10MB, wallclock 30s, redirect cap 5, scheme allowlist, decompression window cap, server read timeout 10s on the API surface. |
| Cloud sync provider (when used, v0.3+) | Reads sync blobs | Client-side encryption only. Provider sees ciphertext. |
| External inference provider (v0.2+, if used) | Reads prompts and content | Off by default; explicit per-call opt-in; non-suppressible warning per query. |

---

## Out of scope (v0.1)

- **Local-host root compromise.** Defeats user-scoped storage encryption
  by definition. Addressed at platform level.
- **Side-channel attacks on local inference.** Out of scope for this
  application; relevant to the inference runtime.
- **Hardware compromise.** TEMPEST, evil-maid, supply-chain firmware.
- **TLS 1.3-only enforcement.** Stdlib does not currently expose a
  `min_version` knob per-client; this lands in v0.4 hardening.
- **OS keychain integration for the capability token.** v0.1 stores the
  token as an `0600` file. Same posture as SSH keys. Migration to
  Keychain (macOS) / libsecret (Linux) lands in v0.4.

---

## Implementation specifics worth flagging

### Concurrency
- The daemon uses three threads: UDS accept, loopback accept, scheduler.
- All three share one `*storage.Db`. Connection opened with
  `SQLITE_OPEN_FULLMUTEX` so SQLite serializes per-call. Writes wrap
  multi-row work in `BEGIN IMMEDIATE` so partial writes are impossible.

### Request bounds
- Request line + headers ≤ 8 KB
- Request body ≤ 1 MB (API)
- Fetched feed body ≤ 10 MB
- XML element depth ≤ 32, element count ≤ 100k, attribute length ≤ 64 KB,
  text run ≤ 1 MB
- Sanitized HTML output ≤ 256 KB

### Error surface
- All API errors return generic JSON shapes (`{"error":"bad_request"}`,
  `{"error":"unauthorized"}`, `{"error":"internal"}`). Internal
  diagnostics are logged at `err`/`warn` level but never returned to
  the client.

### Logging policy
- Body content is never logged.
- Operational log shape: `feed_id`, status, byte count, parse outcome,
  duration.
- Authentication failures, rate-limit breaches, and rule-engine
  timeouts (when the engine ships) are logged as security-relevant.

---

*Revisions to this document precede revisions to the threat model
itself. If a change to the code requires a change to this document,
update the document first.*
