# Running PathLMS on Unraid

**This covers only what Unraid decides differently. It is not the installation
instructions.**

`INSTALL.md`, which came with the same release as this file, is the general
install. Follow it, with one change: the Compose plugin holds both files for you,
so section 2 below replaces its first two steps, about making a directory and
renaming the settings file. Every other step is correct as written. Come here for
the four things an Unraid server answers in its own way:

1. How a stack of eight containers is run at all, since Unraid's own Docker tab
   adds one container at a time. Seven keep running; the eighth sets up the file
   storage once and exits.
2. Where the data goes, given an array and one or more pools. On this platform
   that is the question with the most durable consequences.
3. Which ports are opened, given that Unraid's own web interface has taken two.
4. What backs it up, given that you almost certainly already back something up.

**Only the first is peculiar to this platform.** Every deployment has to decide
where its data lives, which ports it opens and what backs it up. Where a section
is really the general question, it says so and points at the general answer
rather than inventing a second one.

**Part of this was watched on a real Unraid server on 2026-08-30, and the rest
was not.** Every screen and control named in sections 1 and 2 was observed being
used. Everything else came from this release's own files or from Unraid's own
documentation. The last section, "What is not verified here", draws the line
item by item. Read it before you rely on this one.

---

## Before you start

- **The array has to be started and the Docker service has to be on.** Nothing
  in this stack can run before either, and the data directories you are about to
  choose live on storage the array provides.
- **You need the files from the release page.** `docker-compose.yml`,
  `pathlms.env`, `INSTALL.md` and this file. One more comes with them and you
  want it on the server: `CHANGING-THE-PORT.md`, which is how you move a port
  later. Copy it into the stack's own folder, which section 2 names, so that it
  is there on the day PathLMS is not answering and you need it.
- **You need a terminal on the server at some point.** The command that
  generates your credentials, in step 3 of `INSTALL.md`, runs there.

---

## 1. Use a Compose plugin, not Unraid's container templates

**Install Compose Manager Plus from Community Applications.** It puts the
`docker compose` command on the server and adds a Compose tab to manage stacks
from.

1. In the Unraid web interface, click the **Apps** tab in the top row.
2. Type `Compose Manager Plus` into the search box and press Enter.
3. Click **Install** on the result of that name.
4. Wait for the install to finish and close the window it opened.
5. Look at the top row of tabs again. There is now a **Compose** tab, or a
   Compose section under the Docker tab depending on your Unraid version. That
   is where everything in section 2 happens.

There is an older plugin called Docker Compose Manager. Its own author has
deprecated it in favor of this one, and the newer plugin reads the same project
folders, so having the old one installed already is not a problem.

**Why not the Docker tab.** It adds one container per template, and there is no
template mechanism for the three things this stack depends on:

- **Waiting for another container to report healthy.** The application does not
  start until the database says it is ready, and the web server does not start
  until the application says it is.
- **Running a container once, to completion, before another starts.** One
  container sets the ownership of the uploads directory and exits. The object
  store waits for it. Without that ordering the object store starts, reports
  itself healthy, answers its health check correctly, and cannot write a single
  byte. Every signal you have is green while file upload is dead. The compose
  file says so in its own comments, because it is a failure somebody already had.
- **Three container networks**, two of them closed off from the internet
  entirely, so that reaching one container is not reaching all of them. The
  database and the cache sit where nothing outside can reach them.

Rebuilding that as eight templates means creating the networks by hand,
maintaining more than twenty environment values per container by hand, and
losing the ordering outright. Do not.

---

## 2. Create the stack

1. On the Compose tab, add a new stack and call it `pathlms`, **all lowercase**,
   and leave the Stack Directory box on `default`.

   **The lowercase matters and the reason is not the one you would guess.** The
   containers are named `pathlms-api` and so on either way, because the compose
   file pins that name itself and it does not come from this box. What the name
   decides is the folder your two files land in, which is
   `/boot/config/plugins/compose.manager/projects/<what you typed>/`. Every
   command on this page that you run in a terminal uses that path in lowercase,
   including the one that checks the deployment is alive and the one that
   gathers evidence when something is wrong. Type it with capitals and each of
   those is wrong by one letter, on a system where that matters.

   The dialog also has an Advanced section holding a Stack Directory box, which
   reads `default`. Leave it. It decides where the plugin keeps stacks in
   general rather than anything about this one.

   **The compose file editor is behind the cog beside the stack's name.** It
   names the file it is editing across the top of the page, and it has a CANCEL
   and a SAVE CHANGES button. Nothing you type is written until you press SAVE
   CHANGES. (Observed on a real server, 2026-08-30.)

   **The same cog also offers an editor for the settings file, and it writes
   straight to `.env`.** Its heading reads `.../projects/pathlms/.env`, so the
   renaming step that the rest of this page insists on is already done for you.
   You never handle a file called `pathlms.env` on Unraid at all: you copy what
   is inside it into that editor.

   (Both editors, and the `.env` in that heading, observed on a real server on
   2026-08-30. This page said twice that day that the settings editor might not
   exist. It does.)

