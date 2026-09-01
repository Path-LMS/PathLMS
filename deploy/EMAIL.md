# Setting up email

PathLMS sends one kind of message: the link somebody follows to set a new
password after forgetting theirs. Nothing else sends email to anybody.

**With no mail server configured, PathLMS runs normally.** The "forgot your
password" page tells people to ask an administrator instead. Setting mail up is
worth doing, and nothing breaks while you have not.

**There is no screen for this.** Mail is configured in the settings file beside
your compose file, and nowhere else.

---

## Before you start

Open a terminal on your server, in the folder that holds `docker-compose.yml`.
The file you will edit is called `.env` and sits beside it.

Have ready, from whoever provides your email:

- the mail server's address
- the port it wants
- the account name and password it wants you to sign in with

For Gmail, Microsoft 365, and most large providers, the password is an **app
password** created in that account's security settings. The password you use to
read your own mail will be refused.

---

## The steps

**1. Copy the settings file, so you can put it back.**

```
cp .env .env.backup
```

No output means it worked.

**2. Check PathLMS knows its own address.**

Open `.env` and find the line starting `PATHLMS_PUBLIC_URL=`. It must name the
address people actually type:

```
PATHLMS_PUBLIC_URL=https://learn.example.com
```

If it is missing or empty, add it now. **Mail will not start without an
address**, and this is deliberate: a recovery link built from a guess fails days
later, in a stranger's inbox, for somebody who cannot report it.

**3. Add the mail server.** At the end of `.env`:

```
SMTP_HOST=smtp.example.com
```

This one line is what switches mail on.

**4. Add the port.**

```
SMTP_PORT=587
```

A plain whole number. Leave the line out and PathLMS uses 587. Write something
that is not a number and PathLMS will not start.

**5. Say whether the connection is encrypted from the first byte.**

```
SMTP_SECURE=false
```

Use `false` with port 587, which is the usual case. Use `true` only with port
465.

**This setting is badly named and `false` does not mean unencrypted.** It means
"encrypted from the very first byte", which is how port 465 works. Port 587 works
the other way round: the connection opens in plain text and is upgraded
immediately. Both are encrypted, and PathLMS refuses to send at all if the
upgrade does not happen, so a server that cannot encrypt gets no mail and no
password rather than both in the open.

**6. Add the account.**

```
SMTP_USER=no-reply@example.com
SMTP_PASSWORD=your-app-password
```

Set both or neither. If your mail server accepts mail from your own server with
no sign-in, leave both lines out. Setting one without the other stops PathLMS
starting.

**7. Add the address messages come from.**

```
MAIL_FROM=PathLMS <no-reply@example.com>
```

This is a separate setting from the account name above, and it is required once
mail is on. Many providers refuse to send unless this is an address they host
for you.

**8. Save the file, then restart.**

```
docker compose up -d
```

**9. Check it started.**

```
docker compose ps
```

The api service should say running. If it is restarting over and over, read the
next section.

**10. Test it for real, because nothing else proves it.**

There is no test button. Sign out, go to `/forgot-password`, enter the address
of a real account, and then go and look in that mailbox. The page says the same
thing whether or not anything was sent, on purpose, so that the page cannot be
used to discover which addresses have accounts. It therefore tells you nothing
about whether your settings work.

---

## When something is wrong

**PathLMS will not start.** This is the commonest mistake and it is deliberate:
a half-finished mail setup is refused rather than silently swallowing every
message forever.

```
docker compose logs api
```

Look for a line beginning `FATAL: Mail is half-configured`. It lists exactly
which settings are missing, by name. Fix those lines and run
`docker compose up -d` again.

To back out entirely, put your copy back and restart:

```
cp .env.backup .env
docker compose up -d
```

With none of the mail settings present, PathLMS starts normally with mail off.

**PathLMS starts and no message arrives.** The settings are complete and
something in them is wrong, most often the password or the port.

```
docker compose logs api | grep "\[mail\]"
```

A failed send prints a line beginning `[mail] a message could not be delivered:`
followed by your mail server's own words, which usually name the problem. No
such line and no message means the address you tried has no account here.

**The link in the message points at the wrong address.** Links are built from
the address in step 2, and never from anything a visitor sends. If they are
wrong, that setting is wrong.
