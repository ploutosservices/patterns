# A device register composed from the routers' leases

A house network carries appliances, phones, guests' laptops and the odd device nobody remembers
buying. A hand-kept list of "our devices" is right on the day it is typed. This write-up describes
a register that starts from what the network itself says is there, and treats the difference
between the two as the thing to look at.

## The rule

1. **The DHCP lease tables are the population.** Every device that took an address from a
   router is in a lease, with its MAC and the address it was given. That is the only first-party
   account of what is on the wire.
2. **Rows are keyed by MAC.** A register row for a network device is `mac:<address>`; its IP is
   never stored as identity, because a lease can move.
3. **Everything else is an overlay on the lease.** Vendor discovery (a gateway's list of its
   members, a cloud's device list) adds the model, the serial and the routes. A person's
   correction (a name, a room, a note) is a separate, declared overlay with a reason and a ticket.
   Neither overlay can invent a network device that no lease has seen.
4. **The address is composed at read time.** A row's address is the lease at the device's own
   site first, then any lease, then the last address discovery saw, and the row says which
   (`ip_kind`).
5. **A reconciler compares both directions, every day:**
   - a lease whose MAC is in no register row is **unregistered**, and that is a finding;
   - a register row whose MAC no router has a lease for is **register-absent**, reported but not
     an alarm, because a seasonal device is not a defect;
   - a randomised MAC (the locally-administered bit set) is a phone or a guest, counted and never
     alarmed.

## What it buys

- **A device nobody wrote down is found within a day.** A gateway that sat on the LAN unnamed for
  four days, while every check said `ok` because none of them looked at the population, is the
  case that started this.
- **The register cannot go quietly stale.** The absent half shows how much of the book the wire
  no longer confirms.
- **Corrections survive.** A person's name for a device is an overlay with its own reason, so a
  rediscovery does not overwrite it, and a whole-row rewrite cannot erase it.

## What it costs

- A device with a static address set on the device itself takes no lease and is invisible to this
  rule.
- The reconciler needs read-only access to every router.
- Finding an unregistered device is not identifying it; a person still has to say what it is, or
  that it should be forgotten.

## Worked example

The service's register code is in `api/register.py` (`_lease_rows()` and the address composition);
the overlays are the declared corrections in `api/register_publish.py`. The daily reconciler is a
separate script owned by the network's operator; it reads the routers' lease tables over their
REST API with a read-only login and the register over HTTP, and exits non-zero only when an
unregistered device is present or a table could not be read.
