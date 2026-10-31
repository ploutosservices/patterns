# One owner per device reach: development reads, only production writes

A home-automation service talks to a few dozen appliances: thermostats, shades, lights, a pool
controller, a car charger. Most of them have one weak network stack and some sit behind vendor
clouds that lock an account after a burst of sign-ins. There is one of each appliance, so there is
no test house. This write-up describes how one such service keeps a development copy useful (it
can read every device) and harmless (it cannot command one), and how the same rule stops two
copies from fighting over a device.

## The rule

1. **A reach is the unit.** A reach is one path to one or more devices: a gateway on the LAN, a
   cloud account, a single device's own API. Every device route names its reach.
2. **Every reach has exactly one owner process.** The owner holds the reach's one session, one
   lock, one loop thread and one bounded queue. Every exchange with the device, a read or a
   command, happens on that session under that lock, one at a time.
3. **Ownership is declared, never inferred.** The process is handed its owned set as
   configuration at start (a JSON list) and never reads it again. It does not infer ownership from
   which credentials happen to be present, and it never tests an environment name such as
   `production`.
4. **The write gate is ownership.** One function decides whether a write may go on the wire:
   does this process own the reach the route uses? Every command path, and every gate inside a
   driver, calls it before writing.
5. **Everyone reads; only the owner writes.** A process that owns no reach can still read: it
   asks the owner over HTTP (`state(key)`), and the owner answers from what its loop holds. A
   person can also ask for a fresh read from any copy. Nothing a non-owner does reaches the
   device's session directly.

## What it buys

- **A development copy is safe by construction.** Development is handed no reaches, so it has
  no device credential to withhold and nothing to sign in with. A mistake in development code
  cannot command a thermostat, and no rule has to remember to check which environment it is in.
- **One session per device.** A vendor that locks an account after repeated sign-ins sees one
  client, not one per developer. A gateway that answers slowly under load sees one poller, at a
  declared pace, whatever the number of readers.
- **The load on a device is a number.** The reach's queue depth is its device count plus one;
  past it, a request is refused as `busy` by name instead of waiting forever. Reads are served
  from the loop's held reading, each field with its own age, so a hundred readers cost the device
  nothing extra.
- **The question "who sent that?" has one answer.** Every accepted command is one ledger row
  written by the owner.

## What it costs

- A reader in development sees the owner's last reading, not the device's answer of this
  instant, unless it asks for a fresh read, which still goes through the owner's queue.
- The owner process is a single point of failure for its reaches. A restart costs one sign-in
  per reach and one loop cycle before reads are fresh again.
- Synchronous drivers need a thread per reach; an asyncio vendor library is driven through its
  own event loop on that thread.

## Worked example

The service's own code is the example. In `api/ownership.py`, `select()` is the gate; in
`api/reaches.py`, the per-reach session, lock, loop and queue; in `api/front.py`, the HTTP reader a
non-owner uses. `tests-write-walk.py` walks every write path and grades that it reaches
`select()`.
