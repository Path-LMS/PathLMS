# Deploying PathLMS

This guide gets PathLMS running on a machine you control, in about fifteen
minutes. Most of that is the machine downloading things, not you working.

You do not need to be a developer. There is no source code to fetch, no `git`,
no Node.js, and no build step on the server. If you can open a terminal, edit a
text file, and run one command, you can do this.

The install is the same on every machine. Do the five steps below and you have a
working system people can sign in to. The rest of this page is detail you reach
for only when you want it: how to make it secure, where your data lives, and the
few things that differ on a cloud machine, your own hardware, or a home
appliance.

---

## Before you begin

You need a machine with **Docker** and **Docker Compose** installed. Two CPU
cores and 4 GB of memory is a comfortable start. Both Intel and ARM machines
work, so a cheaper ARM machine is fine.

Have three things ready. You fill these in during the install:

1. **The web address people will type** to reach PathLMS, for example
   `https://learn.example.com`. If you do not have one yet, you can start
   with the machine's own address and change it later.
2. **A contact email address** for the system.
3. **A password for the first administrator**, which is you. Choose a strong one
   and keep it in a password manager.

---

## Install it in five steps

Run every command in a terminal on the server, in one empty directory you make
for PathLMS.

### 1. Get the files

Download the files attached to the [latest
release](https://github.com/Path-LMS/PathLMS/releases/latest) into that
directory. You get a small set of files:

| File | What it is |
| --- | --- |
| `docker-compose.yml` | The system itself. You do not edit this. |
| `pathlms.env` | Your settings. You fill this in. |
| `INSTALL.md` | The exact steps for this release. It is the authority. |
| `CHANGING-THE-PORT.md` | Keep it. You need it only if you move a port later. |

### 2. Make your settings file

Rename `pathlms.env` to `.env` so Docker can find it:

    mv pathlms.env .env

Then run the one command `INSTALL.md` gives you to fill in every password and
secret key the system needs. It runs inside the software you are about to
install, so nothing extra has to be on the server, and it writes fresh secrets
that exist nowhere else. `INSTALL.md` has the exact command, because it names
this release's own version.

**Do not copy passwords or keys from anywhere, including examples online.**
PathLMS refuses to start on a password that has been published in its own code,
on purpose. The refusal shows up only in the log, so a copied password looks
like a broken download.

### 3. Fill in the three things it cannot guess

Open `.env` in a text editor and set the three things you got ready:

    PATHLMS_PUBLIC_URL=http://your-server-address:3001
    INITIAL_ADMIN_EMAIL=you@example.com
    INITIAL_ADMIN_PASSWORD=the-strong-password-you-chose

Use a plain `http://` address you can reach today, such as the server's own
address. You make it secure and switch to `https://` afterwards, and the
[section on that](#making-it-secure-who-holds-your-certificate) below is short.

### 4. Start it

    docker compose up -d

The first start builds the database, which takes a minute or two. Watch it come
up:

    docker compose logs -f api

When it is ready it says it is listening. If it restarts over and over instead,
the reason is in that log, and it names the setting that is wrong.

### 5. Sign in and secure your account

Open your address in a browser and sign in with the email and password from step
3. Then read **[After it starts](AFTER-IT-STARTS.md)** before you do anything
else. It takes two minutes and saves you an unpleasant afternoon.

Two things on that page matter on day one: set up a code from an app on your
phone, because administrator screens ask for one, and note the rescue command,
which is how you get back in if you are ever locked out.

That is the whole install. Everything below is here when you want it.

---

## Making it secure: who holds your certificate

**Out of the box, PathLMS serves unencrypted traffic.** Somebody has to hold the
certificate that turns on the padlock in the browser. You have two ways to do
that, and most people should take the first.

**Put a reverse proxy in front of it.** This is a small piece of software, such
as Caddy, nginx or Traefik, or a cloud load balancer, that holds your
certificate and passes traffic through to PathLMS. It is the standard way to run
any web application, and if you already run other sites you probably have one.
The exact steps, a copyable configuration for Caddy and nginx, and how to prove
it worked, are on their own page: **[Putting PathLMS behind a proxy you already
run](deploy/BEHIND-A-PROXY.md)**.

**Or let PathLMS hold the certificate itself.** There is nothing extra to
install. Once you are signed in, go to **Settings**, then **Network**, then
**Encryption**, and generate a certificate or upload one you already have.

**Whichever you choose, do it in this order**, because the address is the one
thing a browser is answered from:

1. Install with the plain `http://` address, as in step 3 above, and sign in.
2. Put your certificate in place, by the proxy or in the Encryption screen.
3. Only then change `PATHLMS_PUBLIC_URL` to your `https://` address and run
   `docker compose up -d` again.

Changing the address to `https://` before the certificate exists leaves you
nowhere to sign in, so this order matters.

---

## The rest of this page

Below is detail, not steps. Read a section only when it is the thing you are
trying to do.

- [The address people type](#the-address-people-type), the one setting
  everything depends on.
- [Where your data lives](#where-your-data-lives), and how to keep it safe.
- [The ports PathLMS opens](#the-ports-pathlms-opens).
- Running it in a particular place: [a cloud machine](#a-cloud-machine),
  [your own hardware](#your-own-hardware), or [a home or office
  appliance](#a-home-or-office-appliance).
- [Other ways of running containers](#other-ways-of-running-containers).
- [Common problems](#common-problems).

---

## The address people type

This is `PATHLMS_PUBLIC_URL`, and it is the setting to get right, because getting
it wrong fails in a way that does not look like an address problem: the sign-in
page appears perfectly and then does nothing, and nothing on the screen points
at the cause. It is the most confusing failure PathLMS has.

Write it with the scheme and no path, for example `https://learn.example.com`.
Four things are built from it: which pages a browser is answered from, the links
inside password recovery mail, the address your uploaded files are signed
against, and where company sign-in returns people to.

**If a reverse proxy or load balancer holds your certificate, this is the
address at that proxy**, the one people type, not the machine and port behind it.

---

## Where your data lives

Everything PathLMS stores goes under one directory, which defaults to
`/var/lib/pathlms`. You can point it anywhere with `PATHLMS_DATA_DIR`:

| Path | What is in it |
| --- | --- |
| `<data>/database` | The database. Losing this loses everything. |
| `<data>/uploads` | Every file anybody uploaded. |
| `<data>/cache` | Working state. Losing this costs nobody anything. |
| `<backups>` | The nightly copies, a **separate setting** on purpose. |

    PATHLMS_DATA_DIR=/srv/pathlms
    PATHLMS_BACKUP_DIR=/mnt/backup-disk/pathlms

**Put the backups on a different disk from the data if you possibly can.** They
are a separate setting for exactly this reason: a backup that survives the disk
holding your data is worth the five minutes it costs to set up.

These are ordinary directories on your machine, not hidden Docker volumes, so you
can find them with `ls`, and so `docker compose down -v`, which people reach for
when something has already gone wrong, cannot destroy your backups.

**You do not need to create or change the ownership of anything.** Docker makes
the directories on first start and the containers take ownership of their own.
Do check who owns the backup directory with `ls -l` before you set up whatever
copies those files off the machine, because that job needs permission to read
them.

**Any folder works for the uploads.** Older versions kept a hidden label on each
uploaded file that some storage could not hold, and on those the uploads
silently failed. Since version 0.100.0 the labels are kept in ordinary files
instead, so a network share, a Windows drive or a Mac folder all work.

---

## The ports PathLMS opens

PathLMS opens two ports, on every installation. Port **3001** is the plain one
and port **3443** is the encrypted one. Nothing else is opened, so the database,
the cache and the file store cannot be reached from outside at all.

Check both are free before you start. If one is taken, Docker refuses to start
and names the port:

    ss -tlnp | grep -E ':(3001|3443)\b'

To publish either one somewhere else, put these in your settings file:

    PATHLMS_PUBLISHED_PORT=8080
    PATHLMS_PUBLISHED_TLS_PORT=8443

**Inside the container the web server always uses 3001 and 3443 and never
moves**, so it never needs special privileges even when you publish it on port
80.

**The Ports section of the Network screen reports these settings, it does not
change them.** Nothing inside a container can move a port on the machine: Docker
opens it when the container is made. So changing a port is an edit to your
settings file and one command, never a button.

**Changing a port later has traps in it**, and its recovery section is written
for somebody who can no longer reach any screen. Read [Changing the port people
arrive on](deploy/CHANGING-THE-PORT.md) before you change one, not after.

**If a reverse proxy holds your certificate, close port 3443 to everything but
the machine itself.** Set `PATHLMS_PUBLISHED_TLS_ADDRESS` to `127.0.0.1:`, with
the trailing colon. This stops anybody reaching around your proxy, and stops a
stranger filling your log by opening connections to a port with no certificate on
it.

**To bind the plain port to the machine itself only**, which is what you want
when a reverse proxy runs directly on this machine, set:

    PATHLMS_PUBLISHED_ADDRESS=127.0.0.1:

The trailing colon is required: the address and the port are joined together, so
an address without it runs into the port. **Do not set this
if your reverse proxy is itself a Docker container**, because inside a container
`127.0.0.1` means that container, not the machine, and the proxy would lose
PathLMS. Restrict the port with a firewall rule instead.

---

## A cloud machine

Any of the major providers, on their general purpose instances. The five install
steps apply unchanged. What is different is around them.

**The instance.** Two virtual cores and 4 GB of memory is a comfortable start.
ARM instances work and cost less, and the install is identical.

**Where the data lives.** Do not leave it on the instance's boot disk. Attach a
separate volume, mount it, and point `PATHLMS_DATA_DIR` at it. Boot disks are the
part that gets replaced when you resize, rebuild or migrate, and a database on
one is a database you can lose to a routine operation.

**What backs it up.** You want two layers. Your provider's volume snapshots
protect you from losing the machine. The nightly copies PathLMS takes protect you
from losing a single table, and they are the only layer you can restore
selectively from. Point `PATHLMS_BACKUP_DIR` at a path you also sync to object
storage on a schedule, because a backup that only exists on the machine it came
from is not one.

**What sits in front.** The provider's load balancer is the natural place for the
certificate, and its managed certificate service means one less thing to renew.
Give it `http://<private-ip>:3001/api/health/ready` as the health check target,
and keep 3001 and 3443 closed to the internet in the security group.

**Run exactly one instance.** PathLMS runs as one copy and cannot be spread
across several, so put one instance behind the load balancer and use it for the
certificate and health check, not for sharing load. **Do not point autoscaling at
it**: a second instance coming up while the first runs will break report
downloads.

---

## Your own hardware

Your own machine, or a virtual machine on your own hypervisor. The five steps are
the whole install. What is different is that every layer around it is yours.

**Where the data lives.** You have the freest hand here, and nothing will catch a
bad choice for you. Put `PATHLMS_DATA_DIR` on redundant storage, whether that is a
RAID array, a mirrored pair or a filesystem that does it for you. Put
`PATHLMS_BACKUP_DIR` on a different physical disk, and copy those files to a
different machine on a schedule.

**Snapshots are not a substitute for the nightly copies.** A filesystem snapshot
of a running database catches it mid-write. It will usually restore, and it is
not something to rely on. The nightly copy is taken by the database itself, which
is what makes it complete.

**What sits in front.** A reverse proxy on the same machine is the simplest thing
that works. Caddy will get and renew a certificate for you with almost no setup.
Point it at `127.0.0.1:3001`, and bind PathLMS to the machine itself with
`PATHLMS_PUBLISHED_ADDRESS=127.0.0.1:` so nothing on the network can reach around
the proxy. Skip that setting if the proxy runs in a Docker container, and use a
firewall rule instead.

**Start it on boot.** The containers restart unless you stopped them on purpose,
so a reboot brings PathLMS back on its own as long as Docker itself starts on
boot. Check that once, on a real reboot, rather than assuming it.

---

## A home or office appliance

An appliance brings its own app manager, its own idea of where files live, and
its own web interface already holding a port. Each of those changes a step the
general install takes for granted.

**Unraid has a guide of its own: [Running PathLMS on Unraid](deploy/UNRAID.md).**

For any other appliance, the five steps are still the install. What you work out
for yourself is how that appliance wants a stack of several containers run, since
most are built around adding one container at a time. If it can run a Docker
Compose file, use that, and everything here applies unchanged.

---

## Other ways of running containers

**Docker Compose is the supported way to run PathLMS**, and the only way the
containers are tested and released.

Podman with its Compose support is close enough that people succeed with it,
though nobody here has verified it. Kubernetes is a poor fit and not worth the
effort: PathLMS runs as exactly one copy of one process, so almost everything
Kubernetes is for does not apply.

---

## Common problems

**It will not stay up, and the log mentions a published password.** A fresh
install refuses to run on a password published in the project's own code.
Generate your own, as step 2 does.

**The sign-in page appears and nothing on it works.** `PATHLMS_PUBLIC_URL` does
not match the address you typed in the browser. Fix it and run `docker compose up
-d` again. If something sits in front holding your certificate, this setting is
the address at that proxy, not the machine and port behind it.

**Uploaded pictures do not appear, but everything else works.** Same cause,
different symptom: file links are signed against the address in
`PATHLMS_PUBLIC_URL`, so a wrong one produces links your file store refuses.

**A container will not start and the message is about a port.** Something else on
the machine already holds it. Change `PATHLMS_PUBLISHED_PORT`, or stop the other
thing.

**Your monitoring says it is healthy and it is not.** You are checking a path the
web server answers on its own. Point it at `/api/health/ready` and nothing
shorter, because the web server answers any other path with a success page even
when the application behind it is down.

**Anything else.** `docker compose logs api` is the answer almost every time.
Read the **first** refusal rather than the last line: PathLMS names the setting
that is wrong, and the lines after it are just consequences.
