# How updates work

The easy way is the button on the Updates screen. It does everything below for
you, including the parts you would otherwise have to look up. Your data stays
where it is. Nothing updates itself unless you turn that on.

Doing it by hand means changing a few lines in your settings file and
restarting. Both ways are here.

If you have not installed PathLMS yet, start with [Deployment](DEPLOYMENT.md).

## Before you update

Take a backup. PathLMS also takes one automatically, once a day, into your
backup directory. Check there is a copy from today before you start.

If anything goes wrong, you restore that backup. See "Going back" below.

## Doing the update

On the server, in the directory holding `docker-compose.yml` and `.env`:

1. Write down your current image addresses. This is your way back.
2. Open the [release page](https://github.com/Path-LMS/PathLMS/releases/latest)
   for the version you want, and copy the addresses it lists.
3. Replace those lines in `.env`:

       PATHLMS_API_IMAGE=...
       PATHLMS_WEB_IMAGE=...
       PATHLMS_DB_IMAGE=...

   Change them all together. They are published as one set, and mixing
   versions is not something anybody tests. The database image also carries
   the backup and restore scripts, so leaving it behind leaves those tools
   behind too.

   Two more parts may be named as well, the cache and the file store:

       PATHLMS_CACHE_IMAGE=...
       PATHLMS_OBJECT_STORE_IMAGE=...

   PathLMS does not build those two. It carries the versions it was tested
   with, and the release page lists them when they have moved. If your
   settings file does not name them at all, you are on the versions the
   compose file names, and that is fine. The button does this part for you.

4. Start it:

       docker compose up -d

   This pulls whatever is new and restarts only what changed. The rest is
   left alone.

5. Watch it come up:

       docker compose logs -f api

   Any database change the new version needs runs on its own, safely, when
   it starts.

## When an update moves the database to a new version

Now and then an update carries a new major version of the database. PathLMS
tells you before it starts, and it does the whole thing itself.

It takes a copy. It builds the new database beside the old one, which stays
exactly where it is. It loads the copy in. Then it counts everything, every
table, every row, every account, every value it keeps locked away, and only
switches over when the counts match. If anything does not match, or anything
fails part way, it puts the old database back and comes up on the version you
were already on.

PathLMS is paused while this happens. Measured here: about half a minute, plus
about a tenth of a second for every megabyte of database. A 356 megabyte
database paused for 57 seconds.

The old database is not deleted. It sits on your disk until you remove it
yourself.

## When an update moves the cache to a new version

The cache holds sign-ins and a few counters. Nothing in it is yours and nothing
in it is permanent: everything there can be rebuilt by people signing in again.

Going forward costs nothing. People stay signed in.

Going back to an older PathLMS across a new cache version signs everybody out
once. They sign in again and carry on. Nothing anybody saved is affected.

There is one thing to know if you do that by hand. The older cache cannot read
the file the newer one wrote, so it refuses to start, and PathLMS cannot sign
anybody in while that is true. `scripts/rollback.sh` handles this for you: it
moves that file aside, starts the cache, and tells you it did. If you went back
by editing your settings file instead, and PathLMS does not come back, this is
almost certainly why. The cache says so in its own log:

    docker compose logs redis --tail 20

Look for "Can't handle RDB format version". The cure is to move the `.rdb` file
in your data folder's `cache` directory out of the way and start again. Nothing
in that file is yours.

## Knowing an update exists

You do not have to watch the release page. PathLMS checks every few hours and
tells you on the administrator home screen and in Settings, General,
Updates.

The check only asks a public list of version numbers. It sends nothing about
your installation or the people using it. If it cannot reach the internet, it
says so rather than telling you that you are current.

Turn it off with `PATHLMS_UPDATE_CHECK=off` in your settings file. If you run
your own copy of the images inside your network, point
`PATHLMS_UPDATE_CHECK_IMAGE` at that instead.

## The Updates screen

Open Settings, then Updates, on the General tab. It shows the version you are
running, whether a newer one exists, a set of safety checks (the database,
cache and file store all answer, there is disk space free, your backup
destination can be reached, and no upgrade is already running), and a record
of what happened last time.

Pressing the button asks for your password, because an update outlives the
page you started it from. It runs the safety checks and writes down that you
pressed it. It does not take a backup and it does not touch your deployment.
The real backup happens in step 3 above, when you run the update, unless you
turn on the automatic updater below.

## The automatic updater

Two lines near the bottom of your `.env`, when both are present and
uncommented, turn on a small updater that replaces the containers for you.
With this on, pressing the button in Settings is the whole update:

    COMPOSE_PROFILES=updater
    PATHLMS_UPDATER_ENABLED=yes

Turning this on means whoever publishes PathLMS images can change your
deployment, with nobody here approving it first. If you would rather do the
swap yourself every time, remove or comment out either line and run
`docker compose up -d` again.

To check which mode you are in, run `check-my-settings.sh` next to your
compose file. It answers in one sentence.

## The "Back in a moment" page

While an update runs, anyone visiting PathLMS sees a page saying it will be
back soon. This protects the backup: copying the database while people are
still using it would miss things.

You do not normally have to do anything. The page comes down by itself when
the update finishes, or if PathLMS comes back without one running, usually
within a minute. Either way, it clears itself within a day at the most.

If you need to end it sooner, the page names the exact command, written for
whoever looks after the system. The install instructions in every release
carry the same command. If PathLMS will not start at all, remove the file it
reads instead:

    docker run --rm -v pathlms_upgrade-state:/state alpine rm -f /state/paused.json

Do not do this while an update is genuinely still running. Read what the page
says it is doing first: ending a backup early can leave you with one you
cannot use.

## Going back

Put the previous image addresses back in `.env` and run `docker compose up -d`
again. That undoes the software. Every version you have updated from is still
on the machine, so this does not need a download and works with the internet
unplugged.

Those old versions take about one and a half gigabytes of disk each. This
clears the ones you are not using, and always keeps the two most recent so the
way back stays:

    bash scripts/remove-old-images.sh

It says what it would remove and removes nothing until you add `--yes`.

The data usually will not go back on its own. A newer version can change the
database, and running older software against a newer database is not safe to
try. Restore the backup from before you updated instead, and you are back
where you were, minus whatever happened since.

This is why testing a restore before you need one is worth doing.
[After it starts](AFTER-IT-STARTS.md) has the commands.

## Version numbers

PathLMS follows Semantic Versioning. While the version starts with `0.`, a
middle-number change can still break something. Read the release notes
before you move, and take a backup either way.
