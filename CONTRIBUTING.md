# Contributing

**Bug reports, questions and deployment experiences are welcome now. Code
contributions cannot be accepted yet.**

---

## Why code cannot be accepted yet

No license has been chosen. See [LICENSE.md](LICENSE.md).

Accepting a patch into software with no license would leave both sides in an
unclear position about what had been given and what could be done with it.
Sorting that out afterwards is far harder than waiting, so the answer today is
that pull requests will not be merged.

This will change. When the license question is settled, this page will say what
the process is.

## What is genuinely useful right now

**Tell us when something is wrong.** Open an issue. The most valuable reports are
the ones that name what you did, what you expected, and what happened instead.

**Tell us when the documentation is wrong.** This is worth as much as a bug
report and it is easier to act on. Documentation that describes something the
software does not do is a defect, and it is treated as one here. If an
instruction on any of these pages did not work, that is a report worth making.

**Tell us about your environment.** PathLMS is meant to run anywhere Docker
Compose runs, and the only way anybody finds out where that is not true is when
somebody says so. A note saying which provider, which instance type and what you
had to do differently is genuinely useful.

**Tell us where a limit caught you out.** The README carries a list of what
PathLMS does not do. If you were misled by it, or if it is missing a limit you
ran into, say so. That list works only while it is honest.

## Reporting something well

Please include:

- **The version.** It is on the Settings screen and at the top of the README.
- **Where it is running.** Cloud instance, virtual machine, bare metal, home
  appliance, and roughly what the machine is.
- **What sits in front of it**, if anything, and whether traffic is encrypted.
- **What the log said.** `docker compose logs api` is where PathLMS explains
  itself, and the **first** refusal is usually the real one. The lines after it
  are consequences.

**Remove anything private before you paste.** Logs can carry email addresses and
network addresses. Never paste anything out of your `.env`: every value in that
file is a credential for your installation.

## Security problems

Do not open a public issue for a security problem. See [SECURITY.md](SECURITY.md).

## Code of conduct

Be decent to people.
