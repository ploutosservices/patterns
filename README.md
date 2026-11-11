# patterns

Methods from a home-automation service that runs two houses, written up for outside review.
Each write-up states its rule, what the rule buys and what it costs. The service's own code is in a
private repository; the file and function names in each "Worked example" section are there so a
reader can see the method is practised, not proposed.

- [One owner per device reach: development reads, only production writes](one-owner-per-reach.md)
  — every device is reached by one process holding one session, and the write gate is ownership,
  not an environment name.
- [A device register composed from the routers' leases](register-from-leases.md) — the DHCP lease
  tables are the population, and a MAC on the wire with no register row is a finding.
- [One refresh per refused token, then an hour's wait](one-refresh-then-an-hour.md) — a refused
  vendor token earns one refresh; a second refusal turns the reach red and nothing is sent for an
  hour.
- [A credential substituted in every encoding](credential-in-every-encoding.md) — every value the
  client sent is replaced by a placeholder, plain, percent- and form-encoded, in every error and
  every recording.

Criticism is the point. Open an issue: https://github.com/ploutosservices/patterns/issues
