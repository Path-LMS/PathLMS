# Running PathLMS behind BunkerWeb, with ModSecurity and the OWASP rules on

BunkerWeb holds your certificate and filters every request before PathLMS sees
it. Two of its default protections and three of its filter rules mistake normal
PathLMS use for an attack. Switch off the two protections, add three narrow
exceptions for the rules, and leave the filter itself on and blocking.

That is the whole job, and this page is the long version of it. It builds on
[Putting PathLMS behind a proxy you already run](BEHIND-A-PROXY.md), which you
do first: the forwarded headers, the settings file and closing the open port are
all there, and none of it is repeated here. What PathLMS does was read out of the
files this release ships. The BunkerWeb setting names, the order its custom rule
files load in, the `bwcli` command and some of the log wording come from
BunkerWeb's own documentation and log output. Nobody here has run every BunkerWeb
version, so check the names against yours.

---

## 1. Does this page apply to you?

**One test.** Is BunkerWeb the thing answering on the address people type, with
ModSecurity and the OWASP Core Rule Set (CRS) switched on?

- **Yes.** **This page is for you.** Examples below use `lms.example.com` for the
  address people type. The internet reaches BunkerWeb on `<PUBLIC_IP>` and your
  own network on `<INTERNAL_IP>`.
- **Another proxy holds the certificate** (nginx, Caddy, Traefik, a cloud load
  balancer). This page is not for you. [BEHIND-A-PROXY.md](BEHIND-A-PROXY.md)
  is the whole job.
- **BunkerWeb is in front, but ModSecurity is off.** You will not see the
  filter symptoms below. Sections 3 and 4 still apply, because they are about
  BunkerWeb's rate limiting and not about the filter.

---

## 2. What BunkerWeb has to do

Four things.

1. **Answer on the address people type**, on port 443, for `lms.example.com`,
   and hold the certificate for it.
2. **Send the request on to `pathlms-web` on port 3001.** That is the plain web
   port from [BEHIND-A-PROXY.md](BEHIND-A-PROXY.md). Port 3443 is also
   published, and you do not use it here.
3. **Send `X-Forwarded-Proto: https` and the visitor's own address**, exactly as
   that page describes. BunkerWeb normally does both, and checks three and five
   in section 6 of that page prove it.
4. **Keep the connection limit on.** This is `USE_LIMIT_CONN`, and it caps how
   many connections one address holds open at once. PathLMS does not need it
   switched off, so it stays on.

### Point it at the web container, and at nothing else

PathLMS runs as several containers, and only one of them answers web requests.

| Container | Takes web requests? | Point BunkerWeb at it? |
| --- | --- | --- |
| `pathlms-web` | Yes, on port 3001 | **Yes. This is the only one.** |
| `pathlms-api` | Only from the web container, on the private network | No |
| The updater | It listens on nothing at all | No |
| The database, the cache, the file store | Only from inside the stack | No |

**The web container already forwards `/graphql` and the other API paths to
`pathlms-api` itself.** Pointing BunkerWeb at the API directly skips the
maintenance page, the security headers and the per-path limits that live in the
web container. Section 7 explains why the updater has nothing to point at.

In BunkerWeb's settings, for this one site, that is:

```
SERVER_NAME=lms.example.com
USE_REVERSE_PROXY=yes
REVERSE_PROXY_HOST=http://pathlms-web:3001
REVERSE_PROXY_URL=/
```

**Never allow every visitor in by adding their address to a whitelist.** It is
tempting, because it makes every symptom below go away at once. It also turns
off BunkerWeb's protection for that address for good. Everything on this page
fixes a cause instead.

---

## 3. Switch off BunkerWeb's rate limit: `USE_LIMIT_REQ`

```
USE_LIMIT_REQ=no
```

### What goes wrong

People see pages that half load. A course list appears with gaps, a save seems
to do nothing, and the browser's network tab shows status **429**, "Too Many
Requests". It clears after a few seconds and comes back.

