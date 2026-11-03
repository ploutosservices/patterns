# One refresh per refused token, then an hour's wait

A home-automation service signs in to a dozen vendor clouds: thermostats, shades, a solar
gateway, a doorbell. Each cloud hands out an access token and a refresh token. Some vendors lock
the account after a burst of failed sign-ins; some rotate the refresh token on every use, so a
refresh sent twice can spend the only one there was. A client that answers every refusal with
another sign-in turns one expired token into a locked account. This write-up describes the bound
one such service puts on that loop.

## The rule

1. **A lost session refreshes past the cache once.** When the vendor refuses the cached access
   token, the client does not try it again: the cached token is the one refused. It sends one
   refresh and marks the token that refresh minted as a bypass.
2. **The mark clears on the first answer.** The first read the vendor answers with the marked
   token clears the mark. From then on it is an ordinary token, and its own later refusal earns
   one more refresh.
3. **A refusal of the marked token, or of a refresh, is red.** If the vendor refuses the token a
   bypass just minted, before any read was answered with it, or refuses the refresh itself, the
   reach turns red by name and an hour starts.
4. **Inside the hour nothing is sent.** Every request on the reach is refused locally, by name,
   with the time the hour ends. No sign-in, no refresh, no read.
5. **After the hour, one refresh.** When the hour runs out the client owes the vendor one
   refresh. A refusal starts another hour.
6. **The mark and the hour's start are written with the token, in one write.** A restart reads
   them back, so it neither spends a second refresh nor shortens the hour.
7. **Silence is not a refusal.** A timeout, a closed connection or a 5xx rotated nothing. It
   keeps the cause and is tried again on the next cycle, without starting an hour.

## What it buys

- **The worst case is a number.** A vendor sees at most two refreshes per refused token and then
  one per hour, however many readers, restarts or bugs sit above the client.
- **A rotating refresh token is not burnt twice.** One refresh per refusal means the token the
  vendor last issued is the one the client holds.
- **The red is legible.** The refusal carries the reach's name and the time the hour ends, so a
  person reading an alert knows what failed and when the client will try again.

## What it costs

- A real outage of the vendor's sign-in costs at least an hour of readings after it ends.
- A person who fixes the account by hand, for example by re-linking it in the vendor's app, waits
  out the hour unless the stored state is cleared.

## Worked example

The service's own code is the example. In `api/wire.py`, `token_plan()` decides between serving the
cached token, refreshing past it, and red; `token_gate()` refuses inside the hour;
`bounded_token()` runs one sign-in under the bound and writes the token, its mark and the hour's
start together; `TOKEN_WAIT_SECONDS` is the hour. In `api/act.py`, `_bearer_session()` clears the
mark at the first answered read. `tests-token-bound.py` drives each token kind through a fake
vendor on a stub clock and grades every step above.
