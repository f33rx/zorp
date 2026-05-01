# Reference material

Specs the zorp server has to honor so `curl -F 'zorp=@-' URL`
and friends keep working. Sourced from curl's own documentation
(`everything.curl.dev`, `curl.se/docs`) which states which RFCs each
piece of curl behavior follows.

## What `curl -F 'zorp=@-' https://host/` actually does

Two layers, two specs:

1. **Transport**: an HTTP POST. Governed by HTTP semantics + the
   negotiated wire version.
2. **Body**: a `multipart/form-data` payload with one part named
   `zorp` whose content is read from stdin (`@-`).

So the zorp server has to parse two things correctly: an HTTP
request, and a multipart body inside it.

## HTTP

curl's own protocol page
(<https://everything.curl.dev/protocols/curl.html>) explicitly cites:

| Spec     | RFC      | Notes                                          |
| -------- | -------- | ---------------------------------------------- |
| Semantics| RFC 9110 | Methods, status codes, headers, conneg. The contract that holds across all HTTP versions. |
| Caching  | RFC 9111 | Cache-Control, ETag, conditional requests. |
| HTTP/1.1 | RFC 9112 | On-the-wire syntax; replaces RFC 7230. |
| HTTP/2   | RFC 9113 | Binary framing; replaces RFC 7540. |
| HTTP/3   | RFC 9114 | HTTP over QUIC. |

The 9110-9114 family (June 2022) supersedes the older 7230-7235
family, but a server that implements 7230 today is still on the wire
correctly -- 9112 mostly tightens wording. We should target 9110/9112
since that is what curl itself targets.

For our scope (HTTP/1.1 over Caddy, possibly HTTP/2 once Caddy
upgrades the connection), the load-bearing reading is:

- RFC 9110: <https://www.rfc-editor.org/rfc/rfc9110>
- RFC 9112: <https://www.rfc-editor.org/rfc/rfc9112>

## multipart/form-data (the `-F` flag)

curl's `-F` produces a `multipart/form-data` body. The current spec
is **RFC 7578**, which obsoletes RFC 2388, which itself replaced the
original RFC 1867 (1995). curl's man page and httpscripting doc still
mention "RFC 1867 posting" for historical reasons, but the rules curl
follows now are 7578.

- RFC 7578: <https://www.rfc-editor.org/rfc/rfc7578>
  - Section 4.2: filename handling
  - Section 4.3: charset of field values
  - Section 4.4: ordering of parts (preserved)
  - Section 4.7: the `boundary` parameter

What this means concretely for our server:

- Read `Content-Type: multipart/form-data; boundary=...` from the
  request headers; reject if missing or malformed.
- Parse parts; pull the part with `Content-Disposition: form-data;
  name="zorp"` and treat its body as the paste content.
- Honor `Content-Type` on the part if present (so binary uploads keep
  their type); fall back to `text/plain; charset=utf-8`.
- Do not trust `filename=` blindly (RFC 7578 4.2 -- and curl issue
  #7789 documents real-world filename-escaping inconsistencies, so we
  should ignore filename for routing/storage decisions).

curl issue #7789 -- worth knowing the rough edges exist:
<https://github.com/curl/curl/issues/7789>

Daniel Stenberg's writeup on multipart inconsistencies:
<https://daniel.haxx.se/blog/2021/11/13/fun-multipart-form-data-inconsistencies/>

## URL syntax

curl parses URLs per **RFC 3986** ("RFC 3986+" in curl's own words --
they bend the rules slightly for browser compat, e.g. accepting
extra slashes). Browsers, by contrast, follow the WHATWG URL Living
Standard. For a server we are building, RFC 3986 is what matters --
that is what arrives in the request line.

- RFC 3986: <https://www.rfc-editor.org/rfc/rfc3986>
- curl's URL syntax page: <https://curl.se/docs/url-syntax.html>

Implication: keep paste IDs to URL-safe characters (we already chose
base32, which is fine: `[A-Z2-7]`).

## Other curl POST modes (for awareness)

We are committing to `-F` (multipart) as the canonical wire format,
but it is worth knowing the alternative so we do not accidentally
half-support it:

- `-d` / `--data` produces `application/x-www-form-urlencoded`. Not an
  IETF RFC; the encoding is defined in the HTML / WHATWG URL spec.
  curl's own writeup: <https://everything.curl.dev/http/post/postvspost.html>
- `--data-binary @-` produces a raw body with whatever content-type
  the user sets. Useful for uploading bytes verbatim. No form parsing
  involved.

For zorp, we should accept all three on the same `POST /`
endpoint:

1. `multipart/form-data` with a `zorp` field -- the canonical path.
2. `application/x-www-form-urlencoded` with a `zorp` key -- so
   `curl -d 'zorp=hello'` works.
3. Any other content type -- treat the whole body as the paste, store
   the content-type verbatim. So `curl --data-binary @file -H
   'Content-Type: image/png'` does the right thing.

That gives us a clean three-rule decision tree and keeps the contract
simple to document.

## Spec contract for the zorp server (summary)

What we are committing to honor:

- RFC 9110 (HTTP semantics) -- methods, status codes, headers
- RFC 9112 (HTTP/1.1 wire syntax) -- whatever Caddy speaks upstream
- RFC 7578 (multipart/form-data) -- `-F` body parsing
- RFC 3986 (URL syntax) -- request-line parsing, ID character set

Anything beyond that (HTTP/2, HTTP/3, caching) is delegated to Caddy
and the platform; the service tier does not need to implement them
itself.

## Sources

- curl protocol map: <https://everything.curl.dev/protocols/curl.html>
- curl man page: <https://curl.se/docs/manpage.html>
- curl httpscripting: <https://curl.se/docs/httpscripting.html>
- curl URL syntax: <https://curl.se/docs/url-syntax.html>
- curl multipart guide: <https://everything.curl.dev/http/post/multipart.html>
- curl `-d` vs `-F`: <https://everything.curl.dev/http/post/postvspost.html>
