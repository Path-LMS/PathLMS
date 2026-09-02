# Putting PathLMS behind a proxy you already run

Something else holds your certificate and answers on the address people type.
Turn on one setting so PathLMS believes it, point the proxy at the plain port,
and close that port to everybody else.

That is the whole job, and this page is the long version of it. Every claim here
was read out of the files this release ships, except where a section says
otherwise.

---

## 1. Does this page apply to you?

**One test.** When somebody types your address into a browser, does the padlock
come from PathLMS, or from something else?

- **From something else.** nginx, Caddy, Traefik, HAProxy, a cloud load
  balancer, Cloudflare, or an appliance's built-in reverse proxy holds the
  certificate and forwards the request onward. **This page is for you.**
- **From PathLMS itself.** You generated or uploaded a certificate under
  Settings, then Network, then Encryption, and browsers connect to PathLMS on
  port 3443. **This page is not for you, and you should not turn on the setting
  below.** There is nothing extra to do.
- **Neither, because nothing is encrypted at any point.** Reasonable only on a
  closed network nobody outside can reach. This page is not for you either.

---

## 2. What the proxy has to do

Five things. All five are ordinary reverse proxy behavior, and most proxies do
four of them without being asked.

1. **Answer on the address people type**, on port 443, with the name that
   appears in the browser bar.
2. **Hold the certificate for that name**, and renew it. Nothing inside PathLMS
   obtains or renews a certificate for you when your proxy is the one serving it.
3. **Send the header `X-Forwarded-Proto: https` itself, replacing anything the
   visitor sent.** This is the one that matters, and it is explained below.
4. **Send the header `X-Forwarded-For` carrying the visitor's own address.**
   Almost every proxy already does. What it is for is explained below, and it
   only counts once you have also done step two of section 5.
5. **Pass the request through to PathLMS on port 3001**, keeping the `Host`
   header exactly as the browser sent it.

### Why the overwrite in step 3 matters

That header is how the proxy tells PathLMS "the person you are answering reached
me over an encrypted connection." A visitor can send that header too, because a
header is just text anybody can type. If the proxy passes along whatever arrived
instead of writing its own, then a visitor gets to make the claim about
themselves, and PathLMS believes it. Overwriting means the only thing that can
make the claim is the thing that actually knows.

### Why the forwarded address in step 4 matters

Once a proxy is in front, every request reaches PathLMS from one address: the
internal one your proxy comes through. Everybody looks like the same visitor.

That costs two things, and the first one is the one people actually notice.

**PathLMS limits some things per visitor, and counts them by address.** One
address means one budget for the whole deployment instead of one each. Company
sign-in becomes ten a minute for everybody combined. Uploading files becomes
five a minute for the entire organization, which is few enough that one person
building a course will meet it on their own and see their own uploads refused.

**And every address PathLMS writes down names your proxy rather than a person.**
The activity trail, the security events, and the screen showing somebody which
devices they are signed in on all say the same wrong thing about everybody. A
record naming the wrong address is worse than one naming none, because it reads
exactly like a record naming the right one.

The header alone does not fix it. PathLMS will not believe a forwarded address
until you have told it where your proxy is, which is step two of section 5, and
the reason for that is in the warning there.

### Why the `Host` header in step 5 matters

Uploaded pictures and files are fetched using addresses PathLMS signs, and the
signature covers the host name. A proxy that rewrites `Host` to the internal
machine name breaks every one of those signatures, so uploads and downloads fail
while every page still loads perfectly. Send the original `Host` through
unchanged.

---

## 3. A worked nginx server block

Complete and copyable. It goes in your nginx configuration, wherever your other
sites live, which on most installations is a file under `/etc/nginx/sites-available/`
or `/etc/nginx/conf.d/`. Replace `learn.example.com` with your address, and
`127.0.0.1:3001` with wherever PathLMS is reachable from the proxy.

