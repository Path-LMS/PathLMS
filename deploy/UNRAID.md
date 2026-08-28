# Running PathLMS on Unraid

**This covers only what Unraid decides differently. It is not the installation
instructions.**

`INSTALL.md`, which came with the same release as this file, is the general
install, and every step in it is correct on Unraid. Follow it. Come here for the
four things an Unraid server answers in its own way:

1. How a stack of seven containers is run at all, since Unraid's own Docker tab
   adds one container at a time.
2. Where the data goes, given an array and one or more pools. On this platform
   that is the question with the most durable consequences.
3. Which port is opened, given that Unraid's own web interface has taken two.
4. What backs it up, given that you almost certainly already back something up.

**Three of those are general questions wearing Unraid clothes.** Every
deployment has to decide where its data lives, which port it opens and what
backs it up. Only the first is peculiar to this platform. Where a section is
really the general question, it says so and points at the general answer rather
than inventing a second one.

**What was checked, and how.** Every claim below about PathLMS itself was read
out of the files this release ships, and you can check it against them. Every
claim about Unraid came from Unraid's own documentation or from the plugin's own
documentation. The ones that came from neither are listed in the last section,
which is called "What is not verified here". Nobody who wrote this has an Unraid
server. Read that section before you rely on this one.

---

## Before you start

- **The array has to be started and the Docker service has to be on.** Nothing
  in this stack can run before either, and the data directories you are about to
  choose live on storage the array provides.
- **You need the files from the release page.** `docker-compose.yml`,
  `pathlms.env`, `INSTALL.md` and this file.
- **You need a terminal on the server at some point.** The command that
  generates your credentials, in step 3 of `INSTALL.md`, runs there.

---

## 1. Use a Compose plugin, not Unraid's container templates

**Unraid's Docker tab adds one container per template.** You press Add
Container, choose a template, and get a container. There is no template
mechanism for the three things this stack depends on:

- **Waiting for another container to report healthy.** The application does not
  start until the database says it is ready, and the web server does not start
  until the application says it is. Both are written in the compose file as
  `condition: service_healthy`.
- **Running a container once, to completion, before another starts.** One
  container sets the ownership of the uploads directory and exits. The object
  store waits for it with `condition: service_completed_successfully`. Without
  that ordering the object store starts, reports itself healthy, answers its
  health check correctly, and cannot write a single byte. Every signal you have
  is green while file upload is dead. The compose file says so in its own
  comments, because it is a failure somebody already had.
- **Three container networks**, two of them closed off from the internet
  entirely, so that reaching one container is not reaching all of them. The
  database and the cache sit where nothing outside can reach them.

Rebuilding that as seven templates means creating the networks by hand,
maintaining more than twenty environment values per container by hand, and
losing the ordering outright. Do not.

**Install Compose Manager Plus from Community Applications.** Open the Apps tab
in the Unraid web interface, search for it, and install it. It puts the
`docker compose` command on the server and adds a Compose tab to manage stacks
from.

There is an older plugin called Docker Compose Manager. Its own author has
deprecated it in favor of this one, and the newer plugin reads the same project
folders, so having the old one installed already is not a problem.

---

## 2. Create the stack

1. On the Compose tab, add a new stack and call it `pathlms`.

2. Open its compose file editor and paste in the whole of `docker-compose.yml`
   from the release. Do not edit it.

3. Open its environment file editor and paste in the whole of `pathlms.env` from
   the release. This is the file you fill in, and steps 3 and 4 of `INSTALL.md`
   say how. Come back here for the two storage settings, which are section 3
   below.

4. **Turn on autostart for the stack.** Unraid's Docker tab has its own
   autostart switches and they do not govern a Compose stack, so a server that
   reboots comes back with PathLMS stopped unless you set this in the plugin.

### Where those two files end up, and why you may want to move one

By default the plugin keeps a stack's files under
`/boot/config/plugins/compose.manager/projects/`, and `/boot` is the USB flash
drive Unraid starts from. That is fine for the compose file, which is read once
when the stack starts.

**The environment file is a different matter, because it holds every secret this
installation has**: the key that signs everybody's sign-in, the database
passwords and the file store password. Unraid's automated flash backup copies
configuration off the flash drive, and plugin configuration is part of what it
copies. Whether it picks up this particular folder is something to check on your
own server rather than take from this page. If you would rather that file was
never carried off in a backup, the plugin can read a stack's environment file
from a path outside the projects folder, so put it on a share you control
instead.

