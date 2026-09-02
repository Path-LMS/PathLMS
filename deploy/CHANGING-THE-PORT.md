# Changing the port people arrive on

You edit one line in your settings file and create the containers again. There is
no screen that does it, and there cannot be: Docker opens a host port when a
container is created, and nothing running inside a container can move it.

The whole change is two or three lines and one command. Most of this page is
about the three things that are easy to get wrong around it, and about what to do
if the system does not come back.

**Read section 1 first even if you are in a hurry.** This deployment opens two
ports, and half of the trouble people have here is moving the one nobody arrives
on.

---

## 1. There are two ports, and you need to know which one is yours

PathLMS opens two ports on your machine, on every installation, whether or not
you use both.

| Setting | Default | What it is |
| --- | --- | --- |
| `PATHLMS_PUBLISHED_PORT` | `3001` | The plain port. Unencrypted HTTP. |
| `PATHLMS_PUBLISHED_TLS_PORT` | `3443` | The encrypted port. HTTPS, and it serves nothing until PathLMS holds a certificate. |

**One test tells you which one people arrive on.** When somebody types your
address into a browser, where does the padlock come from?

- **From PathLMS itself.** You generated or uploaded a certificate under
  **Settings**, then **Network**, then **Encryption**. Browsers connect to the
  encrypted port, so **`PATHLMS_PUBLISHED_TLS_PORT` is the one to change**, and
  it is the one your firewall has to allow.
- **From something else in front**, such as nginx, Caddy, Traefik, HAProxy, a
  cloud load balancer or an appliance's built-in reverse proxy. Your visitors
  arrive at that thing, not here. **`PATHLMS_PUBLISHED_PORT` is the one to
  change**, because that is the port your proxy talks to, and the address people
  type does not change at all. See [Putting PathLMS behind a proxy you already
  run](BEHIND-A-PROXY.md).
- **Nothing is encrypted anywhere.** Reasonable only on a closed network nobody
  outside can reach. People arrive on the plain port, so
  **`PATHLMS_PUBLISHED_PORT` is the one to change**.

**Inside the container the web server always listens on 3001 and 3443, and those
never move.** That is what keeps the container unprivileged even when you publish
it on port 80, and it is why you never edit `docker-compose.yml` for this.

**The Ports section of the Network tab reads these settings and reports which
port it was told about.** It is a report, not a control. It also cannot tell you
what Docker actually opened, because it reads the value it was handed at startup
rather than the socket. Section 6 has the command that reads the socket.

---

## 2. Changing the port

Every command on this page is run in the directory holding `docker-compose.yml`.
If you are on a storage appliance, read section 7 first, because both the file
and the command are somewhere else.

1. Open your settings file. It is the file called `.env`, sitting beside
   `docker-compose.yml`. It is not `pathlms.env`; that is only what the release
   attaches, and it is renamed during the install.
2. Find the line for the port you are moving, or add it if it is not there. Set
   it to the port you want. Use one of these, not both, unless you genuinely mean
   to move both:

       PATHLMS_PUBLISHED_PORT=8080
       PATHLMS_PUBLISHED_TLS_PORT=8443

3. **If you just changed the port people type, change the address to match**, on
   the `PATHLMS_PUBLIC_URL` line in the same file. Section 3 is about what
   happens when you skip this, and it is the most confusing failure this product
   has.

       PATHLMS_PUBLIC_URL=http://192.0.2.10:8080

   If something in front holds your certificate, leave `PATHLMS_PUBLIC_URL`
   alone: it names the address at the proxy, which has not moved. Change where
   your proxy sends requests instead.
4. Save the file.
5. Create the containers again:

       docker compose up -d

   **A restart is not enough.** A container keeps the settings it was created
   with, so `docker compose restart`, and an appliance's Stop and Start controls,
   apply nothing you have just changed. `docker compose up -d` creates the
   container afresh.
6. Check that Docker actually opened the port:

       docker port pathlms-web

   Right answer names both ports, with the new one among them:

       3001/tcp -> 0.0.0.0:8080
       3443/tcp -> 0.0.0.0:3443

7. Open the new address in a browser and sign in.

### If step 5 fails instead of starting

Docker refuses loudly here rather than starting something broken, which is the
good case. Nothing has changed, and the deployment you had is still the
deployment you have. Two refusals are common.

**"port is already allocated" or "address already in use."** Something else on
this machine holds that port. Find out what, then pick a port nothing holds:

    ss -tlnp | grep -E ':(8080|8443)\b'

**It can also be PathLMS itself.** Both ports are opened on every installation,
so asking for `PATHLMS_PUBLISHED_PORT=3443` collides with the encrypted port this
same deployment already publishes.

