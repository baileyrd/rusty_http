# gap-analysis.md

Parity-loop run: `rusty_http` (this repo) vs. the [`http`](https://docs.rs/http/1.4.2)
crate, pinned at **v1.4.2** (latest on crates.io as of this run,
2026-07-23). Diffed via `cargo public-api` (rusty_http: `cargo +nightly
public-api --all-features -ss`; `http`: same, run against the crate's own
`Cargo.toml` in the local registry cache) — symbol-name matching, then
judgment applied per module below.

`http` is the de facto standard low-level HTTP types crate (used by hyper,
reqwest, actix-web); rusty_http's `method`/`status`/`version`/`header`/`head`
modules cover the same conceptual ground, which makes it the natural
reference surface for this crate's own mission (a sans-IO HTTP/1.1 message
layer), as opposed to a general framework crate.

## Deliberately excluded from the candidate list (scope, not oversight)

- **`http::Version::HTTP_2` / `HTTP_3` / `HTTP_09`** — rusty_http's `Version`
  is intentionally HTTP/1.0 and HTTP/1.1 only (see `ARCHITECTURE.md`
  non-goals: HTTP/2 is out of scope; HTTP/0.9 was never in scope).
- **`http::request::Request<T>` / `Response<T>` / `Builder` / `Extensions`
  / `HeaderName` / `HeaderValue`** — rusty_http's `RequestHead`/`ResponseHead`
  are plain structs with public fields (`method`, `target`/`status`,
  `headers`, `version`), not a body-generic `Request<T>` wrapper with a
  builder API and a typed extension bag. That's a deliberate, existing
  design choice (sans-IO core has no body-generic type; body framing is
  handled separately via `body::Framing`/`BodyReader`), and `Extensions` in
  particular reads as "routing framework" territory, an explicit non-goal.
  Not treated as a gap.
- **`http::status::StatusCode::from_u16` range validation** — rusty_http's
  `from_u16` deliberately never validates (doc comment: "a peer can send
  anything in a status line, and rejecting it is the head parser's call,
  not this type's"). Existing, documented design decision, not a gap.
- **`http::Uri` / `http::uri::*`** — out of scope for this run; rusty_http's
  `Url` type is a different (WHATWG-style) concept and wasn't the reference
  surface chosen for this run (see run scoping: `http` crate types, not
  `url` crate).
- **`http::Error`/`http::Result`, `Method`/`StatusCode` `Default` impls,
  `PartialEq<&str>`/`PartialEq<str>` for `Method`** — noise: internal error
  type shape and marginal ergonomic trait impls with no evidenced call-site
  need in this crate's actual consumers (`rusty_request`, `rusty_tail`).

## Candidate gaps

| Symbol | Category | Platforms | Reference | Breaking? | Est. size | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| Named `StatusCode` constants (`OK`, `NOT_FOUND`, `INTERNAL_SERVER_ERROR`, ...) | const | both | `http::status::StatusCode::{OK,NOT_FOUND,...}` | no | S | rusty_http's `StatusCode` has zero named constants today — every call site spells out `StatusCode::from_u16(404)`. Pure additions (new `impl` consts), same table `canonical_reason` below needs. |
| `StatusCode::canonical_reason()` | fn | both | `http::status::StatusCode::canonical_reason` | no | S | Bundle with the constants above — same reason-phrase table backs both. |
| `Method::is_safe()` / `Method::is_idempotent()` | fn | both | `http::method::Method::{is_safe,is_idempotent}` | no | S | RFC 7231 §4.2 classification. Pure additions on the existing enum; `Extension` variant should return `false`/`false` (unknown method, not provably safe/idempotent). |
| `HeaderMap::get_all()` | fn | both | `http::header::HeaderMap::get_all` (simplified: no `HeaderName`/`GetAll` wrapper types here, just `impl Iterator<Item = &str>`) | no | S | rusty_http's `HeaderMap` is already a multi-map internally (`append` keeps repeats; `get` only returns the first). No way today to read all `Set-Cookie` values from a `HeaderMap` directly — `cookie.rs`'s `store_from_response` takes a pre-extracted iterator of values instead, which is exactly the friction this closes. |
| `HeaderMap` sensitive-value marking + `Debug` redaction | fn | both | `http::HeaderValue::{is_sensitive,set_sensitive}` / `HeaderMap`'s `Debug` masking | no | M | No existing header is ever marked sensitive today, so no current `Debug` output changes — purely additive (`insert_sensitive`/`append_sensitive` plus an internal per-entry flag). Worth doing given the crate's own stated security posture (`ARCHITECTURE.md`'s "shared attack surface" framing); `Authorization`/`Cookie` values otherwise appear in full in any `{:?}` of a parsed head. |

Five candidates, all purely additive (no `Breaking? yes` rows this round —
nothing here requires changing an existing public signature or adding a new
dependency). Reporting before filing issues, per the loop's checkpoint.