2. Open the compose file editor and paste in the whole of `docker-compose.yml`
   from the release. Do not edit it. Press SAVE CHANGES.

   **Check the heading before you paste**: it must end in `docker-compose.yml`.
   Pasting the compose file into the environment editor, or the other way round,
   is the mistake this page warns about twice below, and the heading is where you
   catch it in one glance.

   You will know the paste landed because the first line of the file reads
   `# ===...` and the text a few lines down says the stack is the one an
   administrator receives rather than the one a developer runs.

3. Open the settings editor behind the same cog, paste in the whole of
   `pathlms.env` from the release, and press SAVE CHANGES. Check its heading ends
   in `.env` before you paste. This is the file you fill in, and steps 3 and 4 of
   `INSTALL.md` say how. Come back here for the two storage settings, which are
   section 3 below.

4. **Turn on autostart for the stack**, using the AUTO START switch at the right
   of the stack's own row on the Compose tab. It arrives OFF. Unraid's Docker tab
   has its own autostart switches and they do not govern a Compose stack, so a
   server that reboots comes back with PathLMS stopped unless you set this one.
   (Switch and its position observed on a real server, 2026-08-30.)

### Every setting goes in that one file, and you edit it in a browser

**You do not need a terminal to change a PathLMS setting on Unraid.** The
settings editor behind the cog beside the stack is the form: open it, add or
change a line, and press SAVE CHANGES. Everything on this page that says
"settings file" means that editor.

**Then use the stack row's COMPOSE UP control.** The row for your stack carries
four things on the right: COMPOSE UP, COMPOSE DOWN, UPDATE STACK, and an AUTO
START switch. COMPOSE UP is the one that creates the containers again from the
file you just changed, and it is what every instruction below means by "bring the
stack up again".

**There is no Start control on this row**, which is worth saying because it is
what an earlier version of this page warned you about. Observed on a real server
on 2026-08-30: the three buttons are the ones named above and nothing else. If
your plugin version does show a Start as well, it is the wrong one: a container
keeps the settings it was created with, so starting a stopped one applies nothing
you have just typed.

**UPDATE STACK is not the one you want either**, unless you are deliberately
pulling newer images. It is next to COMPOSE UP and does a different job.

**Unraid's own Docker tab builds a container from a template, and a template is
where you would otherwise add a variable. That is not the route here**, for a
reason that is about Compose rather than about Unraid and that decides where
every future setting goes: most PathLMS settings are read by Compose while it
reads `docker-compose.yml`, before any container exists. They are not values a
container holds, so there is nowhere on a container to put them.

Unraid marks the containers it created itself and offers a template only for
those, so these seven should show no template at all. **If the Docker tab does
offer you an edit form on one of them, do not use it.** It is not governing
these containers, and anything typed there would not reach the stack. The
troubleshooting section at the end of this page says the same thing about
starting and stopping them.

| Setting | What reads it, and when |
| --- | --- |
| The three image addresses | Compose, to decide which image to pull |
| `PATHLMS_DATA_DIR`, `PATHLMS_BACKUP_DIR` | Compose, to decide which directory to mount |
| `PATHLMS_PUBLISHED_PORT`, `PATHLMS_PUBLISHED_ADDRESS`, `PATHLMS_PUBLISHED_TLS_PORT`, `PATHLMS_PUBLISHED_TLS_ADDRESS` | Compose, to decide which port to open |
| `PATHLMS_PUBLIC_URL`, `PATHLMS_TRUST_FORWARDED_PROTO` | Compose, which then passes them on to a container |
| Every password, key and address | The application, which is handed the whole file |

**Two of them are worth singling out**, because a value set on a container would
half work and the half that fails is the invisible half.
`PATHLMS_PUBLISHED_PORT` and `PATHLMS_PUBLISHED_ADDRESS` are read twice: by
Compose, to decide which port is opened, and by the application, so the Ports
section of the Network tab can report which port you are on. Set on a container
only, that screen would report a port nothing had opened.