**"permission denied" on a port below 1024.** Ports below 1024 need
administrator rights on the machine, and a rootless Docker does not have them.
Use a port above 1024 and let something in front answer on 443, or run Docker
with the rights it needs. This has nothing to do with the container, which stays
unprivileged whatever you publish it on.

---

## 3. The address people type is a third thing, and moving a port does not move it

`PATHLMS_PUBLIC_URL` is the address people type. It is a separate setting from
the two ports, and nothing updates it for you.

Four things follow it, and all four break together when it is wrong:

- which pages a browser is answered from at all;
- the links inside password recovery mail;
- the address every uploaded file's link is signed against;
- where company sign-in returns people to.

**What a wrong one looks like is the reason this section exists.** The sign-in
page draws perfectly and then does nothing. Nothing on that screen names a
setting, no error mentions an address, and the request succeeds from the command
line while failing in every browser. Uploaded pictures stop appearing for the
same reason, because their links are signed against the old address and your
object store refuses the signature.

**So move it in the same edit.** If people now type `http://192.0.2.10:8080`,
then that is what `PATHLMS_PUBLIC_URL` says, including the port unless it is the
default for the scheme.

**The address in your settings file is permanently allowed**, whatever anybody
later saves on a screen. That is deliberate, and it is the thing that makes this
recoverable: putting the old port and the old address back in `.env` is always
enough to reach the product again.

**To serve two addresses at once**, add the second rather than replacing the
list:

    PATHLMS_ADDITIONAL_ORIGINS=https://also-here.example.com

---

## 4. Which network interfaces the ports are opened on

Two more settings decide that, and they are separate from the port numbers.

| Setting | Empty, which is the default | Set to `127.0.0.1:` |
| --- | --- | --- |
| `PATHLMS_PUBLISHED_ADDRESS` | The plain port is open on every network on this machine. | The plain port is reachable only from this machine. |
| `PATHLMS_PUBLISHED_TLS_ADDRESS` | Follows the setting above. | The encrypted port is reachable only from this machine. |

**The trailing colon is required and is not a typo.** The value is joined to the
port number, so `127.0.0.1` without the colon runs into it and produces something
Docker cannot parse.

**When narrowing is right.** A reverse proxy running directly on this machine
holds your certificate. Everything that should reach PathLMS is already on this
machine, so closing the ports to the network stops anybody going around the proxy
and reaching PathLMS unencrypted. If you have also told PathLMS to believe a
forwarded visitor address, closing these ports is not optional, and
[BEHIND-A-PROXY.md](BEHIND-A-PROXY.md) explains why.

**When narrowing is a disaster.** Set either one on a deployment people reach
across a network and every browser except one on the machine itself stops
reaching PathLMS. Nothing warns you, because from the machine's own point of view
it is working perfectly. The two cases that catch people out:

- **A deployment people use from their own computers, with nothing in front.**
  Leave both empty.
- **A reverse proxy that is itself a Docker container**, even one on this same
  machine. Inside a container, `127.0.0.1` means that container and not the
  machine, so your proxy loses PathLMS the moment you set this. Use a firewall
  rule allowing only the proxy's address instead.

---

## 5. Narrowing one port can silently narrow both

**`PATHLMS_PUBLISHED_TLS_ADDRESS` follows `PATHLMS_PUBLISHED_ADDRESS` when it has
no value of its own.** That is usually what somebody wants, because somebody
narrowing one port usually meant both. It surprises people in one direction.

**Setting `PATHLMS_PUBLISHED_ADDRESS=127.0.0.1:` closes the encrypted port too**,
unless you have separately given `PATHLMS_PUBLISHED_TLS_ADDRESS` a value. On a
deployment where PathLMS holds its own certificate, that closes the port your
visitors actually arrive on, and the plain port they do not use is the one you
were thinking about.

**You cannot undo the fallback with an empty value.** Writing
`PATHLMS_PUBLISHED_TLS_ADDRESS=` does not mean "every network"; an empty value
falls back exactly as an absent one does. To have the encrypted port on every
network while the plain port is on this machine only, clear
`PATHLMS_PUBLISHED_ADDRESS` and narrow nothing, or put a firewall rule on the
plain port instead.

---

## 6. If it does not come back on the new port

**Read this before you change a port, not after.** It is written for somebody who
can no longer reach the screen, so it names no screen. This page ships with the
release and sits in the same directory as your compose file, so it is here on
your machine whether or not PathLMS is running.

### The one-line answer

Put the old value back in `.env` and create the containers again, in the
directory holding `docker-compose.yml`:

    docker compose up -d

The old port is not lost. It is the value you edited over, and if you left the
line commented out rather than deleting it, it is a few lines above.

