# zorp -- project plan

A self-hosted public pastebin. Single [Go](https://go.dev/) binary,
embedded [SQLite](https://www.sqlite.org/), server-side encryption at
rest, 7-day default TTL, syntax highlighting on demand. The HTTP API
is the entire UX -- [`curl`](https://curl.se/), the `zorp` CLI, and a
browser all use the same URLs.

## Goals

- `cmd | zorp` (or `curl -F 'zorp=<-' URL`) returns a paste URL on stdout.
- Open the URL with `?lang` to see a syntax-highlighted view with line
  numbers (`?py`, `?go`, etc.).
- Pastes auto-expire after 7 days unless told otherwise.
- Bodies are encrypted at rest.
- Single host, two-process deploy ([Caddy](https://caddyserver.com/) + zorp).

## Non-goals

- User accounts, logins, per-user paste lists.
- Editing, versioning, comments, search.
- Markdown rendering, image hosting, file uploads beyond text.
- Real-time collab or mobile/desktop apps.
- Multi-host scale-out.

## Architecture

```
   zorp / curl / browser
            |
            v
       +---------+        +----------------------+
       |  caddy  | -----> | zorp (Go + SQLite)   |
       |  TLS    |  HTTP  | API + render +       |
       |  rlim   |        | sweeper              |
       |  body$  |        +----------------------+
       +---------+
```

- **Edge (Caddy)**: TLS via [Let's Encrypt](https://letsencrypt.org/),
  per-IP rate limiting, body size cap. No auth.
- **Service (zorp)**: one Go binary. Owns `pastes.db` (SQLite,
  [WAL](https://www.sqlite.org/wal.html)). Handles POST/GET,
  server-side rendering with [chroma](https://github.com/alecthomas/chroma),
  periodic TTL sweep.

Two processes total. `pastes.db` is on a host bind-mount; the master
encryption key file lives on a separate volume so disk-level
compromise of the data dir alone does not yield plaintext.

## Wire contract

| route                | behavior                                                                      |
| -------------------- | ----------------------------------------------------------------------------- |
| `GET /`              | Plain-text help page -- the docs are the homepage.                             |
| `POST /`             | Form field `zorp=<content>`. Returns 200 with the paste URL on one line.       |
| `POST /?ttl=<dur>`   | Same, with explicit TTL override (e.g. `?ttl=1h`, `?ttl=30d`).                 |
| `GET /{id}`          | Raw paste body, original Content-Type.                                         |
| `GET /{id}?<lang>`   | HTML render with chroma syntax highlighting and line numbers.                  |
| `GET /{id}#n-42`     | Browser scrolls to line 42 in the rendered view.                               |
| 410                  | Returned past expiry; the row is also async-deleted.                           |
| 404                  | Unknown id.                                                                    |

POST also accepts `application/x-www-form-urlencoded` (`-d 'zorp=...'`)
and `--data-binary` with any content type, per `REFERENCES.md`. The
canonical form is the multipart `-F 'zorp=<-'` flavor that the CLI
already speaks.

## Decisions locked

- **Language**: Go. Pure-Go SQLite via
  [`modernc.org/sqlite`](https://pkg.go.dev/modernc.org/sqlite),
  [`chroma`](https://github.com/alecthomas/chroma) for highlighting,
  [`golang.org/x/crypto/chacha20poly1305`](https://pkg.go.dev/golang.org/x/crypto/chacha20poly1305)
  for AEAD.
- **Storage**: embedded SQLite, WAL mode. Bodies stored as ciphertext.
- **Encryption**: server-side only. Single master key
  (`ZORP_KEY` env var; operators may inject from a secret store like
  [1Password](https://1password.com/) before launch). XChaCha20-Poly1305
  per paste with a random 24-byte nonce. No client-side / fragment-key
  scheme. TLS at the edge handles transport.
- **Auth**: none. Open POST, rate-limited per IP at the edge.
- **Audience**: open / multi-user. No accounts; anyone with a URL reads.
- **TTL**: default **7 days**. `?ttl=<duration>` on POST overrides,
  bounded by a server-configured max (above the default). Min 60s.
- **Rendering**: server-side HTML, no JS framework, no build step.
- **Wire format**: `zorp=` POST field + plain-text URL response.
- **Help page**: `GET /` returns a plain-text usage guide -- the
  canonical `curl -F 'zorp=<-' URL` example front and center.
- **Domain**: `zorp.shornyak.com`.
- **Backup**: deferred. `cp pastes.db` is the v1 strategy.

## Still open

1. **Content-Type at upload.** Server can't infer language from
   `multipart/form-data`. Options: always store as
   `text/plain; charset=utf-8` and let `?lang` pick the lexer at
   render time, accept a `?ct=` query, or sniff. Decide before
   Phase 1.
2. **Body size cap.** Pick a number (e.g. 64 KiB / 1 MiB / 5 MiB).
3. **Rate-limit policy.** Per-IP per-minute and per-day thresholds
   for Caddy.
4. **Logging policy.** What the service logs. IPs and ids fine;
   body bytes never.

## Phases

0. **Decide** remaining open items above.
1. **Walking skeleton.** SQLite-backed POST/GET, plain text response,
   no encryption yet, no rendering. The CLI works end-to-end.
2. **Storage hardening.** Encrypt body BLOBs with the master key,
   TTL sweeper, read-path TTL enforcement.
3. **Rendering.** Chroma highlighting, single HTML template,
   light/dark CSS, line anchors.
4. **Edge + deploy.** Caddy with TLS, rate limit, body cap. Manual
   backup procedure documented.
5. **Polish.** [Vim](https://www.vim.org/) plugin, zorp `--clipboard`
   polish, whatever else.

## Risks

- **Open POST endpoint.** A public open paste service eventually
  attracts warez, spam, illegal content. Per-IP rate limits + 7d TTL
  + body cap reduce blast radius; be ready to add an API key or
  basic-auth if abuse gets bad.
- **Master key loss = data loss.** Lost key = unrecoverable noise.
  Back it up to two physically separate locations; never store it
  on the same volume as the DB.
- **SQLite single writer.** Fine for personal scale. Migrate to
  [Postgres](https://www.postgresql.org/) if it ever bites.
- **Wire-format lock-in.** `zorp=` POST field, plaintext URL
  response, `?lang` for highlighting. Treat as a stable contract.

## Out of scope (anchor)

- Search, analytics, accounts, edits, comments
- Markdown rendering, file/image hosting
- Real-time, mobile, desktop
- Multi-host distribution