```nginx
server {
    listen 443 ssl;
    http2 on;

    # The name people type. It has to match the certificate, and it has to match
    # PATHLMS_PUBLIC_URL in your settings file.
    server_name learn.example.com;

    ssl_certificate     /etc/letsencrypt/live/learn.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/learn.example.com/privkey.pem;

    # Uploaded files can be large. PathLMS accepts up to 500 megabytes on its
    # upload paths, and nginx allows one megabyte by default, so without this
    # line the proxy refuses the upload before PathLMS ever sees it.
    client_max_body_size 500M;

    location / {
        proxy_pass http://127.0.0.1:3001;

        # REQUIRED. This is the header PathLMS reads to decide whether the
        # request was encrypted. The value has to be exactly "https", and it has
        # to be written here rather than passed along, so that a visitor cannot
        # claim it for themselves. Casing does not matter. A list such as
        # "https, http" is refused, which is what a proxy that appends rather
        # than replaces would produce.
        proxy_set_header X-Forwarded-Proto https;

        # ALSO REQUIRED, for a different reason. Addresses for uploaded
        # files are signed, and the signature covers the host name. $host is the
        # name the browser asked for. Replacing it with the internal address
        # breaks every file link while leaving every page working.
        proxy_set_header Host $host;

        # Who asked. Without these, every entry in the PathLMS activity trail
        # and every rate limit sees this proxy's own address instead of the
        # visitor's, so everybody shares one budget and no record names a
        # person. Sending them is half the job: PathLMS ignores a forwarded
        # address until you tell it where this proxy is, which is step two of
        # section 5.
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Long enough for a large upload to finish. The nginx default is 60
        # seconds, and PathLMS allows 300 on its own upload paths.
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;

        # Large files stream through rather than being written to the proxy's
        # own disk first.
        proxy_request_buffering off;
    }
}

# Send plain requests to the encrypted address. Without this, somebody who types
# the address without https reaches nothing.
server {
    listen 80;
    server_name learn.example.com;
    return 301 https://$host$request_uri;
}
```

**If the block you already have passes through whatever forwarded value
arrived**, using something like `$http_x_forwarded_proto`, replace it with the
literal `https` shown above. Passing it through is the exact failure this page
exists to prevent.

---

## 4. The same thing in Caddy

Caddy obtains and renews the certificate on its own, so this is the whole of it:

```caddy
learn.example.com {
    reverse_proxy 127.0.0.1:3001
}
```

Caddy sets `X-Forwarded-Proto`, `X-Forwarded-For` and `X-Forwarded-Host` on
proxied requests, and passes the original `Host` header through unchanged, which
is the behavior PathLMS needs. Caddy also places no size limit on a request body
by default, so large uploads work with nothing added.

**Not verified by anybody here: whether Caddy replaces an `X-Forwarded-Proto`
that a visitor sent, or keeps it.** The setting that governs it is
`trusted_proxies` on the `reverse_proxy` directive, and Caddy's own
documentation for that directive is the thing to read before relying on the
answer. Whatever it says, the fourth check in section 6 settles it for your
installation in one command. Run it.

---

## 5. What to set in PathLMS

Four things. The first three go in your settings file, which is the file named
`.env` sitting beside `docker-compose.yml`. The fourth is a screen.

**One. Tell PathLMS to believe the forwarded header.**

```
PATHLMS_TRUST_FORWARDED_PROTO=true
```

Unset, empty, or anything unrecognized means only PathLMS's own connection
counts as encrypted. That is the safe direction and it is deliberate: believing
a forwarded header with nothing in front lets any visitor claim their own plain
connection was encrypted and be believed. A misspelling therefore lands on the
strict answer rather than the trusting one.

**`true` is the value to write.** These exact spellings work as well, and no
others: `TRUE`, `True`, `1`, `yes`, `YES`, `on`, `ON`. That is worth knowing for
the opposite reason too: if you typed one of those and assumed it had been
ignored, it was not. Anything else is treated as unset, `Yes` and `On` included.

