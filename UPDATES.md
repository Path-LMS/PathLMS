# How updates work

Nothing deploys itself. An update happens when you decide it does, it takes three
edited lines and one command, and your data stays where it is.

---

## What moving to a new version costs you

Three lines and a restart. Your settings do not move, your data directory is
untouched, and there is no build step on the server.

Every release publishes the container images PathLMS is made of and attaches the
files you install it with to a release page. **Upgrading is changing the three
image addresses at the top of your `.env` and starting the stack again.**

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

**If the check cannot run, it says so rather than saying you are up to date.** A
deployment three versions behind that cannot reach the internet must never be
told it is current. And a deployment isolated on purpose sees one line on a page
somebody chose to open, with no warning anywhere else.

Switch it off with `PATHLMS_UPDATE_CHECK=off` in your settings file. If you pull
PathLMS from a mirror inside your own network, point
`PATHLMS_UPDATE_CHECK_IMAGE` at that mirror instead of switching the check off.

## What the Updates section does

It answers "is this safe to do right now". Open **Settings**, then **Updates** on
the **General** tab. It shows:

- **The version you are running**, and whether a newer one has been published.
- **Six safety checks**, always all six and always in the same order, whether
  they passed or not: that no upgrade is already running, that the database
  answers, that the cache answers, that the object store answers, that there is
  free disk space, and that the backup destination can be reached. You see what
  was looked at, not only what went wrong.
- **A record of every step**, kept where a restart cannot erase it, so you can
  see afterwards what happened and in what order.

Starting one asks for your password, because an update outlives the session you
started it in.

**IT DOES NOT TAKE A BACKUP, AND THIS PAGE SAID IT DID.** Corrected 2026-08-29.

Pressing the button runs the six checks and writes down that you pressed it.
That is all it does. It changes nothing, and it takes nothing.

The backups are taken by the upgrade itself, and the upgrade is the step you run
on the server, described further down this page. So on an ordinary installation,
where you do the last step yourself, the backups happen when you run it and not
when you press the button.

**Take your own backup before you upgrade.** They are also taken for you once a
day into your backup directory, so the shortest honest instruction is to look in
that directory and check the newest one is from today before you start.

Why this matters more than an ordinary wrong sentence: the section below on
going back tells you to restore the backup taken before the update. Believing
this page, you would have pressed the button, upgraded, hit trouble, and gone
looking for a copy that was never made.

## Who replaces the containers, and how to tell which it is here

**CORRECTED 2026-08-31. This section used to say "PathLMS does not replace its
own containers, and it is built so that it cannot", and then told you the final
swap was yours. On a deployment installed from a release, that is wrong, and it
is wrong in the direction that matters: it describes a safety property your
deployment may not have.**

Half of it is still exactly true and is worth keeping. **The container that
serves the internet cannot create containers and is built so that it cannot.**
Creating a container accepts arbitrary bind mounts, so the right to create one on
a machine is the right to be root on that machine, and handing that to the part
facing the internet would undo the rest of the hardening in this stack.

What was missing is that a **separate small updater container** holds exactly
that authority and does nothing else, and that **the settings file a release
produces switches it on for you.** Two lines near the bottom of your `.env`, both
uncommented:

```
COMPOSE_PROFILES=updater
PATHLMS_UPDATER_ENABLED=yes
```

With both present, pressing the button in Settings is the whole of an update.
PathLMS checks, pauses, takes the backups, and the updater replaces the
containers. Nobody goes to the server.

**What you are accepting, said once and plainly.** Whoever controls the published
PathLMS images can change this deployment with nobody here approving it. That is
not a side effect, it is the feature. What keeps the trade small is that the
updater never takes an image address from PathLMS, only a version number it
checks itself, and it reads PathLMS's own record without being able to write to
it, keeping its memory somewhere PathLMS cannot reach.

**What it does hold, said rather than glossed over.** It holds the container
service, which is the authority described above. It holds your backup directory,
and it writes there, because taking the backups during an update is its job, and
those backups are complete copies of your database and of every uploaded file. It
is on no internal network, so it has no route to the database, the cache or the
file store while they are running, but that is a statement about reachability and
not about what it can read from disk.

**To have the swap be yours instead**, remove either line, or comment it out, and
bring the stack up again. Pressing the button then runs the safety checks and
writes the update down, and **nothing else happens until somebody runs the
upgrade here.** That command is what takes the backups and replaces the
containers, and the steps for it are below and do not change.

**The screen does not currently tell anybody that**, which is worth knowing
before you turn the updater off. It says the next step takes the backups, and on
this arrangement the next step is a person going to the server.

**To find out which of the two you are in right now**, run `check-my-settings.sh`
beside your compose file. It reports the answer in a sentence and judges nothing.

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

   Any change the new version needs to make to the database runs on its own at
   startup, in a transaction, recorded so it cannot run twice.

**Move all three addresses together, every time.** They are published from one
job under one version, and mixing them is not something anybody tests. The
database image in particular is not just a schema: it also carries the backup and
restore scripts your deployment runs, so leaving it behind leaves your backup
code behind with it.

## The "Back in a moment" page, and how to end it

While an update runs, everybody visiting PathLMS meets a page saying it will be
back in a moment. That page is deliberate. An update takes two safety copies,
one of the database and one of the uploaded files, and a copy taken while people
are still saving work is a copy that misses things. The page is what stops that.

**It is not a fault, and you do not normally have to do anything.** It ends
three ways:

- The update takes it down when it finishes.
- If the update stopped part way through and PathLMS came back without it,
  PathLMS notices that it is serving again and takes the page down by itself,
  usually within a minute of coming back. It does that only when it can observe
  that the update reached the point of replacing the containers and that the
  PathLMS now answering is not the one that put the page up, so it can never end
  a pause while a safety copy or a restore is still running.
- Every notice carries a deadline in any case, so even one PathLMS cannot make
  sense of ends within a day at the very most.

**To end it now, the page tells you how.** While a deployment is paused, the
"Back in a moment" page carries a paragraph addressed to whoever looks after the
system, naming the exact command to run on the server. It is written for the
person locked out, so it is on the one screen that person can still reach. The
install instructions that come with every release carry the same command.

**If PathLMS will not start at all**, so that command has nothing to talk to,
removing the file it reads does the same job and needs nothing running:

```bash
docker run --rm -v pathlms_upgrade-state:/state alpine rm -f /state/paused.json
```

**Do not end it while an update is genuinely running.** The page itself names
the step the update is on, so read it before you decide. Waiting costs minutes.
Ending it early can cost you a safety copy that turns out to be incomplete on
the day you need it.

## Going back

Put the three previous addresses back in `.env` and run `docker compose up -d`
again. That is the whole of it for the software.

**The data is the part that does not simply revert.** A newer version may have
changed the shape of the database, and going back to older software over a newer
database is not something to attempt hopefully. That is what the backup taken
before the update is for: restore it, and you are back where you were, having
lost only whatever happened in between.

Which is the real argument for testing a restore before you need one. [After it
starts](AFTER-IT-STARTS.md) has the commands.

## Version numbers, and what they promise

PathLMS follows Semantic Versioning. While the version begins with `0.`, the
specification itself says that anything may change at any time, so a change in
the middle number can carry a break. **Read the release notes before you move,
and take the backup either way.**
