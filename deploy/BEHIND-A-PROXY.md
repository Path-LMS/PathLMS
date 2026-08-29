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

Four things. All four are ordinary reverse proxy behavior, and most proxies do
three of them without being asked.

1. **Answer on the address people type**, on port 443, with the name that
   appears in the browser bar.
2. **Hold the certificate for that name**, and renew it. Nothing inside PathLMS
   does this for you when something else is in front.
3. **Send the header `X-Forwarded-Proto: https` itself, replacing anything the
   visitor sent.** This is the one that matters, and it is explained below.
4. **Pass the request through to PathLMS on port 3001**, keeping the `Host`
   header exactly as the browser sent it.

### Why the overwrite in step 3 matters

That header is how the proxy tells PathLMS "the person you are answering reached
me over an encrypted connection." A visitor can send that header too, because a
header is just text anybody can type. If the proxy passes along whatever arrived
instead of writing its own, then a visitor gets to make the claim about
themselves, and PathLMS believes it. Overwriting means the only thing that can
make the claim is the thing that actually knows.

### Why the `Host` header in step 4 matters

Uploaded pictures and files are fetched using addresses PathLMS signs, and the
signature covers the host name. A proxy that rewrites `Host` to the internal
machine name breaks every one of those signatures, so uploads and downloads fail
while every page still loads perfectly. Send the original `Host` through
unchanged.

---

## 3. A worked nginx server block

Complete and copyable. Replace `learn.example.com` with your address, and
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

        # LOAD BEARING. This is the header PathLMS reads to decide whether the
        # request was encrypted. The value has to be exactly "https", and it has
        # to be written here rather than passed along, so that a visitor cannot
        # claim it for themselves. Casing does not matter. A list such as
        # "https, http" is refused, which is what a proxy that appends rather
        # than replaces would produce.
        proxy_set_header X-Forwarded-Proto https;

        # ALSO LOAD BEARING, for a different reason. Addresses for uploaded
        # files are signed, and the signature covers the host name. $host is the
        # name the browser asked for. Replacing it with the internal address
        # breaks every file link while leaving every page working.
        proxy_set_header Host $host;

        # Who asked. Without these, every entry in the PathLMS activity log and
        # every rate limit sees the proxy's own address instead of the
        # visitor's, so one busy visitor looks like everybody at once.
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
answer. Whatever it says, the third check in section 6 settles it for your
installation in one command. Run it.

---

## 5. What to set in PathLMS

Three things. The first two go in your settings file, which is the file named
`.env` sitting beside `docker-compose.yml`. The third is a screen.

**One. Tell PathLMS to believe the forwarded header.**

```
PATHLMS_TRUST_FORWARDED_PROTO=true
```

Unset, empty, or anything unrecognized means only PathLMS's own connection
counts as encrypted. That is the safe direction and it is deliberate: believing
a forwarded header with nothing in front lets any visitor claim their own plain
connection was encrypted and be believed. A misspelling therefore lands on the
strict answer rather than the trusting one, which is why it has to be exactly
`true`.

**Two. Tell PathLMS the address people type**, which is the one at the proxy and
not the machine and port behind it:

```
PATHLMS_PUBLIC_URL=https://learn.example.com
```

**Get this right before the first start.** It decides which pages a browser is
answered from, the links inside recovery mail, the address file links are signed
against, and where company sign-in sends people back to. A wrong value gives you
a sign-in page that draws perfectly and then does nothing, with nothing on that
screen pointing at a setting.

Then start the stack again. Both settings take effect on the next start of the
web container.

```
docker compose up -d
```

**Three. Record the shape you are in.** Sign in as the administrator, go to
**Settings**, then the **Network** tab, and choose **Something else handles
encryption in front of this system**.

**That screen records your answer. It does not reconfigure anything**, and it
says so itself. It does not move your proxy, your certificate or your ports.
What it does is stop PathLMS guessing from the address alone, so it no longer
offers to make you a certificate you already have and do not want.

**Do not also generate a certificate inside PathLMS when something in front
already holds one.** A certificate the browser does not trust, served together
with an instruction telling browsers to insist on encryption, is how somebody
locks themselves out of their own system with no way back from inside it.

---

## 6. How to prove it worked

Four checks, in this order. Run them from a terminal on any machine that can
reach both addresses.

**One. The web container says which answer it took**, on every start, because a
setting whose effect is otherwise invisible needs somewhere to report itself:

```
docker compose logs web | grep "transport profile"
```

Right answer:

```
transport profile: upstream. PATHLMS_TRUST_FORWARDED_PROTO is set, so a
forwarded claim of encryption from whatever sits in front is believed.
```

If it says `direct`, the setting did not reach the container. Check the
spelling, and check that the settings file is named `.env` and sits beside
`docker-compose.yml`.

**Two. A request through the proxy is treated as encrypted:**

```
curl -sI https://learn.example.com/ | grep -i strict-transport-security
```

Right answer:

```
strict-transport-security: max-age=31536000
```

That header is the visible proof. PathLMS sends it only when something in front
terminated encryption, so seeing it means every part of the chain agreed.

**Three. A request straight to the PathLMS port is not:**

```
curl -sI http://<your-server-address>:3001/ | grep -i strict-transport-security
```

Right answer: **nothing at all.** No output. That request arrived unencrypted
and carried no forwarded claim, so PathLMS correctly declines to say otherwise.

**Four. Sign in through the proxy address and upload a picture.** Set a person's
photograph under People, or put a picture in a lesson, and confirm it appears
after the page reloads. That is what proves the `Host` header survived the hop,
and no status code will tell you.

**One thing check three does not prove, said plainly.** Add the header by hand to
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

**How to close it when the proxy runs on the same machine.** Two settings, both
in the same `.env` file:

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

Then start the stack again, and check from a different machine that both ports
have gone:

```
curl -sS --max-time 5 http://<your-server-address>:3001/ ; echo "exit $?"
```

Right answer is a connection failure, not a page.

**How to close it when the proxy is on a different machine.** The two settings
above would cut your proxy off as well, so use a firewall rule instead: allow
port 3001 from the proxy's address only, and deny it from everywhere else. On a
cloud provider that is a security group rule. Do this rather than hoping, because
it is the only thing standing between an open port and the internet.

Once you are signed in there is a screen for this too. The **Ports** section of
the **Network** tab offers **Every network on this machine** or **One address
only**, and it reads the real settings, so it can tell you which you actually
have.

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
  your proxy does on those counts is what you get. PathLMS has limits of its own
  and they are unchanged either way.
- **It does not let PathLMS run as more than one copy.** This product runs as a
  single instance. Put exactly one behind your proxy, and use the proxy for its
  certificate rather than for spreading load. Two copies break report downloads.

---

## Related

- [Running PathLMS on Unraid](UNRAID.md), which is where most people reading this
  page will have come from.
- `INSTALL.md`, which came with the same release, for the install itself.