BunkerWeb's log for it reads like this:

```
denied access from limit ... current rate = 12r/s ... max rate = 2r/s
```

The three phrases to look for are `denied access from limit`, `current rate`
and `max rate`.

### Why it happens

**PathLMS draws one screen with many small requests, all at the same moment.**
Everything the screen needs goes to the same address, `/graphql`, and a busy
screen sends a dozen of them in the first second. That is how the screen works,
and it is ordinary use.

BunkerWeb's default limit counts requests per address per second and calls
anything over its ceiling abuse. A burst of twelve is over a ceiling of two, so
BunkerWeb answers 429 to the ones that arrive last, and those are the ones the
screen was waiting for.

### Why switching it off is safe

**PathLMS limits itself, and its limits are set for what each path is.** The web
container allows `/graphql` a hundred requests a second with a burst of twenty,
company sign-in ten a minute, uploads five a minute, and security reports ten a
minute. Those numbers are set in the web container's own configuration.
BunkerWeb's one number for the whole site is cruder than all of them.

Those limits count by visitor address, so they only work per person once
PathLMS knows where BunkerWeb is. That is step two of section 5 in
[BEHIND-A-PROXY.md](BEHIND-A-PROXY.md), and it matters more here.

---

## 4. Switch off bad behavior banning: `USE_BAD_BEHAVIOR`

```
USE_BAD_BEHAVIOR=no
```

### What goes wrong

Somebody is fine all morning, then every page returns a block page and stays
that way for minutes or hours. A different person in the same building is fine.
Unbanning the address fixes it, and a day later it happens again.

### Why it happens

**BunkerWeb's bad behavior feature counts certain status codes, 429 included,
and bans the address that collects too many.** Before section 3, every burst
produced 429s, so the feature counted PathLMS's own screens as misbehavior and
built toward a ban.

**It is worse when many people share one address.** A school, a company office
or a hairpin route (where your own network reaches the public address by going
out and back in) all show BunkerWeb a single address for dozens of people. One
ban then removes all of them.

### What this does not switch off

**Switching off rate limiting and bad behavior banning does not switch off the
filter.** ModSecurity and the CRS are separate features with separate settings,
and they stay on. A request that looks like an attack is still refused with a
403, whichever address it came from.

---

## 5. Keep ModSecurity and the OWASP rules on, and blocking

```
USE_MODSECURITY=yes
USE_MODSECURITY_CRS=yes
MODSECURITY_SEC_RULE_ENGINE=On
```

| Setting | What it does |
| --- | --- |
| `USE_MODSECURITY` | Turns on the filter that reads every request |
| `USE_MODSECURITY_CRS` | Loads the OWASP Core Rule Set, the community's list of what attacks look like |
| `MODSECURITY_SEC_RULE_ENGINE` | `On` refuses matching requests. `DetectionOnly` writes them down and lets them through |

**`DetectionOnly` is for diagnosis only.** Use it for a few minutes while you
work out which rule is misreading PathLMS, then put it back to `On`. A
deployment left in `DetectionOnly` has a filter that watches attacks go by.

### Tune the filter, never turn it off

Any filter built on general rules will sometimes misread an honest request. The
answer is to tell the filter about that one request. The answer is never to
stop filtering, because the same rules that misread a GraphQL query are the ones
stopping real injection attacks on every other path.

---

## 6. The three false alarms, and the exceptions that fix them

### What goes wrong

A page that worked yesterday returns **403 Forbidden** from BunkerWeb. Saving
something, searching, or filtering a list does it, and the same page loads
fine when you only read it.

### Why it happens

**Every screen in PathLMS talks to `/graphql`, and its requests carry things a
general rule reads as attacks.** The CRS scores each rule that matches. Each
match adds points to a running total for the request, and once the total
reaches the limit, rule `949110`, "inbound anomaly score exceeded", refuses it.
Three rules misread PathLMS. Each one adds points, and together they reach the
blocking limit.