**Two. Tell PathLMS where your proxy reaches it from**, so that a visitor's own
address survives the hop:

```
PATHLMS_TRUST_FORWARDED_ADDRESS_FROM=172.18.0.0/16
```

**Write the address your proxy comes from, not the address people type and not
this machine's own address.** On a stack like this one the proxy usually
arrives over a Docker network, so a whole network is the honest answer. This
command prints yours:

```
docker network inspect pathlms_frontend --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

Several are allowed, separated by commas or spaces. A single address works too,
such as `PATHLMS_TRUST_FORWARDED_ADDRESS_FROM=192.0.2.5`.

**Without this, section 2 step 4 does nothing**, and the two costs described
there are what your deployment has: one shared rate-limit budget for everybody,
and every recorded address naming your proxy.

**This one is different from the others on this page, and it is worth a
paragraph rather than a footnote.** You are asserting something PathLMS cannot
check: that nothing except your proxy can reach port 3001. Everything else here
is safe to get wrong. This is not. Anybody who can reach that port while this
setting names their address can put whatever address they like in a header, and
PathLMS will write it into an activity trail that cannot be corrected
afterwards, and will let them pick a fresh address for every request so that
the limits stop applying to them at all.

**So do section 7 before this, not after it.** Section 7 is how you close the
port. It is written as the thing that goes wrong most because it usually is,
and this setting is the one that makes it matter.

**A typo lands on the safe side.** Unset, empty, misspelled, a name instead of
an address, or an entry naming the whole internet (`0.0.0.0/0`, `::/0`, `any`)
all give you exactly what a deployment with no proxy gets: nothing forwarded is
believed. One bad entry refuses the whole setting rather than quietly applying
the rest, so it is never half in force.

**Three. Tell PathLMS the address people type**, which is the one at the proxy
and not the machine and port behind it:

```
PATHLMS_PUBLIC_URL=https://learn.example.com
```

**Get this right before the first start.** It decides which pages a browser is
answered from, the links inside recovery mail, the address file links are signed
against, and where company sign-in sends people back to. A wrong value gives you
a sign-in page that draws perfectly and then does nothing, with nothing on that
screen pointing at a setting.

Then run this in the directory holding `docker-compose.yml`. All three settings
take effect when the containers are created again, which is what this does and
what a restart does not.

```
docker compose up -d
```

**On an appliance with a stack interface, use its control for deploying or
building the stack, and not its Start control.** The difference is the one that
catches people out, and it is ordinary Compose behavior rather than anything
about your appliance: a container keeps the environment it was created with, so
starting a stopped one applies nothing you have just changed. `docker compose up
-d` creates the container afresh from the changed file. Stop and Start do not.
On some interfaces the deploy control is called Build, and an existing stack has
to be cleaned first so the old containers are removed. **Nobody here has tested
any particular appliance**, and the first check in section 6 settles it for
yours in one command: if it still says `direct` after you have saved the
setting, this is why.

**Four. Record how this deployment is reached.** Sign in as the administrator,
go to **Settings**, then the **Network** tab, and choose **Something else handles
encryption in front of this system**.

**That screen records your answer. It does not reconfigure anything**, and it
says so itself. It does not move your proxy, your certificate or your ports.
What it does is stop PathLMS guessing from the address alone, so it no longer
offers to make you a certificate you already have and do not want.

**Do not also generate a certificate inside PathLMS when your proxy already
holds one.** A certificate the browser does not trust, served together with an
instruction telling browsers to insist on encryption, is how somebody locks
themselves out of their own system with no way back from inside it.

### If you run this on a storage appliance

**If you run this on a storage appliance, that file is still the answer and you
probably do not need a terminal.** Unraid, Synology, QNAP and TrueNAS all run a
stack like this one through a compose feature of their own, and those features
offer somewhere in a browser to give a stack its settings. Put the three
settings from steps one, two and three there.

**What to look for is the stack's settings file, not a per container variable
form**, and the reason is worth one paragraph because it applies to every
PathLMS setting rather than to this one. Most of them are read by Compose while
it reads `docker-compose.yml`, before any container exists: the image
addresses, the storage directories and the four port settings decide what is
pulled, what is mounted and what is opened, all of which are settled before a
container starts. A value added to a container afterwards decides none of them.
This setting does reach a container, and even here `docker-compose.yml` already
carries a line for it that reads the value out of the settings file, so a second
line placed beside it is not an override. One file, one place, every setting.

**Unraid has a page of its own with the exact route**: [Running PathLMS on
Unraid](UNRAID.md).

**On any other appliance, check one thing, because nobody here has run PathLMS
on one.** The application container is handed a file named `.env`, by that exact
name, sitting beside `docker-compose.yml`. Some stack interfaces keep settings
in a list of their own and write them out under a different name or in a
different place. If yours does, the stack does not start and Compose says which
file it could not find, so it is a clean failure rather than a quiet one. After
you save your settings, look in the stack's directory for a file called `.env`
before you go looking for anything else.

---

## 6. How to prove it worked

Five checks, in this order. Run the first two on the server itself, in the
directory holding `docker-compose.yml`. Run the rest from any machine that can
reach both addresses.

**One. The web container writes down which answer it took**, on every start,
because a setting whose effect is otherwise invisible needs somewhere to report
itself:

```
docker compose logs web | grep "transport profile"
```

On an appliance, open the same log from the appliance's own container list: the
container is called `pathlms-web`, and the line is near the top of its log. That
way this check needs no terminal either.

Right answer:

```
transport profile: upstream. PATHLMS_TRUST_FORWARDED_PROTO is set, so a
forwarded claim of encryption from whatever sits in front is believed.
```

`upstream` means PathLMS is believing your proxy. If it says `direct` instead,
the setting did not reach the container: check the spelling against the list
above, and check that the settings file is named `.env` and sits beside
`docker-compose.yml`.

**Two. The web container also writes down whose address it is willing to
believe**, in the same log and for the same reason:

```
docker compose logs web | grep "visitor address"
```

Right answer, naming the network you actually wrote:

```
visitor address: trusting a forwarded address from 172.18.0.0/16. A visitor's
own address now decides which rate-limit budget they spend, and is what this
deployment records.
```

If it says `not trusting any forwarded address` instead, the line tells you
which of the reasons applies. `is not set, or is empty` means the setting did
not reach the container: check the file is named `.env` and sits beside
`docker-compose.yml`. A line naming a value in square brackets means PathLMS
would not believe what you wrote, and it says what to write instead. Either way
nothing is trusted, which is the safe direction and is also the state you were
in before.

**Three. A request through the proxy is treated as encrypted:**

```
curl -sI https://learn.example.com/ | grep -i strict-transport-security
```

Right answer:

```
strict-transport-security: max-age=31536000
```

That header is the visible proof. PathLMS sends it only when your proxy told it
the connection was encrypted, so seeing it means every part of the chain agreed.

**Four. A request straight to the PathLMS port is not:**

```
curl -sI http://<your-server-address>:3001/ | grep -i strict-transport-security
```

Right answer: **nothing at all.** No output. That request arrived unencrypted
and carried no forwarded claim, so PathLMS correctly declines to say otherwise.

**Five. Sign in through the proxy address, then look at your own address.** Two
things at once, and both need a browser rather than a command.

Upload a picture first: set a person's photograph under People, or put a
picture in a lesson, and confirm it appears after the page reloads. That is
what proves the `Host` header survived the hop, and no status code will tell
you.

Then go to **Settings**, then **Devices you are signed in on**. The device
marked **This device** shows the address it signed in from. **It should be your
own computer's address**, the one you would see on a site that reports it back
to you. If it instead shows an address beginning `172.` or `10.`, ending in
`.0.1`, and identical for everybody, that is the internal gateway rather than a
person. Step two of section 5 is not in force, and check two above says why.

**One thing check four does not prove, said plainly.** Add the header by hand to
that same direct request and PathLMS will believe it, because believing it is
exactly what you just asked for:

```
curl -sI -H "X-Forwarded-Proto: https" http://<your-server-address>:3001/
```

That request gets the header back. It is not a defect, and it is the reason for
the next section.

---

## 7. The thing that goes wrong most

**The port PathLMS listens on stays reachable from outside, so anybody can go
around the proxy entirely and reach PathLMS unencrypted.** Every deployment
behind a proxy has this on day one, because publishing a port on every network
interface is the default, and it is the right default for somebody with nothing
in front.

Once you have told PathLMS to believe a forwarded header, that same open port is
also where somebody can send the claim themselves.

**That matters more for the forwarded address than for anything else on this
page.** A forged claim of encryption makes the sender's own browser insist on
encryption and grants nobody anything. A forged address is written into an
activity trail that cannot be corrected afterwards, and it lets the sender
choose a fresh address per request and so spend everybody's rate-limit budget
one bucket at a time. If you have set
`PATHLMS_TRUST_FORWARDED_ADDRESS_FROM`, this section is not optional.

**How to close it when the proxy runs directly on the machine.** Two settings,
both in the same `.env` file, and these two in particular belong nowhere else:
Compose reads them to decide which port it opens at all, which is settled before
any container starts.

```
PATHLMS_PUBLISHED_ADDRESS=127.0.0.1:
PATHLMS_PUBLISHED_TLS_ADDRESS=127.0.0.1:
```

**The trailing colon on each is required and is not a typo.** The value is joined
to the port number, so an address written without the colon runs into it.

The first closes port 3001 to everything except the machine itself, which is
where your proxy is. The second does the same for port 3443, the encrypted port,
which is published on every installation and which you are not using when
something else holds the certificate. Leaving 3443 open on a deployment holding
no certificate lets anybody write one line to your log for every connection they
open, which on a server billed by log volume is a bill.

Then run `docker compose up -d` again, and check from a different machine that
both ports have gone:

```
curl -sS --max-time 5 http://<your-server-address>:3001/ ; echo "exit $?"
```

Right answer is a connection failure, not a page.

**Do not use these two settings if your proxy is itself a Docker container**,
even when that container runs on this same machine. `127.0.0.1` inside a
container means that container, not the machine, so a proxy container on a bridge
or macvlan network can no longer reach PathLMS the moment you set them. Use the
firewall rule below instead.

**How to close it when the proxy is on a different machine, or is a container.**
The two settings above would cut your proxy off as well, so use a firewall rule
instead: allow port 3001 from the proxy's address only, and deny it from
everywhere else. On a cloud provider that is a security group rule. Do this
rather than hoping, because it is the only thing standing between an open port
and the internet.

Once you are signed in, the **Ports** section of the **Network** tab reports
which of these you actually have, because it reads the same two settings. It
reports them; it does not change them. Both are changed in your settings file,
and [Changing the port people arrive on](CHANGING-THE-PORT.md) covers the rest of
what moves with them.

---

## 8. What this does not give you

Said plainly, because each of these is something a reasonable person might
assume comes along with it.

- **It does not encrypt the hop between your proxy and PathLMS.** That connection
  is plain text. On the same machine, or on a private network you control, that
  is normal and fine. Across a network you do not control it is not, and
  arranging encryption for that hop is yours to do.
- **It does not manage your certificate.** Obtaining and renewing it belongs to
  whatever is in front. Nothing in PathLMS will warn you when it is close to
  expiring.
- **It does not close the open port.** Section 7 is a separate job and nothing
  does it for you.
- **It does not change the address PathLMS uses in links and in mail.** That
  comes from `PATHLMS_PUBLIC_URL` and from nothing else, so a proxy set up
  perfectly with that setting wrong still produces recovery mail nobody can use.
- **It does not add sign-in protection, rate limiting or filtering.** Whatever
  your proxy does on those counts is what you get. PathLMS has limits of its
  own, and what step two of section 5 changes is only who they are counted
  against: per person once PathLMS knows where your proxy is, and everybody
  sharing one budget until then. The limits themselves are the same numbers.
- **It does not encrypt or authenticate the forwarded address.** Once you have
  told PathLMS where your proxy is, PathLMS believes what arrives from there.
  The only thing separating that from anybody on the internet writing their own
  address into your activity trail is section 7, and nothing does section 7 for
  you.
- **It does not cover the few seconds during an update when nothing answers.**
  This one is worth reading rather than skimming, because somebody hit it on the
  first upgrade of a real deployment.

  While PathLMS is updating it shows a page saying so, which reloads itself at
  five, ten, twenty, thirty and sixty seconds until the system is back. That page
  is drawn by PathLMS's own web server, and near the end of an update that web
  server is itself replaced. For those seconds nothing on the machine answers at
  all, so your proxy draws its own error page.

  **The harm is not the seconds. It is that your proxy's error page does not
  reload itself**, so somebody who lands in that moment loses the retry that
  would have carried them through and sits on a dead page until they refresh by
  hand. Over a five minute update that page reloads about nine times on its own,
  and each one is a chance to land in it.

  **What to do about it, in nginx or anything built on it.** Handle only the
  codes your proxy produces when it cannot reach PathLMS, and pass everything
  else through untouched:

  ```nginx
  # Only the codes this proxy makes itself when PathLMS is not answering.
  # 503 is deliberately absent: that is PathLMS's own update page, which is
  # better than this one and should be passed through unchanged.
  error_page 502 504 = @pathlms_updating;

  location @pathlms_updating {
      default_type text/html;
      add_header Cache-Control "no-store" always;
      add_header Retry-After 10 always;
      return 503 '<!doctype html><meta charset="utf-8"><title>PathLMS is updating</title><meta http-equiv="refresh" content="10"><style>body{font-family:system-ui,sans-serif;margin:0;min-height:100vh;display:grid;place-items:center;background:#faf9f7;color:#1c1b1a}main{max-width:30rem;padding:2rem;text-align:center}h1{font-weight:600;font-size:1.5rem}p{line-height:1.7;color:#57534e}</style><main><h1>PathLMS is updating</h1><p>It is installing a new version and will come back on its own in a moment. This page checks again every ten seconds, so there is nothing to do.</p></main>';
  }
  ```

  The line that refreshes every ten seconds is the whole point of the block. It
  puts back the retry PathLMS's own page would have done.

  **In Caddy there is a better answer**, because Caddy can avoid showing anything
  at all. `lb_try_duration 30s` with `lb_try_interval 1s` inside the
  `reverse_proxy` block makes it keep retrying instead of giving up on the first
  refused connection, so a visitor sees their page take a couple of seconds
  longer and nothing else. Put a `handle_errors` block underneath it for the case
  where PathLMS is down for a longer reason, because holding a request open for
  thirty seconds is the wrong answer to a real outage.

  **Test it before you need it.** Stop the PathLMS web container by hand and load
  your address. You should get your holding page, refreshing itself. Start the
  container again afterwards.
- **It does not let PathLMS run as more than one copy.** This product runs as a
  single instance. Put exactly one behind your proxy, and use the proxy for its
  certificate rather than for spreading load. Two copies break report downloads.

---

## Related

- [Running PathLMS on Unraid](UNRAID.md), which is where most people reading this
  page will have come from.
- `INSTALL.md`, which came with the same release, for the install itself.
