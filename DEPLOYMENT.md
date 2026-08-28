# Deploying PathLMS

PathLMS runs anywhere Docker Compose runs. That is the whole story for most
installations, and this page is arranged to match: **the general path below is
the main road, and each environment afterwards is a short detour off it.** If you
read only the general path and your environment is not listed, you have read
everything you need.

Nothing here needs the source code, `git`, Node.js, or a build step on the
server.

---

## Before you start: four decisions

These four have consequences you live with. Everything else can be changed from
the Settings screen once you are in.

### 1. What address will people type?

This is the single most consequential setting, and getting it wrong produces the
most confusing failure PathLMS has. Four things derive from it: which pages a
browser is answered from, the links inside password recovery mail, the address
your uploaded files are signed against, and where company sign-in returns people
to.

If the address is wrong, the sign-in page renders perfectly and then does
nothing, and nothing on that screen points at the cause.

Write it with the scheme and no path: `https://learn.example.com`.

### 2. Where will the data live?

Everything PathLMS stores goes under one directory, which defaults to
`/var/lib/pathlms`:

| Path | What is in it |
| --- | --- |
| `<data>/database` | The database. Losing this loses everything. |
| `<data>/uploads` | Every file anybody uploaded. |
| `<backups>` | The nightly dumps, a **separate setting** on purpose. |

Both are settings you can point anywhere:

    PATHLMS_DATA_DIR=/srv/pathlms
    PATHLMS_BACKUP_DIR=/mnt/backup-disk/pathlms

**Put the backups on a different disk from the data if you possibly can.** They
are a separate setting for exactly this reason: an arrangement where the backup
survives the disk holding the data is worth the five minutes it costs to set up.

They are directories on your machine rather than Docker volumes so that you can
find them with `ls`, and so that `docker compose down -v`, which people reach for
when something has already gone wrong, cannot destroy your backups.

**You do not need to create or chown anything.** Docker creates the directories
on first start, and the containers take ownership of the ones they own. Do check
the ownership of the backup directory with `ls -l` before you write whatever
copies those files off the machine, because that job needs permission to read
them.

### 3. Which port is open, and what else is using it?

PathLMS publishes exactly one port on the host, and it defaults to **3001**.
Nothing else is exposed: the database, the cache and the object store sit on
networks that cannot reach the internet and cannot be reached from it.

To publish a different port:

    PATHLMS_PUBLISHED_PORT=8080

**The container always listens on 3001 inside itself and never moves.** That is
deliberate: it means the web server never needs extra privileges even when you
publish it on port 80.

A second setting decides which of the machine's networks that port is opened on.
Empty, which is the default, means all of them. To open it on one address only,
which matters when something else in front holds your certificate:

    PATHLMS_PUBLISHED_ADDRESS=127.0.0.1:

The trailing colon is required. The two settings are joined together, so an
address without it runs into the port number.

Check the port is free before you start, because the failure otherwise arrives
from Docker rather than from PathLMS:

    ss -tlnp | grep -E ':(3001|8080)\b'

### 4. What sits in front of it?

**An installation built from a release serves unencrypted HTTP unless you add
one file.** Out of the box it holds no certificate and listens on no encrypted
port. You have two ways to change that, and most people should take the first.

**Put something in front of it.** A reverse proxy or a load balancer holding
your certificate, pointed at the PathLMS port.
Caddy, nginx, Traefik, HAProxy, or your cloud provider's load balancer all work.
Nothing in the PathLMS settings file changes when you do this. Caddy, nginx,
Traefik, HAProxy and every cloud load balancer are all fine.

**Or let PathLMS hold the certificate itself.** A release attaches a short file
called `docker-compose.encryption.yml` for this. Put it beside your compose
file, make sure your address begins with https, and start everything with both
files:

    docker compose -f docker-compose.yml -f docker-compose.encryption.yml up -d

Then sign in and, under Settings, Network, Encryption, either generate a
certificate or upload one you already hold. Until you have added one, the
encrypted port is open and every attempt to use it fails, which is deliberate:
refusing to start the whole system would take away the screen you need in order
to fix it.

Then set `PATHLMS_PUBLIC_URL` to the address people type at that proxy, which
will be the `https://` one, and **not** to the machine and port behind it.

Once you are signed in, the Network tab records which of these you chose, so the
rest of the product stops guessing. [After it starts](AFTER-IT-STARTS.md) covers
that screen.

---

## The general path

Everything from here is identical in every environment.

### 1. Get the files

Download the files attached to the [latest
release](https://github.com/path-lms/pathlms/releases/latest) into an empty
directory on the server:

| File | What it is |
| --- | --- |
| `docker-compose.yml` | The stack. Do not edit it. |
| `pathlms.env` | Your settings. You fill this in. |
| `INSTALL.md` | Numbered steps, which the rest of this section follows. |

A release also attaches one short page for each environment that decides
something its own way. Take the one that matches yours, if there is one, and
ignore the rest.

### 2. Follow `INSTALL.md`

It is short and it is the authority, because it ships with the release and names
that release's exact image fingerprints. In outline it has you:

- rename `pathlms.env` to `.env`, so Compose finds it;
- run one command that generates every password and key this installation needs.
  That command runs inside the image you are about to deploy, so nothing extra
  has to be installed on the server, and the values it prints are made fresh for
  you and written down nowhere else;
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
volume snapshots protect you from losing the machine. The nightly dumps PathLMS
takes protect you from losing a table, and they are the only layer you can
restore selectively from. Point `PATHLMS_BACKUP_DIR` at a path you also sync to
object storage on a schedule, because a backup that only exists on the machine it
came from is not one.

**Which port is open.** Keep 3001 closed to the internet in the security group or
firewall rules. Open only the load balancer's port, and let the load balancer
reach the instance on 3001 over the private network. On a single instance with no
load balancer, run a reverse proxy on the same machine, publish the proxy on 443,
and leave PathLMS bound where it is.

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

**Where the data lives.** You have the freest hand here and the most rope. Put
`PATHLMS_DATA_DIR` on redundant storage, whether that is a RAID array, a mirrored
pair or a filesystem that does it for you. Put `PATHLMS_BACKUP_DIR` on a
different physical disk, and copy those files to a different machine on a
schedule.

**Snapshots are not a substitute for the dumps.** A filesystem snapshot of a
running database captures it mid-write. It will usually restore and it is not
something to rely on. The nightly dump is taken by the database itself and is
consistent by construction.

**Which port is open.** Nothing publishes itself here, so open the one port
deliberately and check nothing already holds it:

    ss -tlnp | grep ':3001'

If this machine is reachable from outside your network, put a firewall rule in
front rather than relying on the port being obscure.

**What sits in front.** A reverse proxy on the same machine is the simplest thing
that works. Caddy will obtain and renew a certificate for you with almost no
configuration; nginx or Traefik will too with a little more. Point it at
`127.0.0.1:3001`.

When the proxy is on the same machine, bind PathLMS to the loopback interface
only, so nothing on the network can reach around the proxy. That is a **second
setting**, alongside the port rather than replacing it:

    PATHLMS_PUBLISHED_ADDRESS=127.0.0.1:

**The trailing colon is required and is not a typo.** Leave
`PATHLMS_PUBLISHED_PORT` as it is; the two are joined together, so an address
written without the colon runs into the port number.

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
live, and its own web interface already holding a port. Those change the shape of
the install in ways the general path does not cover.

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
certificate, this setting is the address at that proxy, not the machine behind
it.

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