**The passwords settle it on their own.** The application container is handed
the settings file whole, by name, so the file has to exist and has to be called
`.env` no matter what else you do. There is no arrangement in which you do not
have that file. On Unraid the plugin makes it for you under that name, which the
next section is about.

### The settings file has to be called `.env`, and this is the step that breaks installs quietly

**The release attaches the settings file as `pathlms.env`, and the compose file
asks for it by the name `.env`, in the same folder as itself.** Its own first
line tells you to rename it, and it is worth repeating here because on Unraid
you are not moving files around by hand and it is easy to skip.

Named anything else, the file is simply not read. Every container starts with no
settings at all, and what you get back is a stack that will not come up with
failures that name nothing useful.

**Where that folder is.** The plugin keeps each stack in
`/boot/config/plugins/compose.manager/projects/<stack name>/`, so for a stack
called `pathlms` your two files end up there as:

```
/boot/config/plugins/compose.manager/projects/pathlms/docker-compose.yml
/boot/config/plugins/compose.manager/projects/pathlms/.env
```

**On Unraid the plugin does the renaming for you, and this is the one place this
page is easier than the general instructions.** The settings editor behind the
cog writes to `.env` directly: its heading says so. Open it, paste the contents
of `pathlms.env` in, press SAVE CHANGES, and the file is correct by
construction. There is no step where a file called `pathlms.env` sits on the
server waiting to be renamed, because you never put one there.

**If you copied the file onto the server some other way**, over the network or
with a file manager, rename it on the way in:

```
cp pathlms.env /boot/config/plugins/compose.manager/projects/pathlms/.env
```

**And if you would rather use a terminal anyway**, Unraid opens one from the icon
that looks like `>_` at the top right of its own web page:

```
cd /boot/config/plugins/compose.manager/projects/pathlms
nano .env
```

Paste, then press Ctrl+O, Enter, Ctrl+X. That is write, confirm, and leave.

**Check it before starting the stack**, because this costs one command and saves
an hour:

```
ls -la /boot/config/plugins/compose.manager/projects/pathlms/
```

You want to see `.env` in that listing. If you see `pathlms.env` instead,
nothing has read it yet.

### Pasting into the browser editor can add an invisible character to every line, and it stops updates

**This is worth two minutes now because it costs an evening later, and it is
invisible in every editor including the one you paste into.**

Windows ends every line of a text file with two characters where Linux uses one.
A file that has passed through a Windows machine, a browser saving it, or a
copy and paste, can arrive with the extra one on every line. Nothing on screen
shows it and the stack starts perfectly.

**What it breaks is the update.** The part of PathLMS that carries out an update
reads the file listing your containers before it does anything. To find its own
section of that file it looks for a line that ends in a colon, and with the extra
character on the end it never does. So it refuses to act, parks itself, and
writes the reason into its own log where nobody is looking. Pressing update then
does nothing at all, for ever, with no explanation on any screen.

**Check both files in one command**, on the Unraid terminal:

```
# counts the invisible characters in the two files that make up the stack
tr -dc '\r' < /boot/config/plugins/compose.manager/projects/pathlms/docker-compose.yml | wc -c
tr -dc '\r' < /boot/config/plugins/compose.manager/projects/pathlms/.env | wc -c
```

**Zero from both is what you want.** Any other number means the file has them.

**Fix it with one command per file**, and it changes nothing else:

```
sed -i 's/\r$//' /boot/config/plugins/compose.manager/projects/pathlms/docker-compose.yml
sed -i 's/\r$//' /boot/config/plugins/compose.manager/projects/pathlms/.env
```

**Then restart the stack** so the change is picked up.

**The checker a release attaches finds this for you**, along with several other
things that are easy to get wrong when pasting into a text box. It changes
nothing and takes seconds:

```
cd /boot/config/plugins/compose.manager/projects/pathlms
bash check-my-settings.sh
```

Run it before your first start, and again any time you edit either file in the
browser editor.

### Both of those files are on the flash drive, and one of them holds every secret

`/boot` is the USB flash drive Unraid starts from. That is fine for the compose
file, which is read once when the stack starts.

**The settings file is a different matter, because it holds every secret this
installation has**: the key that signs everybody's sign-in, the database
passwords and the file store password. Unraid's automated flash backup copies
configuration off the flash drive, and plugin configuration is part of what it
copies. Whether it picks up this particular folder is something to check on your
own server rather than take from this page.

**Do not move that file somewhere else on a first install.** An earlier version
of this page suggested it, and the suggestion was wrong in a way worth spelling
out. Compose looks for `.env` **beside the compose file**, in the same folder,
and nowhere else. Move it and the application gets no settings at all, which is
the same silent failure as naming it wrongly.

