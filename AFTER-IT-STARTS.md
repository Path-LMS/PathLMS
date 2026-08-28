# After it starts

Five things on your first day, in this order. The first one takes two minutes and
prevents the only failure in PathLMS that has no button, no command and no script
to get you out of.

---

## 1. Create a second administrator. Today.

**There is no supported way to recover the first administrator's password, and
being the only administrator who has forgotten it can lock your organization out
permanently.**

Here is the trap, stated plainly because it is easy to walk into.

The step that creates the first administrator runs **only while there are no
accounts at all**. Once you have signed in, an account exists, so that step never
fires again. Resetting a password requires the system to send email, and **email
is not configured by default**. So if you are the only administrator, you forget
the password, and no mail server is set up, nothing in the product will let you
back in.

What to do about it, which takes a minute:

1. Keep the password in a password manager, not in your head and not in a file on
   the same machine.
2. **Create a second administrator account that somebody else controls.** Two
   administrators means one can always restore the other.
3. If real people are going to sign in, set up email as well, so that ordinary
   password resets work for everybody rather than for nobody.

If it has already happened, the installation is not lost, but recovering it means
somebody with access to the database writing a new password hash directly into
the users table. That is a job for whoever administers the server, and the
product will not walk you through it.

## 2. Remove the first administrator's password from the settings file

Open `.env` and empty the `INITIAL_ADMIN_PASSWORD` line. It has done its job: it
acts only while there are no accounts at all, so from now on it is a working
password sitting in a file for no reason.

Deleting it changes nothing about your ability to sign in.

## 3. Tell PathLMS how it is reached

Go to **Settings**, then the **Network** tab. The first section is **How this
system is reached**, and it asks one question with three answers:

| Answer | What it means |
| --- | --- |
| This system handles encryption itself | Browsers connect straight to PathLMS and it holds the certificate. |
| Something else handles encryption in front of this system | You run nginx, Caddy, Traefik or a load balancer in front, and it holds the certificate. |
| Nothing encrypts the traffic | Reasonable only on a closed network nobody outside can reach. |

**Answer it even though nothing appears to depend on it.** Until you do, three
other parts of that screen have to guess, and they guess separately: the
encryption section, the ports commentary, and whether the product offers to help
you with a certificate at all. Recording the answer once stops all three
hedging.

**All three answers are available to an installation built from a release**, and
which is true of yours depends on what you set up before you got here.

Out of the box the stack serves unencrypted HTTP and holds no certificate, so
most installations are the middle answer: something in front holds it. The first
answer needs one extra file a release attaches,
`docker-compose.encryption.yml`, plus a certificate you generate or upload from
the Encryption section further down this same tab.
[Deploying PathLMS](DEPLOYMENT.md) covers both.

The three sections beneath it are **Web address**, **Ports** and **Encryption**,
which is the order you will want them in.

## 4. Check the address is right

Still on the **Network** tab, the **Web address** section holds the address
people type to reach this deployment. It is the same value as
`PATHLMS_PUBLIC_URL` in your settings file, and it is worth checking with your
own eyes because a wrong one fails in a way that does not look like a wrong
address.

Four things derive from it:

- which pages a browser is answered from at all;
- the links inside password recovery mail;
- the address every uploaded file's link is signed against;
- where company sign-in returns people to.

**If it is wrong, the sign-in page renders perfectly and then does nothing, and
uploaded pictures do not appear.** No error on the screen points at the cause.

If something sits in front holding your certificate, this is the address at that
proxy, not the machine and port behind it.

## 5. Confirm a backup actually landed

Backups run on their own from the first start: one a day by default, kept for
fourteen days, and the first runs immediately rather than after a day, precisely
so that you find out now if the directory is not writable.

Look in your backup directory and check there is a file in it.

Then do the thing almost nobody does: **restore one somewhere and check it comes
back.** A backup you have never restored is not a backup, and the moment you find
out otherwise is the worst possible moment.

### How to restore one

The restore script travels inside the database image, because a server has no
copy of the source for it to come from. Copy it out once, onto the machine:

    docker cp pathlms-db:/opt/pathlms/scripts/. ./pathlms-scripts/

**To rehearse, restore into a scratch database rather than over your real one.**
This is the version to run today, while nothing is wrong:

    export BACKUP_DIR=/var/lib/pathlms/backups
    DB_NAME=pathlms_restore_test ./pathlms-scripts/restore-database.sh --latest

`BACKUP_DIR` is wherever you pointed `PATHLMS_BACKUP_DIR`, and `--latest` picks
the most recent dump in it. Naming a different `DB_NAME` builds the restored copy
beside your live database instead of on top of it, so you can look at the result
and then throw it away.

**To restore for real**, over the live database, leave `DB_NAME` out. It
overwrites everything and asks you to type the word `RESTORE` before it will:

    ./pathlms-scripts/restore-database.sh --latest

The script compares what came back against an inventory taken at backup time and
tells you what it checked. That comparison exists because a restore of this
database once lost a piece of it and reported success.

Two more things worth arranging while you are here:

- **Copy the backups off the machine.** They are on a schedule, and they are on
  the same machine as the thing they are protecting until you move them. Point
  `PATHLMS_BACKUP_DIR` at a path that something else syncs elsewhere.
- **Check who owns the files** with `ls -l` before you write that sync job, so it
  has permission to read them.

---

## Then the ordinary setup

The administrator home screen shows a five step list. Nothing behind those steps
is locked, there is no wizard deciding what you do first, and you can ignore all
five and work in whatever order suits you.

Roughly, the order most people want is:

1. **Set up your groups**, because a person's place in the structure can decide
   what they are given.
2. **Add people**, or connect company sign-in so they arrive on their own. The
   **SSO / Authentication** tab in Settings sets that up, and speaks the two
   standards most company identity providers use, OIDC and SAML.
3. **Set your brand color and logo** on the Appearance screen. Set one color and
   the whole palette is generated from it. Have the logo ready as a PNG or a
   WebP: uploads in SVG are refused, because that format can carry a program
   inside it.
4. **Write a course.**
5. **Enroll people**, which takes one pass: pick people or a whole group, pick
   the courses, done. It then names anybody it could not enroll and why.

**Set up email early if real people will use this.** Without it, nobody can reset
their own password, including you.

## What to read next

- [Deploying PathLMS](DEPLOYMENT.md), if something about the environment is still
  unsettled.
- [How updates work](UPDATES.md), before the first one arrives rather than after.