| Rule | What it is for | Why it fires on PathLMS |
| --- | --- | --- |
| `932235` | Finds commands being smuggled in (remote code execution) | An ordinary GraphQL query looks like a command. Braces, field names and arguments look like code to it |
| `942290` | Finds database injection (SQL and MongoDB) | GraphQL variables are written with a dollar sign, as in `$first`, `$status` and `$search`. The rule reads that as a database operator |
| `920420` | Refuses content types the rule set does not expect | PathLMS tells browsers to report policy violations to `/api/csp-report`, and the browser sends each report with `Content-Type: application/csp-report` |

### The configuration

Two custom configurations, both attached to the `lms.example.com` service only.
The first is of BunkerWeb's type `modsec`, with this content exactly:

```
# GraphQL false-positive exclusions
SecRule REQUEST_URI "@streq /graphql" \
    "id:1001001,\
    phase:1,\
    pass,\
    nolog,\
    t:none,\
    ctl:ruleRemoveById=932235,\
    ctl:ruleRemoveById=942290"
```

The second is of type `modsec-crs`, with this content exactly:

```
# CSP reporting false-positive exclusion
SecRule REQUEST_URI "@streq /api/csp-report" \
    "id:1001002,\
    phase:1,\
    pass,\
    nolog,\
    t:none,\
    ctl:ruleRemoveById=920420"
```

**Why the second one is a different type.** Rule `920420` runs in the first
phase, so the exception has to be loaded before the OWASP rules, and BunkerWeb
loads `modsec-crs` before them and `modsec` after. The GraphQL rules run later,
so `modsec` is early enough for them.

**Where they go.** In BunkerWeb's web interface, create each custom
configuration with its type and attach it to the `lms.example.com` service. On
disk, each is a file ending in `.conf` inside the folder for its type (`modsec`
or `modsec-crs`) in BunkerWeb's configs folder, in a subfolder named for the
service's primary server name, which is `lms.example.com`. Attached to the
service and not to the whole installation, each applies to `lms.example.com` and
to nothing else BunkerWeb protects.

### What the exceptions say

When the request address is exactly `/graphql`, stop checking rules `932235` and
`942290` for this request, and do not write a log line about having decided so.
The second does the same for `/api/csp-report` and rule `920420`.

### What this does not give up

- **Each exception names one path and one or two rules.** `@streq` means the
  address must equal the text exactly, so `/graphql/anything` and every other
  path are untouched.
- **Both rules stay on everywhere else.** A command-injection attempt against a
  sign-in form or a search box is still caught by `932235`.
- **Every other rule still inspects `/graphql`.** SQL injection patterns the
  other rules know, oversized requests, broken encodings, scanner signatures
  and the rest all apply. Only these two misreadings are set aside.
- **PathLMS is not exempt.** The exceptions say nothing about which application
  is behind the filter. They say two rules misread one path.
- **`/graphql` is not unguarded behind the filter.** PathLMS checks every
  request's sign-in and permissions itself. The filter is a second check, and
  these exceptions make one rule's job on one path a little smaller.

**Never switch any of these rules off for the whole site or the whole
installation.** Two of them find command and database injection, and they are
the reason to have the filter.

### Why the `920420` exception is worth having

Browsers send a policy violation report when a page tries to load something it
was told not to. PathLMS collects them so an administrator can see what a
page tried to load. Without the exception the filter refuses every report, so
a real policy problem goes unseen. The report path accepts only a tiny body
(ten kilobytes) and is rate limited by the web container, so the exception
opens very little.

---

## 7. Let PathLMS answer its own maintenance page: `REVERSE_PROXY_INTERCEPT_ERRORS`

```
REVERSE_PROXY_INTERCEPT_ERRORS=no
```

### What goes wrong

During an update, people see a plain BunkerWeb error page that says the service
is unavailable. PathLMS's own page, which says what is happening and reloads
itself, never appears.