### Working out which of the four things went wrong

Take these in order. The first two need nothing but this machine.

**One. Did Docker open the port at all?**

    docker port pathlms-web

Right answer names your new port. If the command prints nothing, or the
containers are not there, the stack did not start:

    docker compose up -d

and read what it prints. A port already in use and a port below 1024 both refuse
here, and both are covered at the end of section 2.

**Two. Does the deployment answer on this machine?**

    curl -fsS http://localhost:8080/api/health/ready

Right answer says it is ready. Use `/api/health/ready` and nothing shorter: the
web server answers unrecognized paths with the application's own page and a
success status, so a shorter path reports health it has not checked.

If this answers and a browser elsewhere does not, the port is open and something
between you and this machine is dropping it. That is a firewall, a router, a
security group or an appliance's own rules. **Nothing in PathLMS and nothing on
this page can see that**, and no check here will ever tell you about it.

**Three. Does the page draw and then do nothing?**

That is not a port problem. The port is fine and the address is wrong, which is
section 3. Ask the application what address it thinks it has:

    docker compose logs api | grep "Public address"

Right answer names the address people now type. If it names the old address,
`PATHLMS_PUBLIC_URL` did not move with the port, or the containers were restarted
rather than created again.

**Four. Did your edit reach the containers at all?**

If nothing you wrote appears anywhere in that log line, the settings file is not
the one Compose read. Check it is named `.env` exactly, and that it sits in the
same directory as `docker-compose.yml`:

    ls -a | grep -E '^\.env$|docker-compose'

### Putting it back by hand

1. Open `.env` in the directory holding `docker-compose.yml`.
2. Set `PATHLMS_PUBLISHED_PORT` back to `3001`, or to whatever it was.
3. Set `PATHLMS_PUBLIC_URL` back to the address that went with it.
4. Save the file.
5. Run `docker compose up -d`.
6. Open the old address in a browser.

**If you cannot get a terminal on this machine at all**, and it is an appliance,
section 7 gives the browser route to the same file and the same command.

---

## 7. On a storage appliance

Unraid, Synology, QNAP and TrueNAS all run a stack like this one through a
Compose feature of their own, and every one of those features offers somewhere in
a browser to edit the stack's settings and to deploy it again. You do not need a
terminal for a port change on any of them.

**Edit the stack's settings file, not a per-container variable list.** All four
port settings are read by Compose while it reads `docker-compose.yml`, before any
container exists, so they decide which port is opened at all. A value added to a
container afterwards decides none of them, and the change appears to have been
saved while nothing has moved.

**Use the appliance's Deploy or Build control, and not its Start control.** This
is ordinary Compose behavior rather than anything about your appliance: a
container keeps the settings it was created with, so starting a stopped one
applies nothing. On some interfaces the deploy control is called Build, and an
existing stack has to be cleaned first so the old containers are removed.

**Unraid has the exact route, including where the file is on disk**: [Running
PathLMS on Unraid](UNRAID.md).

**On any other appliance, check one thing.** The stack is handed a file named
`.env`, by that exact name, beside `docker-compose.yml`. Some interfaces keep
settings in a list of their own and write them out under a different name. If
yours does, the stack does not start and Compose names the file it could not
find, so it is a clean failure rather than a quiet one. **Nobody here has run
PathLMS on any appliance other than Unraid.**

**If you pasted into a browser editor and the stack now behaves oddly**, check
for invisible characters. A browser editor can add one to the end of every line:

    tr -dc '\r' < .env | wc -c

Right answer is `0`. Anything else, and this removes them:

    sed -i 's/\r$//' .env

---

## 8. What this does not do, said plainly

- **It does not tell you whether anybody else can reach the new port.** Every
  check on this page runs on the machine itself, or asks the application what it
  was told. A firewall, a router or an appliance's own rules can drop the port
  from everywhere else while all of them pass. Test from a computer somebody
  actually uses.
- **It does not put the old port back on its own.** Nothing here has a timer. If
  you change a port and cannot reach the result, section 6 is what you do, and it
  needs somebody with access to the machine or to the appliance's interface.
  Before you change a port, make sure you have one of those.
- **It does not move your proxy, your firewall or your certificate.** Those are
  all outside PathLMS, and a port change here leaves each of them pointed where
  it was.
- **It does not change what the container listens on.** That is always 3001 and
  3443, and it is not a setting.

---

## Related

- [Deploying PathLMS](../DEPLOYMENT.md), for the install itself.
- [Putting PathLMS behind a proxy you already run](BEHIND-A-PROXY.md), if
  something in front holds your certificate.
- [Running PathLMS on Unraid](UNRAID.md).