---

## 3. Where the data goes

**Two settings decide everything here**, and both live in the environment file:

```
PATHLMS_DATA_DIR=/mnt/user/appdata/pathlms
PATHLMS_BACKUP_DIR=/mnt/user/backups/pathlms
```

`PATHLMS_DATA_DIR` holds the database, the uploaded files and the cache.
`PATHLMS_BACKUP_DIR` holds the backups. **They are two settings on purpose, and
on Unraid the reason lands harder than anywhere else**: they should be on
different storage, so that losing the storage the data is on does not also lose
the recovery.

### Put the data on a pool, with no secondary storage

Set the share `PATHLMS_DATA_DIR` sits in, which is `appdata` above, to:

| Share setting | Value |
| --- | --- |
| Primary storage | your pool, usually `cache` |
| Secondary storage | **None** |

**Secondary storage set to None is the setting that matters, and this is why.**
When a share has secondary storage, Unraid's mover shifts files between the two
on a schedule. Unraid's own documentation says: "Always disable Docker and VM
services before moving files with the Mover. This prevents open files from being
skipped during the transfer." Read from the other side, that sentence says a
mover run with Docker running moves part of a file set and skips the rest.

The file set in question would be a live PostgreSQL data directory. **Nobody
here has tested what happens to a database whose files are half moved, and this
page is not going to guess.** With secondary storage set to None the mover has
nowhere to move it to, so the question never arises. That costs you one
dropdown.

A pool is also the right place for reasons of speed rather than only of safety.
Unraid's documentation puts it plainly: "Storing applications like Docker
containers or virtual machines on a cache pool improves their performance,
reduces wear on your main array, and minimizes the time it takes to access
frequently-used files." A database on the array writes through parity, which is
the slowest thing this product can be asked to do.

### Put the backups on the array, where parity covers them

Set the share `PATHLMS_BACKUP_DIR` sits in to primary storage of **Array**.

**Pools are not parity protected.** Unraid's documentation is explicit that
redundancy in a pool comes from mirroring rather than from parity, and that a
single-device pool has none at all. So on the very common Unraid server with one
cache drive, the database you just put on the pool has no redundancy whatsoever.
That is a perfectly reasonable arrangement, and it is only reasonable because
the backups are somewhere else.

The stack takes a backup of its own accord, once a day, keeping fourteen days.
Section 6 covers it.

### About `/mnt/user` against `/mnt/cache`

The paths above go through `/mnt/user`, which is Unraid's share layer: it
presents the pool and the array as one tree and routes each write according to
the share settings you have just set.

Some long-standing community advice says to point database containers at a pool
directly, as `/mnt/cache/appdata/pathlms`, rather than through `/mnt/user`.
**That advice is not in Unraid's own documentation and has not been tested
against PathLMS by anybody here.** The documented hazard it grew up around is
the mover, and setting secondary storage to None removes that hazard whichever
path you use.

So `/mnt/user/appdata/pathlms` is the conventional choice and is what this page
recommends. If you would rather address the pool directly you can, and the trade
is worth knowing first. A `/mnt/cache/...` path names one specific pool, so
moving the data to a different pool later means editing this setting and
restarting the stack, and Unraid's share settings no longer route it for you.

### What happens if you get this wrong

- **Data on a share whose secondary storage is the array.** The mover will
  eventually try to move a running database. See above.
- **Backups on the same pool as the data.** The pool failing takes both, which
  is the one arrangement `PATHLMS_BACKUP_DIR` exists to prevent.
- **Data on the array with no pool at all.** It works. It is slow, every write
  goes through parity, and the whole server will feel it.

---

## 4. Permissions

**There is normally nothing to do before the first start**, and that is a fact
about this stack rather than about Unraid. Docker creates the directories, and
the containers arrange their own ownership: the database image takes ownership
of its data directory when it sets itself up, and a one-shot container sets the
ownership of the uploads directory before the object store is allowed to start.

**Do not pre-create those directories and set them to `nobody:users`.** The
usual Unraid convention of running a container as user 99 and group 100 does not
apply here. The containers in this stack run as their own non-root users, and
the compose file names them: the object store runs as user 1001, and the
database and the application each run as their own.

### Do not run Tools, then New Permissions, over a share holding PathLMS data

