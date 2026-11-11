# Twenty device libraries, read for one property: does a write go out once?

A home-automation service drives about twenty kinds of device: thermostats, mini-splits, shades, a pool
controller, a UPS, a solar gateway, TVs, appliances. For each one it has its own small "wire" module instead
of the open-source Python library that already exists. That is a cost, so in late 2026 every library was read
from its own source, to decide whether it could replace the hand-written code. This write-up is that review,
re-checked at each library's current HEAD, with every defect reported upstream.

## The test

A library is adopted when it does all three:

1. **It covers every read and every command the service uses.**
2. **It can be held under one session per device.** The service has one process per device, holding one
   session and one lock (see [one owner per device reach](one-owner-per-reach.md)). A library that opens its
   own connections, threads or sessions behind that process's back breaks the invariant.
3. **It removes more code than its adapter adds.**

Underneath these is one property that decides most cases: **a command is sent once, and the caller learns
the device's own verdict.** A read sent twice costs nothing. A toggle, a "next mode" or a relative setpoint
sent twice is a different command from the one the caller asked for. A 200 carrying an error, or a reply that
is dropped, tells the caller that something happened when it did not.

## What we found

Each finding below was reproduced at the library's HEAD without a device, using a fake socket, server or
transport, and then reported. A finding that turned out to be a design choice we simply dislike was not
reported, and it is listed as such.

| Library | Device | Finding at HEAD | Upstream |
|---|---|---|---|
| tinytuya | Tuya plugs and switches | A write is sent again after a reply timeout, and no setting stops it | [#753](https://github.com/jasonacox/tinytuya/issues/753), fixed by [#754](https://github.com/jasonacox/tinytuya/pull/754): `send_retry_limit=0` |
| aionut | NUT (UPS) | `run_command()` sends `INSTCMD` again when the reply times out | [#93](https://github.com/bdraco/aionut/issues/93) |
| screenlogicpy | Pentair pool controller | A command is sent again after a timeout, and the gateway then reconnects and repeats it: one colour change went out 4 times. `max_retries=0` is refused | [#74](https://github.com/dieselrabbit/screenlogicpy/issues/74) |
| pyoverkiz | Somfy TaHoma | `execute_action_group()` is sent up to 3 times on a timeout | [#2340](https://github.com/iMicknl/python-overkiz-api/issues/2340); a fix existed as PR #2286 |
| samsungtvws | Samsung TV | REST calls ignore the HTTP status: a 404 with a JSON body is returned as a result | [#200](https://github.com/xchwarze/samsung-tv-ws-api/issues/200) |
| pysmartthings | SmartThings | A 400, 405 or 410 to a command returns `None`, which looks like success | [#861](https://github.com/pySmartThings/pysmartthings/issues/861) |
| aiopvapi | PowerView shades | A move the hub refuses with HTTP 200 and `err: 209` is reported as done, and the cache takes the target | [#63](https://github.com/sander76/aio-powerview-api/issues/63) |
| greeclimate | GREE mini-split | The command result's `r` code is dropped, and the requested values become the cached state | [#258](https://github.com/cmroche/greeclimate/issues/258) |
| nexia | Trane thermostats | aiohttp follows the 302 itself, so the re-login code never runs. An expired session's PUT body is sent to the login page and the caller sees 200 | [#237](https://github.com/hvaclibs/nexia/issues/237) |
| pyenphase | Enphase gateway | Token retries on a connection error have no stop, so they retry forever | [#543](https://github.com/pyenphase/pyenphase/issues/543) |
| pylutron | Lutron RadioRA 2 | After one lost reply, that output is never queried again, and the cached level is returned | already open as [PR #133](https://github.com/thecynic/pylutron/pull/133) |
| pywattbox | WattBox | The sync driver compares its transport object with the string `"telnet"` | already reported upstream (PRs #28, #30) |

Read and found to be design choices, not defects. None was reported:

- **aiohue** retries 25 times, but only on 429 and 503, where the bridge says it did not take the request.
- **aiosomecomfort** has a lockout (three failed logins, then ten minutes refused locally). Our first reading
  said it had none, and that was wrong. Re-login is left to the caller.
- **ring_doorbell** refreshes tokens itself and hands each new one to the caller's callback.
- **gehomesdk** reads and writes only over its websocket.
- **pymiele** returns the response, so the caller reads the body. Token refresh is the caller's job.
- **pyblu** parses a chosen subset of the status fields.
- **screenlogicpy**'s fixed 2-second timeout is a missing setting, not wrong behaviour.
- **tesla-wall-connector** handles HTTP status correctly. Its repair of a body missing its closing `}` would
  also accept a body cut off mid-number, but we have never seen a device do that, so it was not reported.

Our first reading also claimed that nexia "can send a write twice". At HEAD that is false: the re-send code is
unreachable, which is the defect reported instead.

## The pattern

Four of the twenty libraries re-send a write after a timeout. Four more turn a refusal into a success: an
ignored HTTP status, an ignored error field in the body, or an ignored result code. Both come from the same
habit. **A timeout is treated as "the request did not arrive", when it only means "the reply did not
arrive".** Retrying is right for a read and wrong for a write, and a library that does not know which
is which gets writes wrong.

What the service's own wire modules do instead:

- **Retries are allowed for connecting and for reads only.** A write is sent once. A lost reply is reported
  as "sent, verdict unknown", and the caller decides whether to read the state back.
- **The vendor's verdict is the result.** An HTTP status, an `err` field and an `r` code each count as an
  answer. A 200 is not taken as success when the body says otherwise.
- **Every retry loop has a stop**, and a refused credential gets the bound described in
  [one refresh per refused token](one-refresh-then-an-hour.md).

## What it costs

Twenty hand-written wire modules, about 5,500 lines, which the service maintains instead of the community.
Each library is read again whenever a release changes one of the findings above. When tinytuya's
`send_retry_limit=0` ships, it is the first library that passes the test.
