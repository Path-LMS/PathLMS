# Deploying PathLMS

You end up with a system people can sign in to, on a machine you control, in
about fifteen minutes. Most of that is the machine downloading things.

Four decisions come first, because they are the ones you live with. Everything
after them is the same on every machine, and the environment sections at the
bottom cover only what a particular environment decides differently. If your
environment is not listed, the general path is the whole install.

Nothing here needs the source code, `git`, Node.js, or a build step on the
server.

---

## Before you start: four decisions

These four have consequences you live with. Everything else can be changed from
the Settings screen once you are in.

### 1. What address will people type?

Get this one right and everything else follows from it. Get it wrong and the
sign-in page renders perfectly and then does nothing, with nothing on that screen
pointing at the cause. It is the most confusing failure PathLMS has.

Four things derive from this setting: which pages a browser is answered from, the
links inside password recovery mail, the address your uploaded files are signed
against, and where company sign-in returns people to.

Write it with the scheme and no path: `https://learn.example.com`.

### 2. Where will the data live?

Two directories, which you choose, so that you can find your data with `ls` and
so that losing one disk does not lose both copies.

Everything PathLMS stores goes under one directory, which defaults to
`/var/lib/pathlms`:

| Path | What is in it |
| --- | --- |
| `<data>/database` | The database. Losing this loses everything. |
| `<data>/uploads` | Every file anybody uploaded. |
| `<data>/cache` | Working state the cache keeps. Losing this costs nobody anything. |
| `<backups>` | The nightly copies, a **separate setting** on purpose. |

Both are settings you can point anywhere:

    PATHLMS_DATA_DIR=/srv/pathlms
    PATHLMS_BACKUP_DIR=/mnt/backup-disk/pathlms

**Any folder works for the uploads, since 0.99.1.** The file store keeps a
small label on every file it holds. Until 0.99.1 it kept them as extended
attributes, which Unraid's `/mnt/user` share layer, a Windows drive or a Mac
folder shared into Docker Desktop, and some network shares cannot hold, and on
those the store started, reported healthy, and refused every upload. It now
keeps them in ordinary files under `<data>/uploads/labels`, beside the
objects under `<data>/uploads/objects`, and the one-shot container that runs
at every start lays that out and moves an older installation's files into
place once. The Updates card on the Settings screen still runs a check called
"Uploaded files can be written" and names the log to read if the store ever
refuses.

**Put the backups on a different disk from the data if you possibly can.** They
are a separate setting for exactly this reason: an arrangement where the backup
survives the disk holding the data is worth the five minutes it costs to set up.

They are directories on your machine rather than Docker volumes so that you can
find them, and so that `docker compose down -v`, which people reach for when
something has already gone wrong, cannot destroy your backups.

**You do not need to create or chown anything.** Docker creates the directories
on first start, and the containers take ownership of the ones they own. Do check
the ownership of the backup directory with `ls -l` before you write whatever
copies those files off the machine, because that job needs permission to read
them.

### 3. Which ports are open, and what else is using them?

**PathLMS opens two ports on your machine, and both are opened on every
installation.** Port **3001** is the plain one and port **3443** is the encrypted
one. Nothing else is opened, so the database, the cache and the object store
cannot be reached from outside at all.

Check both are free before you start. If one is taken, the failure comes from
Docker rather than from PathLMS: it refuses to start the container, and names the
port.

    ss -tlnp | grep -E ':(3001|3443)\b'

To publish either one somewhere else, put these in your settings file:

    PATHLMS_PUBLISHED_PORT=8080
    PATHLMS_PUBLISHED_TLS_PORT=8443

**Upgrading from a version before 0.89.0? Replace your compose file first.**
Older ones do not tell PathLMS which encrypted port they publish, so on a
deployment holding its own certificate the Network screen reports the number of a
port nobody uses, and changing the port moves one nobody arrives on. The new file
is attached to the release. Your settings file is untouched by this.

**Inside the container the web server always listens on 3001 and 3443, and never
moves.** That is deliberate: it means the web server never needs extra privileges
even when you publish it on port 80.

Two more settings decide which of the machine's network interfaces those ports
are opened on. Empty, which is the default, means all of them. To open them on
the machine itself only, which is what you want when a reverse proxy running
directly on this machine holds your certificate:

    PATHLMS_PUBLISHED_ADDRESS=127.0.0.1:
    PATHLMS_PUBLISHED_TLS_ADDRESS=127.0.0.1:

The trailing colon on each is required. The address and the port number are
joined together, so an address written without it runs into the port. Set only
the first and the second follows it.

**Do not set either one if your reverse proxy is itself a Docker container.**
Inside a container `127.0.0.1` means that container rather than the machine, so a
proxy running in a container can no longer reach PathLMS. Restrict the ports with
a firewall rule instead, as [Putting PathLMS behind a proxy you already
run](deploy/BEHIND-A-PROXY.md) describes.

**Changing a port after the install is an edit to the same settings file and one
command, and it has three traps in it.** Which of the two ports people actually
arrive on depends on who holds your certificate. The address people type is a
separate setting that does not move with the port, and a page that draws and then
does nothing is what you get when it is left behind. And narrowing the plain port
to one network narrows the encrypted port with it.

All three, and what to do when the system does not come back on the new port, are
on their own page: [Changing the port people arrive
on](deploy/CHANGING-THE-PORT.md). **Read it before you change a port rather than
after**, because its recovery section is written for somebody who can no longer
reach any screen.

**The Ports section of the Network tab reports these settings and does not change
them.** Nothing running inside a container can move a port on the machine: Docker
opens it when the container is created, which is why this is an edit to a file
and a command rather than a button.

### 4. What sits in front of it?

Somebody has to hold your certificate, and until somebody does, **an installation
built from a release serves unencrypted traffic.** Port 3443 is listening from
the first start, but it holds no certificate, so nothing is served on it. You
have two ways to change that, and most people should take the first.

**Put a reverse proxy in front of it**, holding your certificate and pointed at
port 3001. Caddy, nginx, Traefik, HAProxy and every cloud load balancer are all
fine. Set `PATHLMS_PUBLIC_URL` to the address people type at that proxy, which
will be the `https://` one, and **not** to the machine and port behind it. Then
set `PATHLMS_TRUST_FORWARDED_PROTO=true`, which tells PathLMS to believe the
proxy when the proxy says a visitor arrived over an encrypted connection. Without
it, every request is treated as unencrypted no matter what your proxy did.

**The exact steps, a copyable nginx block, the Caddy equivalent, how to prove it
worked, and how to stop somebody reaching around the proxy, are all on their own
page: [Putting PathLMS behind a proxy you already run](deploy/BEHIND-A-PROXY.md).**

**Or let PathLMS hold the certificate itself.** There is nothing to download and
nothing extra to run. Port **3443** is already open. That is the one browsers
connect to and the one your firewall has to allow.

**Do it in this order.** `PATHLMS_PUBLIC_URL` is the only address a browser is
answered from, so setting it to your `https://` address before a certificate
exists leaves you nowhere to sign in: the encrypted port has nothing to serve
yet, and the plain port now refuses your browser because it is a different
address.

1. Set `PATHLMS_PUBLIC_URL` to the plain address you can reach today, for example
   `http://192.0.2.10:3001`, and start the stack.
2. Sign in at that address. Under **Settings**, then **Network**, then
   **Encryption**, generate a certificate or upload one you already hold.
3. Change `PATHLMS_PUBLIC_URL` to the encrypted address, including the port
   unless it is 443, for example `https://learn.example.com:3443`.
4. Run `docker compose up -d` again, and use the new address from then on.

Until step 2 is done, port 3443 is open and every attempt to use it fails.
Nothing is served on it and no browser gets a page, so there is nothing for
anybody to click past. That is deliberate: refusing to start the whole system
would take away the screen you need in order to fix it.

**If port 3443 is already taken on that machine**, Docker will refuse to start
and say so, naming the port. Put a free one in `PATHLMS_PUBLISHED_TLS_PORT` in
your settings file and start again.

**If a reverse proxy or load balancer holds your certificate, close port 3443 to
everything but the machine itself.** Set `PATHLMS_PUBLISHED_TLS_ADDRESS` to
`127.0.0.1:`, with the trailing colon. Leave it out and it follows whatever you
set for the plain port.

That closes two holes. Anybody who can reach port 3443 directly is going around
your proxy, and can claim their connection was encrypted when it was not. And
while no certificate is in place, every connection attempt writes a line to your
log, which nothing slows down because no request is ever completed, so a stranger
can fill a log as fast as they can open connections.

Once you are signed in, tell PathLMS which of these you chose, on the Network
tab, and it stops offering to make you a certificate you already have. [After it
starts](AFTER-IT-STARTS.md) covers that screen.