That tool rewrites ownership across a whole share. There are community reports
of running it over `appdata` and breaking every container on the server, which
is why Unraid ships a separate Docker Safe New Perms for that directory.
**Nobody here has tested what either one does to a PathLMS installation, and
there is no reason to find out.** If you have already run one, read the recovery
below.

### Recovering from an ownership problem

Two of the three directories repair themselves, by design:

- **Uploads.** The one-shot container that sets ownership runs every time the
  stack comes up, so stopping and starting the stack puts it back.
- **The database.** Its own entrypoint takes ownership of its directory.

**The cache directory is the one to be blunt about, because nothing here claims
it repairs itself.** It holds nothing irreplaceable: losing it signs everybody
out and resets the counters that stop password guessing, and the compose file
says exactly that where the directory is declared. Accounts, progress and every
other durable thing are rows in the database. So if the cache container fails
with a permission error, stop the stack, delete the `cache` directory under
`PATHLMS_DATA_DIR`, and start the stack again.

**Never do that to the `database` or `uploads` directories.**

---

## 5. How it is reached

### The port

PathLMS opens port 3001 on the server unless you say otherwise with
`PATHLMS_PUBLISHED_PORT`. Unraid's own web interface uses ports 80 and 443 by
default, so 3001 does not collide with a stock server. Check that nothing else
you run has taken it.

**Do not publish PathLMS on 80 or 443 on a stock Unraid server.** Doing so means
moving Unraid's own web interface first, under Settings then Management Access,
which changes how you reach Unraid itself. That is a decision about your server
rather than about this product, and there is a better answer below.

### Behind a reverse proxy, which is what most Unraid servers do

Leave `PATHLMS_PUBLISHED_PORT` at 3001. Point your proxy at
`http://<your-server-address>:3001`, let the proxy hold the certificate, and set
the address people type:

```
PATHLMS_PUBLIC_URL=https://learn.example.com
```

**Get that address right before the first start.** It decides which pages a
browser is answered from, the links inside recovery mail, the address file links
are signed against, and where company sign-in sends people back to. A wrong
value gives you a sign-in page that draws perfectly and then does nothing, with
nothing on that screen pointing at a setting.

### Then tell PathLMS which shape it is in

Once you have signed in as the administrator, go to **Settings**, then the
**Network** tab, and choose one of three:

| Option | When |
| --- | --- |
| This system handles encryption itself | Browsers connect straight to PathLMS and it holds the certificate. |
| Something else handles encryption in front of this system | You run nginx, Caddy, Traefik or a load balancer in front. This is the usual Unraid answer. |
| Nothing encrypts the traffic | Nothing is encrypted at any point on the way. Reasonable only on a closed network that nobody outside can reach. |

**Choosing the middle option is the point of this step for most Unraid
servers.** Until you do, PathLMS has to work it out from the address alone, so
it offers to make you a security certificate you already have and do not want.
Saying so once stops that.

**What it does not do is reconfigure your deployment**, and the screen says so
itself. It records your answer so the rest of the Settings screen stops
guessing. It does not move your proxy, your certificate or your ports.

**One thing not to do**: if something in front already holds a certificate, do
not also generate one inside PathLMS. A certificate the browser does not trust,
served with a policy telling browsers to insist on encryption, is how somebody
locks themselves out of their own system with no way back from inside it.

**This is not an Unraid question.** Every deployment behind a proxy answers it
the same way. It is here because on an Unraid server a reverse proxy is close to
universal.

---

## 6. Backups

**The stack backs itself up already.** One backup a day, kept for fourteen days,
of both the database and the uploaded files, written into `PATHLMS_BACKUP_DIR`.
The first one runs immediately rather than in twenty-four hours, deliberately,
so that a directory nobody can write to is a problem you meet on day one. You
can change the interval with `BACKUP_EVERY_SECONDS`, change how long backups are
kept with `RETENTION_DAYS`, or switch the whole thing off with
`BACKUP_EVERY_SECONDS=0`.

**What Unraid adds is where those files should sit.** Section 3 says to put them
on a share whose primary storage is the array, so that parity covers them and
they are not on the same devices as the database. Then include that share in
whatever off-server backup you already run, because parity survives a drive
failing and does not survive the building.

**If you use the Appdata Backup plugin**, it stops containers and archives
`appdata`. That does not conflict with anything here, and it is not a substitute
either: it captures the files, while PathLMS's own backup captures a proper
database dump alongside an inventory of every object in the database, so that a
restore can report what it did not put back. Use both if you want the belt and
the braces. Check that the plugin's schedule and the stack's own are not set to
the same minute.