### Why it happens

**PathLMS answers 503 on purpose while it is being worked on.** When an update
starts, the API writes a small notice onto a volume the web container can read
but not change. While that notice exists, the web container answers 503 on every
route that reaches the API, and its own error page explains what is going on.
The same web container also serves the update's live step list at
`/maintenance.json`, read straight off that volume, so the page can say "taking
a backup" and not just "unavailable". The journal answers only while an update
is in progress. At any other time that address says 404.

**By default BunkerWeb replaces every error status from behind it with its own
page.** A deliberate 503 is an error status, so BunkerWeb swaps PathLMS's page
for its own generic one.

### How the maintenance behavior is divided

| Part | What it does |
| --- | --- |
| The API, or the update process | Writes the update's status to a shared volume |
| The shared volume | Carries the status. It outlives any container |
| The web container | Reads it. While the notice is there it answers 503 on normal routes, draws its own maintenance page, and serves `/maintenance.json` |
| The updater | Runs the update in the background. **It has no port and answers nothing, so there is nothing to point BunkerWeb at** |

**The part that speaks to the browser is the web container, which is why it is
the only thing BunkerWeb talks to.** The updater is background machinery. It
reads a file on a timer and replaces containers, and by design nothing can
connect to it.

### What switching this off does not switch off

**BunkerWeb still holds your certificate and encrypts the connection. It still
runs ModSecurity and the OWASP rules on every request.** It still applies its
other protections. The one thing that changes is that BunkerWeb stops rewriting
error pages that PathLMS itself produced.

**It also does not cover the few seconds during an update when the web container
itself is being replaced** and nothing answers. BunkerWeb draws its own "bad
gateway" for that, and section 8 of [BEHIND-A-PROXY.md](BEHIND-A-PROXY.md) has
the page that reloads itself.

---

## 8. The state to finish in

This is the whole configuration in one place. If yours matches it, you are done.

| Setting | Value |
| --- | --- |
| ModSecurity (`USE_MODSECURITY`) | `yes` |
| OWASP CRS (`USE_MODSECURITY_CRS`) | `yes` |
| `MODSECURITY_SEC_RULE_ENGINE` | `On` |
| `USE_LIMIT_REQ` | `no` |
| `USE_BAD_BEHAVIOR` | `no` |
| Connection limiting (`USE_LIMIT_CONN`) | `yes`, left as it is |
| `REVERSE_PROXY_INTERCEPT_ERRORS` | `no` |
| Rules `932235` and `942290` | Excepted on `/graphql` only, in a `modsec` configuration |
| Rule `920420` | Excepted on `/api/csp-report` only, in a `modsec-crs` configuration |

### What to switch off, and what never to

| Do not | Because |
| --- | --- |
| Turn ModSecurity or the OWASP rules off, for the site or the installation | It removes the filter to fix three false alarms |
| Turn off the command injection or database injection rules for the whole site | They are the reason to have a filter, and they are needed on every path except the two named |
| Exempt `/graphql` from the filter entirely | It is where nearly every request goes |
| Whitelist a visitor's address for good, an administrator's included | It switches the filter off for one address, and that address gets compromised like any other |
| Leave `MODSECURITY_SEC_RULE_ENGINE` on `DetectionOnly` | The filter then watches attacks and stops none |
| Turn a protection off because it gave a false alarm | Find out which rule or limit did it and deal with that one |

**Prefer the narrowest fix that works:** one path, and one rule, in that order.

---

## 9. How to prove it worked

Four checks, in this order. Run them on the machine running BunkerWeb.

**One. With the filter still in blocking mode, use PathLMS for a minute**, then
ask BunkerWeb whether any of the three rules or the score rule fired:

```
docker logs --since 2m <BUNKERWEB_CONTAINER> 2>&1 | \
grep -E '932235|942290|920420|949110'
```