**Whether the plugin can be told to hand Compose a settings file from
another path is not verified here**, and it is not the same thing as moving the
file. If you want that, establish it against your own server first, on an
installation that is already working, so that you can tell a settings problem
from a path problem. Getting the encryption of that flash drive right, or
excluding the folder from whatever backup worries you, is the smaller change.

---

## 3. Where the data goes

**Put the data and the backups on different storage, so that losing one does not
lose the other.** That is the whole of this section, and on Unraid the reason
lands harder than anywhere else. Two settings decide it, and both live in
`.env`:

```
PATHLMS_DATA_DIR=/mnt/cache/appdata/pathlms
PATHLMS_BACKUP_DIR=/mnt/user/backups/pathlms
```

`PATHLMS_DATA_DIR` holds the database, the uploaded files and the cache.
`PATHLMS_BACKUP_DIR` holds the backups.

### Put the data on a pool, with no secondary storage

The share `PATHLMS_DATA_DIR` sits in is `appdata`, from the setting above.

1. In the Unraid web interface, click the **Shares** tab in the top row. This is
   Unraid's own settings for storage and has nothing to do with the Compose
   plugin.
2. Click **appdata** in the list.
3. Set **Primary storage** to your pool, which is usually called `cache`. A pool
   is your solid state drives rather than the array.
4. Set **Secondary storage** to **None**.
5. Click **Apply**.

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

1. On the same **Shares** tab, click the share `PATHLMS_BACKUP_DIR` sits in,
   which is `backups` if you used the setting above as written. If it is not
   there yet, click **Add Share**, name it `backups`, and then open it.
2. Set **Primary storage** to **Array**.
3. Click **Apply**.

**Pools are not parity protected.** Unraid's documentation is explicit that
redundancy in a pool comes from mirroring rather than from parity, and that a
single-device pool has none at all. So on the very common Unraid server with one
cache drive, the database you just put on the pool has no redundancy whatsoever.
That is a perfectly reasonable arrangement, and it is only reasonable because
the backups are somewhere else.

The stack takes a backup of its own accord, once a day, keeping fourteen days.
Section 6 covers it.

### The data folder: `/mnt/cache/...` is recommended, and `/mnt/user/...` works since 0.99.1

**Corrected 2026-09-03, after an installation that followed the previous
version of this page could not upload a file and then could not update.**
This page used to recommend `/mnt/user/appdata/pathlms`, Unraid's share
layer, which presents the pool and the array as one tree. That layer does not
support extended attributes, and the file store keeps a small label on every
file it holds: on `/mnt/user` it starts, reports healthy, and refuses every
write. Nothing uploads, the Updates card's "Uploaded files can be written"
check fails, and a new version that needs the database changed waits on that
check.

Since 0.99.1 the store keeps those labels in ordinary files instead, so
`/mnt/user` works again. This page still recommends the same folder addressed
directly on the pool, `/mnt/cache/appdata/pathlms`: a database is better off on
a plain filesystem than behind the share layer, which is long-standing community
advice this project has not measured, and with secondary storage set to None, as
the steps above require, the two paths name the same files, so changing the
setting moves nothing.

The one trade: a `/mnt/cache/...` path names one specific pool, so moving the
data to a different pool later means editing this setting and bringing the stack
up again. The backups folder has no such requirement and may stay on
`/mnt/user`.

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

### The ports

Leave PathLMS on port 3001 and let a proxy hold your certificate. That is what
most Unraid servers already have, and it avoids the one change on this platform
you do not want to make.

**PathLMS opens two ports, not one.** Port 3001 is the plain one, and port 3443
is the encrypted one. Both are opened on every installation, and the settings
that move them are `PATHLMS_PUBLISHED_PORT` and `PATHLMS_PUBLISHED_TLS_PORT`.
Unraid's own web interface uses ports 80 and 443 by default, so neither collides
with a stock server. Check that nothing else you run has taken either.

**Why 3443 is open even when you are not using it.** The container listens there
on every installation, and publishing it is what lets a deployment turn on
encryption later without a second file and a longer command. Until a certificate
exists, nothing is served on it: a connection is accepted and then refused
outright, so there is no page and nothing to click past.

**Do not publish PathLMS on 80 or 443 on a stock Unraid server.** Doing so means
moving Unraid's own web interface first, under Settings then Management Access,
which changes how you reach Unraid itself. That is a decision about your server
rather than about this product.

### Behind a reverse proxy

