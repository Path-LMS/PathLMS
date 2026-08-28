# How updates work

Nothing deploys itself. An update happens when you decide it does, and it is
three image addresses in one file.

---

## What a new version actually is

Every release publishes three container images and attaches the files you install
it with to a release page. **Upgrading is changing the three image addresses at
the top of your `.env` and restarting.** Nothing else in your settings moves,
your data directory is untouched, and there is no build step.

Each address is pinned to an exact cryptographic fingerprint rather than to a
tag, so the version you install is the version that was tested, and it stays that
way. A tag can be repointed at different bytes after you have read it. A
fingerprint cannot.

## Being told one exists

You do not have to watch the release page. Every few hours the deployment asks a
public container registry which versions have been published, and tells you on
the administrator home screen and in **Settings**, on the **General** tab, in the
section called **Updates**.

What leaves your machine is a request for a public list of version numbers. No
version, no address, no identifier, and nothing about this installation or the
people using it.

It fails silently and it fails closed, which are two separate promises. **Closed**
means anything unexpected reports that the check could not run, never that you are
up to date, because a deployment three versions behind that cannot reach the
internet must not be told it is current. **Silently** means a deployment that is
isolated on purpose sees one calm line on a page somebody chose to open, and no
warning anywhere else.

Switch it off with `PATHLMS_UPDATE_CHECK=off` in your settings file. If you pull
PathLMS from a mirror inside your own network, point
`PATHLMS_UPDATE_CHECK_IMAGE` at that mirror instead of switching the check off.

## What the Updates section does

Open **Settings**, then **Updates** on the **General** tab. It shows:

- **The version you are running**, and whether a newer one has been published.
- **Six safety checks**, always all six and always in the same order, whether
  they passed or not: that no upgrade is already running, that the database
  answers, that the cache answers, that the object store answers, that there is
  free disk space, and that the backup destination can be reached. A screen that
  shows only what is wrong teaches nobody what was looked at, and the moment that
  matters most is the moment something has gone wrong and you are deciding
  whether to trust the thing that told you.
- **A record of every step**, kept where a restart cannot erase it, so you can
  see afterwards what happened and in what order.

Starting one asks for your password, because an update outlives the session you
started it in.

**It backs up before it changes anything.** Nothing irreversible happens until
the checks have passed and the backups exist.

## The one step PathLMS will not take, and why

**PathLMS does not replace its own containers, and it is built so that it
cannot.**

This is a design decision rather than something unfinished. Creating a container
accepts arbitrary bind mounts, so the right to create a container on a machine is
the right to be root on that machine. Handing that authority to the process
serving the internet would undo every bit of hardening in this stack. A
compromised application container gains nothing from any of the work above,
because none of it includes the power to start something new.

So the final swap is yours. Being told the last step is yours means you can plan
for it. Finding out afterwards would mean you could not.

## Doing the swap

On the server, in the directory holding `docker-compose.yml` and `.env`:

1. **Take note of your current image addresses.** Copy the three lines somewhere
   before you change them. This is your way back.
2. **Open the [release page](https://github.com/path-lms/pathlms/releases/latest)
   for the version you want** and copy the three addresses it reports.
3. **Replace the three lines in `.env`:**

       PATHLMS_API_IMAGE=...
       PATHLMS_WEB_IMAGE=...
       PATHLMS_DB_IMAGE=...

4. **Start it:**

       docker compose up -d

   Compose pulls whatever is new, recreates only what changed, and leaves the
   rest alone.

5. **Watch it come up:**

       docker compose logs -f api

   Any migration the new version needs runs on its own at startup, in a
   transaction, recorded so it cannot run twice.

**Move all three addresses together, every time.** They are published from one
job under one version, and mixing them is not something anybody tests. The
database image in particular is not just a schema: it also carries the backup and
restore scripts your deployment runs, so leaving it behind leaves your backup
code behind with it.

## Going back

Put the three previous addresses back in `.env` and run `docker compose up -d`
again. That is the whole of it for the software.

**The data is the part that does not simply revert.** A newer version may have
changed the shape of the database, and going back to older software over a newer
database is not something to attempt hopefully. That is what the backup taken
before the update is for: restore it, and you are back where you were, having
lost only whatever happened in between.

Which is the real argument for testing a restore before you need one.

## Version numbers, and what they promise

PathLMS follows Semantic Versioning. While the version begins with `0.`, the
specification itself says that anything may change at any time, so a change in
the middle number can carry a break. **Read the release notes before you move,
and take the backup either way.**