---

## The general path

Everything from here is identical in every environment.

### 1. Get the files

Download the files attached to the [latest
release](https://github.com/path-lms/pathlms/releases/latest) into an empty
directory on the server. Every command on this page is run in that directory,
from a terminal on the server:

| File | What it is |
| --- | --- |
| `docker-compose.yml` | The stack. Do not edit it. |
| `pathlms.env` | Your settings. You fill this in. |
| `INSTALL.md` | Numbered steps, which the rest of this section follows. |
| `CHANGING-THE-PORT.md` | Keep it beside the others. It is how you move the port later, and its recovery section is written to be read on the day PathLMS is not answering. |

A release also attaches one short page for each environment that decides
something its own way. Take the one that matches yours, if there is one, and
ignore the rest.

### 2. Follow `INSTALL.md`

It is short and it is the authority, because it ships with the release and names
that release's exact image fingerprints. In outline it has you:

- rename `pathlms.env` to `.env`, so Compose finds it;
- run one command, in a terminal on the server, that generates every password and
  key this installation needs. It runs inside the image you are about to deploy,
  so nothing extra has to be installed on the server, and the values it prints are
  made fresh for you and written down nowhere else;
- fill in the three things it cannot generate: your address, your email address,
  and a password you choose for the first administrator;
- apply your answers to decisions 2, 3 and 4 above;
- start it with `docker compose up -d`.

**Do not copy credentials from anywhere, including from any example you find
online.** PathLMS refuses to start a fresh installation on a credential published
in its own source, on purpose, and the refusal appears only in the container log,
so it looks like a broken download.

### 3. Watch it come up

The database builds itself on the first start and that takes a minute or two:

    docker compose logs -f api

When it is ready the application reports that it is listening. If it restarts
over and over instead, the reason is in that log and it names the setting.

### 4. Prove it is actually up

Ask the application itself, not the web server:

    curl -fsS http://localhost:3001/api/health/ready

**Use `/api/health/ready` and nothing shorter.** The web server answers any path
it does not recognize with the application's own page, and it does so with a
success status, so a monitor pointed at `/health/ready` reports that everything
is fine while the application behind it is down. This is the health check address
to give a load balancer, a cloud health probe, or your own monitoring.

### 5. Sign in and secure the account

Open your address, sign in with the email address and password from step 2, then
read [After it starts](AFTER-IT-STARTS.md) before you do anything else. It is two
minutes and it prevents the one failure that has no button to undo it.

---

## Cloud virtual machine

Any of the major providers, on their general purpose instances. Everything in the
general path applies unchanged. What is different is around it.

**The instance.** Two virtual cores and 4 GB of memory is a comfortable start.
Images are published for both Intel and ARM, so an ARM instance is a legitimate
way to spend less, and the install is identical.

**Where the data lives.** Do not leave it on the instance's boot disk. Attach a
separate block volume, mount it, and point `PATHLMS_DATA_DIR` at it. Boot disks
are the part of an instance that gets replaced when you resize, rebuild or
migrate, and a database on one is a database you can lose to a routine operation.

**What backs it up.** You have two layers and you want both. Your provider's
volume snapshots protect you from losing the machine. The nightly copies PathLMS
takes protect you from losing a table, and they are the only layer you can
restore selectively from. Point `PATHLMS_BACKUP_DIR` at a path you also sync to
object storage on a schedule, because a backup that only exists on the machine it
came from is not one.

**Which ports are open.** Keep 3001 and 3443 closed to the internet in the
security group or firewall rules. Open only the load balancer's port, and let the
load balancer reach the instance on 3001 over the private network. On a single
instance with no load balancer, run a reverse proxy on the same machine, publish
the proxy on 443, and leave PathLMS where it is.

**What sits in front.** The provider's load balancer is the natural place for the
certificate, and its managed certificate service means one less thing to renew.
Give it `http://<private-ip>:3001/api/health/ready` as the health check target.

**One thing to know about load balancers here.** PathLMS runs as one copy and
cannot be scaled out, so put exactly one instance behind that load balancer and
use it for its certificate and its health check rather than for spreading load.
Two copies would break report downloads.

**Do not point autoscaling at it**, for the same reason. Anything that can bring
a second instance up while the first is running will break report downloads. If
you want a group of one purely so a failed instance is replaced, that can be made
to work, and you own the part where the data volume is reattached to whatever
comes back. Nothing about that is automatic.

---

## Bare metal, or a virtual machine you administer

Your own hardware, or a virtual machine on your own hypervisor. The general path
is the whole install. What is different is that every layer around it is yours.

**Where the data lives.** You have the freest hand here, and nothing above you
will catch a bad choice. Put `PATHLMS_DATA_DIR` on redundant storage, whether
that is a RAID array, a mirrored pair or a filesystem that does it for you. Put
`PATHLMS_BACKUP_DIR` on a different physical disk, and copy those files to a
different machine on a schedule.

**Snapshots are not a substitute for the nightly copies.** A filesystem snapshot
of a running database captures it mid-write. It will usually restore and it is
not something to rely on. The nightly copy is taken by the database itself, which
is what makes it complete.

**Which ports are open.** Nothing opens a firewall for you here, so decide
deliberately which of the two ports the network may reach, and check that nothing
already holds either:

    ss -tlnp | grep -E ':(3001|3443)\b'

If this machine is reachable from outside your network, put a firewall rule in
front rather than relying on the ports being obscure.

**What sits in front.** A reverse proxy on the same machine is the simplest thing
that works. Caddy will obtain and renew a certificate for you with almost no
configuration; nginx or Traefik will too with a little more. Point it at
`127.0.0.1:3001`.

When the proxy runs directly on this machine, bind PathLMS to the loopback
interface only, so nothing on the network can reach around the proxy. That is a
**second setting**, alongside the port rather than replacing it:

    PATHLMS_PUBLISHED_ADDRESS=127.0.0.1:

**The trailing colon is required and is not a typo.** Leave
`PATHLMS_PUBLISHED_PORT` as it is; the two are joined together, so an address
written without the colon runs into the port number.

**Skip this if the proxy runs in a Docker container**, even on this same machine.
`127.0.0.1` inside a container is that container, so the proxy would lose PathLMS.
Use a firewall rule on port 3001 instead.

Once you are signed in there is a screen for this as well: the **Ports** section
on the **Network** tab offers **Every network on this machine** or **One address
only**, and it reads both settings, so it can tell you which you actually have.

**Start it on boot.** The containers are set to restart unless they were stopped
on purpose, so a reboot brings PathLMS back on its own as long as the Docker
service itself starts on boot. Check that once, on a real reboot, rather than
assuming it.

---

## A home or small-office server appliance

An appliance brings its own application manager, its own idea of where shares
live, and its own web interface already holding a port. Each of those changes a
step the general path takes for granted.

**Unraid has a guide of its own: [Running PathLMS on Unraid](deploy/UNRAID.md).**

For any other appliance, the general path is still the install. What you will
have to work out for yourself is how that appliance wants a stack of several
containers run, because most of them are built around adding one container at a
time. If it can run a Docker Compose file, use it, and everything above applies
unchanged.

---

## Other ways of running containers

**Docker Compose is the supported way to run PathLMS**, and it is the only shape
the containers are tested and released in.

Podman with its Compose support is close enough that people succeed with it, and
it is not something anybody here has verified. Kubernetes is a poor fit and not
worth the effort: PathLMS runs as exactly one copy of one process, so almost
everything Kubernetes is for does not apply, and the parts that would matter
(persistent volumes, one replica, no rolling update) are more work to express
there than in the file you already have.

---

## Common problems

**It will not stay up, and the log mentions a published password.** A fresh
installation refuses to run on credentials published in the project's own source.
Generate your own, as `INSTALL.md` step 3 does.

**The sign-in page appears and nothing on it works.** `PATHLMS_PUBLIC_URL` does
not match the address you typed in the browser. Fix it and run
`docker compose up -d` again. If something sits in front holding your
certificate, this setting is the address at that proxy, not the machine and port
behind it.

**Uploaded pictures do not appear, but everything else works.** Same cause,
different symptom: file links are signed against the address in
`PATHLMS_PUBLIC_URL`, so a wrong one produces links your object store refuses.

**A container will not start and the message is about a port.** Something else on
the machine already holds it. Change `PATHLMS_PUBLISHED_PORT`, or stop the other
thing.

**Your monitoring says it is healthy and it is not.** You are checking a path the
web server answers on its own. Use `/api/health/ready`.

**Anything else.** `docker compose logs api` is the answer almost every time.
Read the **first** refusal rather than the last line: PathLMS names the setting
that is wrong, and the lines after it are consequences.