This is the usual Unraid answer, and there is a page for it with the exact
steps, a copyable nginx block, the Caddy equivalent, and how to prove it worked:
**[Putting PathLMS behind a proxy you already run](BEHIND-A-PROXY.md)**, which
lives with the rest of the deployment documentation under
`docs/public-repo/deploy/`. What follows here is the short version.

Point your proxy at `http://<your-server-address>:3001`, let the proxy hold the
certificate, and add two lines to your settings file. That means the editor
for `.env` in a browser, as above, rather than a terminal:

```
PATHLMS_PUBLIC_URL=https://learn.example.com
PATHLMS_TRUST_FORWARDED_PROTO=true
```

**The second one is not optional and it is easy to miss.** Without it, PathLMS
treats every request as unencrypted no matter what your proxy did, because it
believes only its own connection. Turning it on tells PathLMS to believe the
proxy when the proxy says a visitor arrived over an encrypted connection. Only
turn it on when something really is in front, because with nothing in front it
would let any visitor make that claim about themselves.

**Then close the ports nobody should be reaching around the proxy to. Which
answer is right depends on how your proxy runs, and on Unraid it is usually the
second one.**

**If the proxy is a Docker container**, which is what Nginx Proxy Manager, SWAG
and Traefik are on Unraid, **do not use the two settings below.** Inside a
container `127.0.0.1` means that container rather than the server, so setting
them would leave your proxy unable to reach PathLMS at all. Restrict the two
ports with a firewall rule, or on your router, so that only the proxy can reach
them.

**If the proxy runs directly on the server**, or on a different machine that you
reach over the network, these two settings shut both ports to everything except
the server itself:

```
PATHLMS_PUBLISHED_ADDRESS=127.0.0.1:
PATHLMS_PUBLISHED_TLS_ADDRESS=127.0.0.1:
```

The trailing colon on each is required and is not a typo. If your proxy is on a
different machine, leave these alone as well and use a firewall rule to allow
port 3001 from the proxy's address only. The proxy page covers every case.

**Then press COMPOSE UP on the stack's row, and this matters more than it
looks.** A container keeps the settings it was created with, so stopping and
starting one applies nothing you have just typed. The containers have to be
created afresh from the changed file, which is what COMPOSE UP does. If a
setting looks as though it was ignored, this is almost always why.

The web container writes down on every start whether it is believing the
forwarded claim, and the proxy page has the one command that reads that back, so
you can tell the setting arrived without inspecting a response header.

**Get that address right before the first start.** It decides which pages a
browser is answered from, the links inside recovery mail, the address file links
are signed against, and where company sign-in sends people back to. A wrong
value gives you a sign-in page that draws perfectly and then does nothing, with
nothing on that screen pointing at a setting.

### Then tell PathLMS which shape it is in

Once you have signed in as the administrator, go to **Settings**, then the
**Network** tab, and choose one of three:

The section is called **How this system is reached**, and the three answers are
worded on screen exactly like this:

| Option | When |
| --- | --- |
| PathLMS handles encryption itself | Browsers connect straight to PathLMS and it holds the certificate. |
| A reverse proxy in front of PathLMS handles encryption | You run nginx, Caddy, Traefik or a load balancer in front. This is the usual Unraid answer. |
| Nothing encrypts the traffic | Nothing is encrypted at any point on the way. Reasonable only on a closed network that nobody outside can reach. |

Pick one and press **Record this answer**.

**If you choose the first option, there is nothing extra to add to the stack.**
The encrypted port, 3443 by default, is already open, and what it waits for is a
certificate. Once you have recorded that first answer, a heading called **The
security certificate** appears in this same section, offering **Make a security
certificate here** and **I already have a certificate**.

**Recording the answer is what makes those two buttons appear**, and they are
not drawn under the other two answers, because under those there is nothing to
press. So looking for a certificate control before choosing is looking for
something that is not there yet. Unraid's own firewall and any router in front
of it have to allow that port, which is the part an Unraid server answers in its
own way.

**Choosing the middle option is the point of this step for most Unraid
servers.** Until you do, PathLMS has to work it out from the address alone, so
it offers to make you a security certificate you already have and do not want.
Saying so once stops that.

**This screen is not the same thing as the setting above, and both are needed.**
`PATHLMS_TRUST_FORWARDED_PROTO` decides what the server believes about a
request. This screen decides what the screens say to you. They are deliberately
different, because anybody who can save a setting inside the product should not
be able to change what the server enforces.

**What it does not do is reconfigure your deployment**, and the screen says so
itself. It records your answer. It does not move your proxy, your certificate or
your ports.

