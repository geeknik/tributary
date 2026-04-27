# Tributary

> Tributary is feed intelligence that doesn't phone home.
> Subscribe to anything. Run your own inference. Keep your subscription
> list, your queries, and your alerts on hardware you control.

Tributary treats RSS, Atom, and JSON Feed as the **transport layer**, not
the product. The product is a local query and rules engine over the
aggregated corpus, designed for OSINT, threat-intel, competitive-intel,
and analyst workflows that cannot afford to leak their subscription list,
query history, or extracted entities to a third party.

The reader UI is a debug surface. The product is the API and the
(forthcoming) rules engine.

See [`DESIGN.md`](DESIGN.md) for the contract this codebase implements
and [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md) for the trust
boundaries.

---

## Status

**v0.1 (in development).** Single-machine daemon: fetcher, parsers, JSON
API over UDS + loopback, scheduler, OPML import/export, minimal HTML
console. No telemetry. No phone-home. Inference bridge, embeddings,
similarity, rules engine, and at-rest encryption land in v0.2.

---

## Build

Requires Zig **0.15.2** specifically — Zig 0.16 introduced a stdlib
restructuring (`std.net` → `std.Io.net`, `std.http.Client` API rewrite)
that we have not ported to. The repo ships a project-local shim at
`./bin/zig` that resolves to a pinned 0.15.2 install:

```sh
brew install zig@0.15            # one-time install
./bin/zig build                  # build daemon + CLI via the shim
./bin/zig build test             # full test suite
./bin/zig build fuzz-rss         # fuzz harness (atom, jsonfeed too)
./bin/zig build -Doptimize=ReleaseSafe
```

If your default `zig` is already 0.15.2 you can skip the shim.
No package manager fetches at build time — every dependency is vendored
under `vendor/` and hash-pinned.

Outputs land in `zig-out/bin/{tributaryd,trib,fuzz-*}`.

---

## Run

```sh
# First run generates a 32-byte capability token at
# $XDG_DATA_HOME/tributary/token (or ~/.local/share/tributary/token),
# mode 0600. Print it once and never log it.
./zig-out/bin/tributaryd --db ./tributary.db
```

The daemon listens on:

- Unix domain socket — defaults to `$XDG_RUNTIME_DIR/tributaryd.sock`,
  mode `0600`. Owner-only. Use this for the CLI and scripts.
- Loopback TCP — bound to `127.0.0.1` on a kernel-assigned ephemeral
  port written to `<sock>.port` (mode `0600`). Use this for the web UI.

Both listeners require `Authorization: Bearer <token>` on protected
routes. `GET /v1/health`, `GET /` (the UI), and `GET /ui` are
intentionally unauthenticated — they expose nothing useful without a
token to make subsequent calls.

### CLI

```sh
trib health
trib add https://example.com/feed.xml
trib list
trib items --feed=1 --limit=20
trib items --q='cve OR vulnerability'
trib import subscriptions.opml
trib export > backup.opml
```

`trib` reads the token from the same file the daemon writes. Override
with `--token-file` and `--uds`.

### Web UI

Open the loopback URL in your browser:

```sh
echo "http://127.0.0.1:$(cat $XDG_RUNTIME_DIR/tributaryd.sock.port)"
```

Paste your token into the form to load feeds + items. The page never
phones home; the only network traffic is requests to the local daemon.

---

## What Tributary is *not*

(Listed because design discipline = scope discipline.)

- Not a Feedly clone with worse UX
- Not a social reader
- Not a SaaS — there is no Tributary cloud
- Not a discovery engine — bring your own subscription list
- Not a podcast or video app
- Not an LLM-summarizer-of-everything that summarizes things you
  didn't ask about
- Not a federation client

If the project drifts toward any of the above, the design has failed and
[`DESIGN.md`](DESIGN.md) is wrong, not the code.

---

## Privacy posture

- **No telemetry. Ever.** No analytics, no crash reporting, no update
  pings. Update checks are operator-initiated.
- **Subscription list never leaves the host** unless you explicitly opt
  into encrypted sync (v0.3+).
- **Item bodies retained 90 days by default**, configurable; tagged or
  annotated items are pinned indefinitely.
- **No request-level body logging.** Operational logs record
  `feed_id`, status code, byte count, parse outcome, duration. That is
  the entire log shape.

See [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md) for adversaries
considered and out-of-scope.

---

## License

- Daemon and reference clients: **AGPL-3.0**
- Specs, parser conformance suite, rule DSL grammar, and design
  documents: **CC BY-SA 4.0**
- "Tributary" reserved by Deep Fork Cyber.
