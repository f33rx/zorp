# zorp -- project plan

A self-hosted public pastebin. Single Go binary, embedded SQLite,
server-side encryption at rest, 7-day default TTL, syntax highlighting
on demand. The HTTP API is the entire UX -- `curl`, the `zorp` CLI,
and a browser all use the same URLs.

## Goals

- `cmd | zorp` (or `curl -F 'zorp=<-' URL`) returns a paste URL on stdout.
- Open the URL with `?lang` to see a syntax-highlighted view with line
  numbers (`?py`, `?go`, etc.).
- Pastes auto-expire after 7 days unless told otherwise.
- Bodies are encrypted at rest.
- Single host, two-process deploy (Caddy + zorp).

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

- **Edge (Caddy)**: TLS via Let's Encrypt, per-IP rate limiting, body
  size cap. No auth.
- **Service (zorp)**: one Go binary. Owns `pastes.db` (SQLite, WAL).
  Handles POST/GET, server-side rendering with chroma, periodic TTL
  sweep.

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

- **Language**: Go. Pure-Go SQLite (`modernc.org/sqlite`), `chroma` for
  highlighting, `golang.org/x/crypto/chacha20poly1305` for AEAD.
- **Storage**: embedded SQLite, WAL mode. Bodies stored as ciphertext.
- **Encryption**: server-side only. Single master key, XChaCha20-Poly1305
  per paste with a random 24-byte nonce. No client-side / fragment-key
  scheme. TLS at the edge handles transport.
- **Auth**: none. Open POST, rate-limited per IP at the edge.
- **Audience**: open / multi-user. No accounts; anyone with a URL reads.
- **TTL**: optional, default **7 days**. `?ttl=<duration>` on POST
  overrides. Absent = uses default; explicit `?ttl=0` could mean
  permanent (TBD in Phase 2).
- **Rendering**: server-side HTML, no JS framework, no build step.
- **Wire format**: `zorp=` POST field + plain-text URL response.
- **Backup**: deferred. `cp pastes.db` is the v1 strategy.

## Still open

1. **Domain.** Short hostname; the URL is the UX.
2. **Master key provisioning.** Disk file vs env var vs 1Password
   agent. File on a separate volume from the DB is the simplest answer.
3. **TTL semantics at the edges.** Is `?ttl=0` "permanent" or "burn
   after read"? Is the max TTL configurable or hard-coded? Decide
   during Phase 2.
4. **Help page content.** What does `GET /` actually print?
   Plain-text usage with the canonical `curl -F 'zorp=<-' URL`
   example. Quick decision in Phase 1.

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
5. **Polish.** Vim plugin, zorp `--clipboard` polish, whatever else.

## Risks

- **Open POST endpoint.** A public open paste service eventually
  attracts warez, spam, illegal content. Per-IP rate limits + 7d TTL
  + body cap reduce blast radius; be ready to add an API key or
  basic-auth if abuse gets bad.
- **Master key loss = data loss.** Lost key = unrecoverable noise.
  Back it up to two physically separate locations; never store it
  on the same volume as the DB.
- **SQLite single writer.** Fine for personal scale. Migrate to
  Postgres if it ever bites.
- **Wire-format lock-in.** `zorp=` POST field, plaintext URL
  response, `?lang` for highlighting. Treat as a stable contract.

## Out of scope (anchor)

- Search, analytics, accounts, edits, comments
- Markdown rendering, file/image hosting
- Real-time, mobile, desktop
- Multi-host distribution