**One thing not to do**: if your reverse proxy already holds a certificate, do
not also generate one inside PathLMS. A certificate the browser does not trust,
served together with an instruction telling browsers to insist on encryption, is
how somebody locks themselves out of their own system with no way back from
inside it.

**This is not an Unraid question.** Every deployment behind a proxy answers it
the same way. It is here because on an Unraid server a reverse proxy is close to
universal.

---

## Check it worked, in this order

Four checks. They are in this order because each one rules out a whole class of
problem before the next one can confuse you, and because the last two prove
things a page loading successfully does not.

Checks 1 and 4 are commands. Run them in a terminal on the Unraid server, in the
stack's own folder, which is
`/boot/config/plugins/compose.manager/projects/pathlms/`. Checks 2 and 3 are done
in a browser.

1. **Ask whether the application is ready**, at exactly this address:

   ```
   curl -sS http://localhost:3001/api/health/ready
   ```

   Use `localhost` rather than the server's network address, so that this check
   still works if you closed the ports in section 5. If you moved the port, use
   the number you set in `PATHLMS_PUBLISHED_PORT`.

   **The path matters and a shorter one will lie to you.** The web server
   answers requests for the site itself even when the application behind it is
   dead or still starting, so a plain request to the address gives you a page
   either way. This path is proxied through to the application, so an answer
   here means the application is running and has reached its database. If the
   application is down you get a gateway error rather than a page, which is
   what you want a check to do.

   If this fails, stop and read the log before doing anything else:

   ```
   docker compose logs api
   ```

2. **Sign in.** Go to your address in a browser and sign in as the first
   administrator, using the two values you put in the settings file. This proves
   the address is right, that the browser is answered rather than refused, and
   that the account was created.

   A sign-in page that draws perfectly and then does nothing when you submit it
   is almost always `PATHLMS_PUBLIC_URL` not matching the address you typed.

3. **Upload a picture and confirm it appears.** Set a person's photograph under
   People, or put a picture into a lesson, then reload the page and look at it.

   **This is the check that proves the file storage set itself up**, and nothing
   else proves it. The object store reports itself healthy and answers its own
   health check correctly whether or not the one-shot container that sets the
   ownership of its directory ran, so every signal you have can be green while
   file upload is dead. A picture that appears after a reload is the only
   evidence. If it fails, stop and start the whole stack rather than restarting
   the object store on its own.

4. **Confirm a backup file exists.**

   ```
   ls -la /mnt/user/backups/pathlms/
   ```

   Use whatever you set `PATHLMS_BACKUP_DIR` to. You are looking for a file
   whose name begins with `pathlms-` and ends in `.sql.gz`. That is the
   database. Several more files sit beside it, holding the uploaded files and a
   list of what the backup contained. An empty directory is the answer that
   means something went wrong.

   **The first backup runs immediately rather than in twenty-four hours, and
   that is deliberate**, so that a directory nobody can write to is a problem
   you meet on day one instead of on the day you need a backup. An empty
   directory here means the backup could not write, and section 6 is the rest of
   that story.

Once all four pass, the installation is genuinely working rather than merely
started.

---

## 6. Backups

**The stack backs itself up already**, so there is nothing to schedule. One
backup a day, kept for fourteen days, of both the database and the uploaded
files, written into `PATHLMS_BACKUP_DIR`. The first one runs immediately rather
than in twenty-four hours, deliberately, so that a directory nobody can write to
is a problem you meet on day one. You can change the interval with
`BACKUP_EVERY_SECONDS`, change how long backups are kept with `RETENTION_DAYS`,
or switch the whole thing off with `BACKUP_EVERY_SECONDS=0`.

**What Unraid adds is where those files should sit.** Section 3 says to put them
on a share whose primary storage is the array, so that parity covers them and
they are not on the same devices as the database. Then include that share in
whatever off-server backup you already run, because parity survives a drive
failing and does not survive the building.

**If you use the Appdata Backup plugin**, it stops containers and archives
`appdata`. That does not conflict with anything here, and it is not a substitute
either: it captures the files, while PathLMS's own backup captures a proper
database dump alongside an inventory of every object in the database, so that a
restore can report what it did not put back. Using both is reasonable. Check that
the plugin's schedule and the stack's own are not set to the same minute.

**Restoring is a general matter rather than an Unraid one** and works the same
way everywhere. The script that does it travels inside the database image, so
you can take a copy without needing anything else:

```
docker cp pathlms-db:/opt/pathlms/scripts/. ./pathlms-scripts/
```

**A backup you have never restored is not a backup.** Rehearse one before you
need to, on a day when nothing is wrong. There is a flag for exactly this, and
it is one command:

