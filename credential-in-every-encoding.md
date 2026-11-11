# A credential substituted in every encoding

A service that talks to vendor clouds keeps two things that leak credentials if written
carelessly: its error messages, which go to logs and alerts, and its recordings of what each
vendor answered, which go into the test suite. Masking the password where it appears in plain
text is not enough. The same password appears percent-encoded in a URL, form-encoded in a sign-in
body, and inside tokens the vendor minted after the request was made. This write-up describes how
one such service makes every refusal and every recording safe to keep.

## The rule

1. **The client knows what it sent.** Every value the client puts on the wire that belongs to the
   account is registered with one object, `Secrets`, with a placeholder: `{password}`,
   `{access_token}`, `{refresh_token}`. This covers the configured credentials and every value a
   session mints (tokens, session ids), registered the moment it is learned.
2. **Every encoding is replaced.** A value is searched for as itself, percent-encoded and
   form-encoded, and each is replaced by the placeholder in the same encoding. A URL keeps
   working as a URL, and a body keeps its form shape.
3. **Longest first.** Values are replaced longest first, so a short value that happens to sit
   inside a long one does not split it.
4. **An encoding the scan cannot see is rebuilt.** A `Basic` header is base64 of `user:password`,
   so no scan of the header finds the password. The client keeps the parts it encoded, substitutes
   each, and encodes again, so the recorded header is base64 of `user:{client_secret}`.
5. **Short values are not registered.** A value under four characters is skipped; replacing it
   would rewrite ordinary text.
6. **One substitution for both outputs.** Error messages and recordings pass through the same
   object, so a value that is safe in one is safe in the other. A URL in an error also loses its
   query string, `HttpWire.safe()`.
7. **A recording is substituted when served, not when kept.** An answer is kept raw and
   substituted when it is read out, with every value known by then. A token the answer itself
   minted is learned after the answer arrived and is still replaced.

## What it buys

- **Errors can be posted.** A refusal can go to a chat channel or an alert with its URL and body,
  because nothing in it is a live credential.
- **Recordings can be committed.** A recorded vendor answer can sit in a public test suite.
- **No list of fields to maintain.** The rule follows values, not field names, so a vendor that
  echoes the password back in an unexpected field is still covered.

## What it costs

- Every request and every answer is scanned once per registered value, in three encodings.
- A value the client never sent is not known, so it is not replaced. A vendor that returns a
  secret of its own, such as another account's key, is not covered by this rule.

## Worked example

The service's own code is the example. In `api/wire.py`, `Secrets.add()` registers a value with its
placeholder and `Secrets.__call__()` substitutes; `_encodings()` lists the plain, percent- and
form-encoded forms, and `_encode_like()` writes the placeholder in the matching one. `Recorder` keeps
raw answers and `Recorder.served()` substitutes them when read. In `api/wire_http.py`,
`HttpWire._prepared_view()` rebuilds a `Basic` header from substituted parts.
`tests-recording-wire.py` grades a key in a path, a token minted after the request was sent, a
`Basic` header and a form body.