Right answer: **no output.** Open a course list, save a setting and search for
a person while you watch. Any line naming one of those numbers means an
exception is not in force, or another rule is involved: read the line for the
path it names. If `920420` is the one, check that its block is in a `modsec-crs`
configuration and not a `modsec` one.

**Two. Look for anything else refused or limited**, over a slightly longer
window:

```
docker logs --since 5m <BUNKERWEB_CONTAINER> 2>&1 | \
grep -Ei '403|429|932235|942290|920420|949110|denied access'
```

What to look for:

- `429` or `denied access from limit` means a rate limit is still on. Check
  `USE_LIMIT_REQ`, and that the change was saved and BunkerWeb reloaded.
- `403` with a rule number you do not know is a different rule refusing a
  different request. Note the path and the rule, and handle it as in section 6:
  one path, one rule. Do not reuse the exceptions above for it.
- `949110` on its own, with no rule named beside it, is the total reaching the
  limit. The line just before it in the log names the rules that added up to it.

**Three. During a real update, the page is PathLMS's own.** Start an update on a
test deployment and, while it runs, load `https://lms.example.com/` in a
browser. You want PathLMS's "back in a moment" page, not BunkerWeb's. If you see
BunkerWeb's, `REVERSE_PROXY_INTERCEPT_ERRORS` is still on. **Do not test this by
stopping the web container.** With nothing answering, BunkerWeb draws its own
"bad gateway" whatever this setting says, so the check would fail on a healthy
setup.

**Four. A real attack is still refused.** This is the check that proves the
exceptions are narrow. A request on a path that is not excepted, carrying an
obvious injection string, should still be refused:

```
curl -s -o /dev/null -w '%{http_code}\n' \
  'https://lms.example.com/?q=1%27%20OR%20%271%27=%271'
```

Right answer: **`403`.**

### If one person is locked out right now

**Unban the address, and then find out why it was banned.**

```
docker exec <BUNKERWEB_CONTAINER> bwcli unban <CLIENT_IP>
```

This is a first-aid step and not a fix. If the ban came from bad behavior
counting, section 4 is the fix. If an address is banned again after you unban
it, something is still producing the responses that earn a ban, and the
`grep` above shows which.

---

## 10. Logs are sensitive, so clean them before they go anywhere

**ModSecurity's audit log records whole requests, and a request carries the
person's credentials.** A logged line can contain:

```
Authorization: Bearer <token>
```

Anyone who reads that token can act as that person until it expires.

**Before any log excerpt goes into a troubleshooting note, a document, a
screenshot, an issue report or a knowledge base, clean it.** Search the excerpt
for `Bearer ` and for `Authorization`, and replace whatever follows each with
`<REDACTED>`. Then remove the rest of what should not travel:

- session cookies
- signed links, which carry a signature that lets anyone open the file
- passwords, keys and any other secret
- visitor addresses, unless the address is the point

Look at the excerpt a second time before you send it. Checking your own pasted
text is quicker than rotating a credential afterwards.

---

## 11. What this does not give you

Said plainly, because each is something a reasonable person might assume comes
along with it.

- **It does not make a false alarm impossible.** A new PathLMS version may send
  a request the rules misread in a new way. The fix is the same: find the rule,
  find the path, add one exception.
- **It does not replace PathLMS's own checks.** Sign-in, permissions and its own
  limits all still apply behind the filter.
- **It does not encrypt the hop between BunkerWeb and PathLMS.** See section 9 of
  [BEHIND-A-PROXY.md](BEHIND-A-PROXY.md).
- **It does not close the open port.** Section 7 of the same page does, and
  with a filter in front it matters more, because a visitor who reaches port
  3001 directly goes round the filter completely.

---

## Related

- [Putting PathLMS behind a proxy you already run](BEHIND-A-PROXY.md), the
  page this one builds on.
- [Running PathLMS on Unraid](UNRAID.md), if BunkerWeb runs on the same
  appliance.
- `INSTALL.md`, which came with the same release, for the install itself.