```
export BACKUP_DIR=/mnt/user/backups/pathlms
bash ./pathlms-scripts/restore-database.sh --latest --verify-only
```

`BACKUP_DIR` is wherever you pointed `PATHLMS_BACKUP_DIR`, and `--latest` picks
the most recent copy in it.

**`--verify-only` is what makes this a rehearsal rather than a restore.** The
script always loads the dump into a new, empty database of its own and checks it
completely; the only question is whether it then swaps that copy into place.
With `--verify-only` it never swaps, says so before it starts, and finishes with
`Nothing was replaced, because this was a --verify-only run.` Your live database
is not touched at any point.

**Do not create a database first.** The script creates what it needs, including
on a machine where nothing exists yet. Making an empty one beforehand does not
help it and gives it an empty shell to copy its text settings from.

It asks you to type the word `RESTORE` before it does anything, and it names the
file it is about to read and the database it is about to write into a few lines
above that prompt. Read them before you answer. When you do a real restore, leave
`--verify-only` off: that replaces the live database, and it renames the old one
aside rather than deleting it, so there is still a way back.

---

## 7. Updating

**By default PathLMS installs its own upgrades and you do none of this.** An
administrator presses a button, and the safety checks, the backups, the new
containers and the check that the new version really answers all happen without
anybody at a terminal. `INSTALL.md` covers it under "The easy way, which is how
this arrives". Read what it says about the trade: the extra container it runs can
replace the containers on this machine, which is the same authority as being an
administrator of the machine itself.

**This section is the other way, for a deployment that switched that off.** You
switch it off by putting a `#` in front of these two lines in `.env` and pressing
COMPOSE UP:

```
COMPOSE_PROFILES=updater
PATHLMS_UPDATER_ENABLED=yes
```

Upgrading by hand is four edited lines and one control. The release page for this
product lists every version and reports four image addresses for each. Put those
four addresses into the editor for `.env`, save, and press **COMPOSE UP** on the
stack's row. UPDATE STACK is a different job and will not do this. Inside the
product, on the General tab of Settings, the Updates section checks whether it is
safe to go ahead; replacing the containers is still yours to do, deliberately.

**The fourth address is the one people forget.** Three of them are the
application, the web server and the database. The fourth, `PATHLMS_UPDATER_IMAGE`,
is the extra container that installs upgrades. Leave it behind and your
deployment carries on installing upgrades, and taking its safety copies, with
the tools it had before, because those tools live inside that container.

**Setting that line is not enough on its own, and this page said otherwise until
2026-09-01.** It said the next restart of the machine would bring the new one
up. It does not. Restarting starts the container that is already there, from the
image it already has, and Docker reads neither `.env` nor the compose file to do
it. Pressing COMPOSE UP is what replaces it, and only because you have set the
line first.

**None of this applies if you left the easy way switched on**, which is how a
release arrives. There the extra container brings its own replacement up the
next time it starts, and says in its log which of the two it is running. This
section is only for a deployment that turned that off.

**That section does not take a backup, and this page said it did until
2026-08-29.** It runs its checks and writes down that you pressed it. Nothing
else. Take your own copy before you upgrade, or check that the one taken for you
each day into your backup directory is from today. The section on going back,
below, tells you to restore the copy taken beforehand, and believing the old
sentence you would have gone looking for one that was never made.

**One Unraid detail worth knowing.** The Compose plugin can check for image
updates on a schedule. It cannot move this deployment to a new version, because
the three addresses name digests, and a digest identifies exact bytes and cannot
be repointed at anything else. That is deliberate: it is what makes two
installations of one version provably the same software. Upgrading is always
those three lines.

### Going back

**Going back is the update run backwards, and it is the same four lines.** Put
the four image addresses you replaced back into the editor for `.env`, save, and
press COMPOSE UP again. That is why the paragraph above says
to write down what they were before you change them.

**That is the whole of it when the new version changed nothing in the
database**, which is the ordinary case. Your data is not part of it. Putting the
database container back on an earlier image does not change what is stored, so
long as it is the same major version of PostgreSQL, which an ordinary update
does not change.

**When the new version did change the database, older software cannot undo
that.** It does not know a column has gone; it fails on the first query that
wants one. Going back across a change like that means restoring the copy taken
before you updated, with the restore script from section 6, and everything
anybody did since that copy is lost. Work out which of the two you are in before
you touch anything.