**Restoring is a general matter rather than an Unraid one** and works the same
way everywhere. The script that does it travels inside the database image, so
you can take a copy without needing anything else:

```
docker cp pathlms-db:/opt/pathlms/scripts/. ./pathlms-scripts/
```

**A backup you have never restored is not a backup.** Restore one into a
throwaway location before you need to, on a day when nothing is wrong.

---

## 7. Updating

The release page for this product lists every version and reports three image
addresses for each. Upgrading means putting those three addresses into your
environment file and starting the stack again. PathLMS can also do it for you
from inside the product, taking a backup first and putting everything back if
anything fails.

**One Unraid detail worth knowing.** The Compose plugin can check for image
updates on a schedule. It cannot move this deployment to a new version, because
the three addresses name digests, and a digest identifies exact bytes and cannot
be repointed at anything else. That is deliberate: it is what makes two
installations of one version provably the same software. Upgrading is always
those three lines.

---

## If it does not start

**Read the application's log first.** From the Compose tab, or from a terminal
in the stack's project folder:

```
docker compose logs api
```

Read the **first** refusal rather than the last line. This product's refusals
name the setting that is wrong.

`INSTALL.md` covers the failures that are the same everywhere, including the two
that surprise people most: a fresh installation refusing to run on a password
copied out of somewhere it should not have been, and a sign-in page that appears
and does nothing because `PATHLMS_PUBLIC_URL` does not match the address you
typed.

These are the Unraid ones:

- **Nothing starts at all.** The array is not started, or the Docker service is
  off. Both are required before any container runs.
- **The stack was running and is gone after a reboot.** Autostart for the stack
  was not turned on in the Compose plugin. Unraid's Docker tab switches do not
  govern it.
- **A port is already in use.** Something else on the server has 3001. Change
  `PATHLMS_PUBLISHED_PORT` and start the stack again.
- **The containers appear on the Docker tab looking unmanaged.** That is
  expected. They were created by Compose rather than from an Unraid template.
  Start, stop and update them from the Compose tab, not from the Docker tab.
- **It worked, and then the database would not start after you changed share
  settings.** Suspect the mover. See section 3, and check that secondary storage
  on that share is None.
- **The object store is healthy and uploads fail anyway.** The one-shot
  ownership container did not run, or did not succeed. Stop and start the whole
  stack rather than restarting the object store on its own.

---

## What is not verified here

**Nobody who wrote this has an Unraid server.** That matters, so here is the
line, drawn honestly.

**Taken from Unraid's own documentation**, and quoted above where it is load
bearing: that a share has primary storage, secondary storage and a mover action;
that Docker should be disabled before a mover run so that open files are not
skipped; that pools are not parity protected and get their redundancy from
mirroring instead; that storing container data on a pool improves performance
and reduces wear on the array; that the web interface uses ports 80 and 443 by
default and that they are changed under Management Access; and that the
automated flash backup copies plugin configuration off the flash drive.

**Taken from the Compose plugin's own documentation**: that Compose Manager Plus
is installed from Community Applications; that it supersedes the deprecated
Docker Compose Manager and reads the same project folders; that stacks live
under `/boot/config/plugins/compose.manager/projects` by default; that a stack
can have its own environment file and can read one from outside that folder; and
that per-stack autostart exists.

**Taken from this product's own files**, and checkable against the
`docker-compose.yml` you downloaded: everything about ports, storage settings,
container users, ownership, ordering, networks, backups and digests.

**Not verified anywhere, and marked as such above:**

- Whether Unraid's automated flash backup picks up this plugin's project folder
  in particular. Treat the environment file as though it does.
- What the New Permissions tool, or Docker Safe New Perms, does to a PathLMS
  installation. The community reports of containers breaking are real; the
  effect on this stack in particular is untested.
- Whether addressing a pool directly rather than through `/mnt/user` makes any
  difference to this product. The community advice exists, it is not in Unraid's
  documentation, and the documented hazard behind it is removed by setting
  secondary storage to None.
- What a PostgreSQL data directory does when the mover moves part of it. Which
  is the whole reason this page tells you not to let that happen.

If you run this on Unraid and find any of the above to be different from what is
written here, that is worth reporting. A guess in a document that ships with
every release is worse than no document at all.
