# Recorded vendor answers, replayed through the real driver

A home-automation service parses answers from about twenty vendors: XML from a music player, JSON from a
thermostat cloud, encrypted UDP from a mini-split. Its tests used to hand each parser a dict that somebody
typed, so what they checked was the developer's idea of what a vendor says. The answers that actually caused
trouble were never like that: a shade whose battery reads null because the gateway cannot hear it, or a
panel in a state the code had no branch for. Those can only be seen where the vendor is. This write-up
describes how the service captures them there and replays them everywhere else.

## The rule

1. **Production records.** The process that owns a device keeps the latest raw answer per *recording key*,
   with the time it arrived. The key is the request with its volatile parts replaced: the method, the path,
   the query with nonces as placeholders, and the headers minus credentials. That production process is the
   only one that talks to vendors (see [one owner per device reach](one-owner-per-reach.md)), so it is the
   only place a real answer exists.
2. **Every credential is substituted before an answer is kept.** Any value the client sent, or that the
   vendor handed back as a credential, is replaced by a named placeholder. A cookie is kept as its shape:
   the name, with its value replaced. The rule is the one in
   [a credential substituted in every encoding](credential-in-every-encoding.md).
3. **A developer takes the recording.** One command, run from a development machine, reads two halves
   from production's front door:
   - the **vendor half**, every raw answer, through a door that admits only the owning developer's
     certificate;
   - the **device-layer half**, the composed state of every device and each route's own reading.

   Both halves are written into the repository with the time they were taken.
4. **The suite replays them through the real code.** A test stands the recording up as a local HTTP door
   and points the service's real read entrypoints at it. It also watches the function that would call the
   vendor and fails if anything calls it. The parsers run on the vendor's real bytes, and no account or
   network is involved.
5. **The recording is graded.** Every run checks three things:
   - no credential is left in either half: no JWT anywhere, and no long value under a key whose name
     says token, secret, session or cookie;
   - each route serves the fields its driver claims to carry, and no field declared non-nullable is null;
   - the recording is not older than its declared maximum age.
6. **An old recording fails the suite.** After 30 days the suite stays red until someone retakes the
   recording. A vendor that changed its answers is found within a month, by the suite rather than by
   production.

## What it buys

- **Tests grade the vendor's real answers, not ours.** A parser that handles every answer we imagined, but
  not the one the vendor actually sends, fails here first.
- **A library upgrade is checked against real bytes.** If a new version parses a recorded answer
  differently, the replay shows it.
- **The recording audits the credential handling.** The first recording found two credentials that the
  substitution had missed: a solar gateway's cloud login returned a second, signed token beside the session
  id, and an appliance cloud's token exchange returned a refresh token. Nothing in the client used either
  value, so nothing had ever registered them as secrets. The same recordings found 24 fields across six drivers that
  either did not match their own declarations or did not arrive. In some, the driver claimed a field its
  read never returned, as with a music player's model. In others, a field declared never-null arrived null,
  as with an EV charger's serial. Each one is either a defect in the driver or a declaration that promises
  more than the device gives, and the grade named each by its path in the recording. No test with typed
  input had caught any of them.

## What it costs

- A developer has to retake the recording every month, and each take reads production once per device.
- The recording holds what the house's devices answered at that moment: room names, firmware versions,
  network addresses. It stays in the service's private repository. This write-up is public; the recording
  is not.
- Only the latest answer per key is kept. An answer production saw once and never again is lost unless a
  recording happens to catch it.

## Worked example

The service's own code is the example. Its `record <front> <identity> <tree>` command takes both halves.
`api/owner.py`'s `recording()` serves the vendor half, and `api/wire_http.py`'s `record_answer()` keeps each
answer, with a cookie reduced to its shape. `tests-fixtures/recordings.py` holds the graders:
`fixture_problems()` for credentials, a per-route check of claimed fields and nullability, and the age
check. `tests-recorded-reads.py` stands a recording up as the door and fails if the vendor function is
called.