**Uploaded files are never part of going back.** An update changes software and
the shape of the database. It does not touch the documents and pictures people
have uploaded, so putting the file archive back would delete every file added
since the update and gain nothing at all. `restore-files.sh` is the tool for the
separate case where files really were lost. It is attached to every release, and
it also travels in the database image, so the `docker cp` in section 6 gets you
a copy of it. This reads the newest archive back and checks it is usable, which
is worth doing on a good day rather than a bad one:

```
BACKUP_DIR=/mnt/user/backups/pathlms bash restore-files.sh --verify
```

Drop `--verify` to put the uploaded files back.

**That script has to be told where your backups are**, unlike the other three,
and it is the one thing about it that wastes people's time. It looks in
`./backups` unless `BACKUP_DIR` says otherwise, and it does not read your
settings file. Without it you get `no file archives found in ./backups`, which
names a directory that has never held anything rather than telling you it looked
in the wrong place. Use whatever you set `PATHLMS_BACKUP_DIR` to.

**`rollback.sh` is attached to every release and it cannot help on this route.**
It reads a record that only the upgrade script writes, and updating the way this
page describes writes no such record. Run it and it will say there is no record
of an update to undo, and change nothing. That is worth knowing in advance, so
that a refusal on a bad day reads as the tool working rather than as one more
thing broken.

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
- **Nothing works and no setting seems to have taken.** The settings file is not
  called `.env`, so nothing read it. See section 2. This is the most common way
  an install on this platform fails, and the failures it produces name the
  symptom rather than the cause.
- **A port is already in use.** Something else on the server has 3001 or 3443.
  Docker refuses to start and names the port. Change
  `PATHLMS_PUBLISHED_PORT` for the plain one or `PATHLMS_PUBLISHED_TLS_PORT` for
  the encrypted one, and bring the stack up again.
- **The address answers, but the browser is refused and pages do not fill in.**
  `PATHLMS_PUBLIC_URL` does not match the address you typed. Fix it and bring the
  stack up again.
- **You changed a setting and nothing happened.** You pressed Start rather than
  Compose Up. Start reuses the containers you already have, and a container keeps
  the settings it was created with.
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

**Part of this was watched on a real Unraid server and part of it was not.**
That matters, so here is the line, drawn item by item.

**Watched on a real server on 2026-08-30**, and worth more than anything else on
this page: that the Compose tab's Add New Stack dialog takes a name and an
Advanced section holding a Stack Directory box; that the stack's folder is
`/boot/config/plugins/compose.manager/projects/<the name you typed>/`; that a cog
beside the stack's name opens an editor for the file saying which containers
exist and an editor for `.env`; that each editor names the file it is editing
across the top and writes nothing until SAVE CHANGES is pressed; that the editor
for the settings file writes to `.env` directly, so no renaming is needed; and
that the stack's row carries exactly four controls, COMPOSE UP, COMPOSE DOWN,
UPDATE STACK and an AUTO START switch which arrives off.

**There is no Start control on that row**, which is worth stating because two
earlier versions of this page warned at length about pressing one.

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
can have its own settings file; and that per-stack autostart exists.

**Taken from this product's own files**, and checkable against the
`docker-compose.yml` you downloaded: everything about ports, storage settings,
container users, ownership, ordering, networks, backups and digests.

**Not verified anywhere, and marked as such above:**

- Whether Unraid's automated flash backup picks up this plugin's project folder
  in particular. Treat the settings file as though it does.
- Whether the Compose plugin can be told to hand Compose a settings file from a
  path outside the stack's own folder. **An earlier version of this page said it
  could and told people to do it, and that advice has been withdrawn.** What is
  certain, out of this release's own compose file, is that Compose looks for
  `.env` beside the compose file and nowhere else, so simply moving the file
  leaves the application with no settings at all. Do not try this on a first
  install.
- What the New Permissions tool, or Docker Safe New Perms, does to a PathLMS
  installation. The community reports of containers breaking are real; the
  effect on this stack in particular is untested.
- Whether addressing a pool directly rather than through `/mnt/user` makes any
  difference to this product. The community advice exists, it is not in Unraid's
  documentation, and the documented hazard behind it is removed by setting
  secondary storage to None.
- What a PostgreSQL data directory does when the mover moves part of it. Which
  is the whole reason this page tells you not to let that happen.

**Not watched, and the biggest gap on this page**: nothing below section 2 has
been done on a real Unraid server. The stack has never been started there, no
port has been opened there, no backup has been written there and no upgrade has
been pressed there. Sections 3 to 7 are reasoned from this release's own files
and from Unraid's documentation, which is not the same as having seen them work.

If you run this on Unraid and find any of the above to be different from what is
written here, that is worth reporting. A guess in a document that ships with
every release is worse than no document at all.
