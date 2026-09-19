# After it starts

Five things to do on day one, in this order. The first one takes two minutes
and saves you a bad afternoon.

---

## 1. Create a second administrator today

Here is the trap. The step that creates the first administrator only runs
while there are no accounts at all. Once you have signed in, that step never
fires again. Resetting a password needs email, and email is not set up by
default. Administrators also need an authenticator app, and phones get lost.

What to do, in a few minutes:

1. Keep the password in a password manager, not in your head or a file on the
   same machine.
2. Create a second administrator account that somebody else controls. Two
   administrators means one can always restore the other.
3. Add an authenticator app to your own account: My settings, then
   Authenticator app, then Set up. The button that updates this installation
   asks for one, so do this before your first update, not during it.
4. If real people will sign in, set up email too, so password resets work for
   everybody, not just you. See "Set up email early", below.

### If it has already happened: the rescue command

Run this on the machine PathLMS is installed on. Nobody can reach it over the
internet, and there is no button for it in the product.

    docker exec -it pathlms-api pathlms-rescue you@your-company.example --yes

It makes a new password, prints it once, and signs that account out
everywhere. Copy the password before you close the window: it is not stored
and will not be shown again. Then sign in and change it.

It also removes that account's company sign-in link, if it had one, because
that sign-in method does not ask for an authenticator app. If the account was
suspended, it makes it active again.

If the phone is the problem, add `--forget-the-phone`. That clears their
authenticator app and recovery codes so they can set up a new one.

Anybody who can run a command on that server can already read the whole
database and set a password by hand. This command just turns an hour of that
work into one line, so treat access to that server as access to every
account.

Every use leaves a line in the activity log, at the highest severity, that
cannot be removed afterwards. Nobody is notified. It exists so a rescue can't
be done quietly by someone who is meant to be trusted.

## 2. Remove the first administrator's password from the settings file

Your settings file is `.env`, in the same directory as `docker-compose.yml` on
the server. Empty the `INITIAL_ADMIN_PASSWORD` line, so it reads
`INITIAL_ADMIN_PASSWORD=` with nothing after it, and save.

On a storage appliance that runs the stack for you, the same file is the
settings or environment editor inside that appliance's stack screen.

It already did its job: it only works while there are no accounts. Deleting
it changes nothing about your ability to sign in, and there is nothing to
restart.

## 3. Tell PathLMS how it is reached

Go to Settings, then the Network tab. The first section, "How this system is
reached," offers three answers:

| Answer | What it means |
| --- | --- |
| This system handles encryption itself | Browsers connect straight to PathLMS and it holds the certificate. |
| Something else handles encryption in front of this system | You run nginx, Caddy, Traefik or a load balancer in front, and it holds the certificate. |
| Nothing encrypts the traffic | Reasonable only on a closed network nobody outside can reach. |

Answer it even though nothing seems to depend on it. Until you do, the
encryption section, the ports section and the certificate advice each guess
from your address alone, and may offer to make you a certificate you already
have.

Out of the box the stack serves unencrypted traffic and holds no certificate,
so most installations pick the middle answer: a reverse proxy or load
balancer holds the certificate. The first answer needs nothing extra: port
3443 is already open, waiting for a certificate you generate or upload from
the Encryption section further down the same tab. [Deploying
PathLMS](DEPLOYMENT.md) gives the order to do that in: switching your address
to the `https://` one before the certificate exists locks you out.

The three sections below are Web address, Ports and Encryption, in the order
you will want them.

The Ports section only reports. It shows which ports this deployment
publishes and which networks they are open on, so you can check them against
what you meant. Moving a port is an edit to your settings file and one
command: read [Changing the port people arrive
on](deploy/CHANGING-THE-PORT.md) first, not after, because its section on a
deployment that does not come back assumes you can no longer reach this
screen.

## 4. Check the address is right

A wrong address here fails quietly: the sign-in page renders fine and then
does nothing, uploaded pictures do not appear, and nothing on screen points
at the cause. Check it yourself, it takes thirty seconds.

Still on the Network tab, the Web address section holds the address people
type to reach this deployment, the same value as `PATHLMS_PUBLIC_URL` in your
settings file. It decides:

- which pages a browser is answered from at all
- the links inside password recovery mail
- the address every uploaded file's link is signed against
- where company sign-in returns people to

If a reverse proxy or load balancer holds your certificate, this is the
address at that proxy, not the machine and port behind it.

## 5. Confirm a backup actually landed

Backups run on their own from the first start. The database and uploaded
files are copied once a day, kept for fourteen days, and the first copy runs
immediately, so you find out now if the directory cannot be written to.

Check there is a file in your backup directory. That is whatever you set
`PATHLMS_BACKUP_DIR` to, or `/var/lib/pathlms/backups` if you set nothing:

    ls -l /var/lib/pathlms/backups

Look for a file starting `pathlms-` and ending `.sql.gz` (the database), with
more beside it for the uploaded files and a list of what the backup
contained. An empty directory means the backup could not write. Fix that
today.

Then do the thing almost nobody does: restore one somewhere and check it
comes back. A backup you have never restored is not a backup.

### How to restore one

Run everything here in a terminal on the server, in the directory holding
`docker-compose.yml`.

The restore script lives inside the database image, so copy it out once:

    docker cp pathlms-db:/opt/pathlms/scripts/. ./pathlms-scripts/

To rehearse, restore into a scratch database rather than your real one. Do
this today, while nothing is wrong. Make the scratch database first, since the
script restores into a database rather than creating one:

    docker exec pathlms-db psql -U postgres -c "CREATE DATABASE pathlms_restore_test"

    export BACKUP_DIR=/var/lib/pathlms/backups
    DB_NAME=pathlms_restore_test ./pathlms-scripts/restore-database.sh --latest

`BACKUP_DIR` is wherever you pointed `PATHLMS_BACKUP_DIR`, and `--latest`
picks the newest copy in it. Naming a different `DB_NAME` puts the restored
copy beside your live database instead of over it, so you can look and then
throw it away:

    docker exec pathlms-db psql -U postgres -c "DROP DATABASE pathlms_restore_test"

It asks you to type `RESTORE` either way, because it replaces whatever
database it is pointed at. Read the two lines above that prompt first: they
name the file and the database, your last chance to notice you are about to
overwrite the live one.

To restore for real, over the live database, leave `DB_NAME` out:

    ./pathlms-scripts/restore-database.sh --latest

The script compares what came back against an inventory taken at backup time
and tells you what it checked. That check exists because a restore of this
database once lost a piece of it and reported success anyway.

Two more things to arrange while you are here:

- Copy the backups off the machine. Until you do, they sit on the same
  machine as the thing they are protecting. Point `PATHLMS_BACKUP_DIR` at a
  directory that a sync job you already run copies elsewhere.
- Check who owns the files with `ls -l` before you write that sync job, so it
  can actually read them.

### If you move the backup directory, set the second name too

There are two names for it, read by different things. Set only the first and
everything looks fine until the day it matters.

`PATHLMS_BACKUP_DIR`, in the settings file, is where the automatic daily
copies land. `BACKUP_DIR` is what every script you run by hand reads,
including the restore above and the upgrade. It defaults to `./backups`
beside your `docker-compose.yml` and does not follow `PATHLMS_BACKUP_DIR`.

So if you point `PATHLMS_BACKUP_DIR` elsewhere and change nothing else, the
upgrade writes its own backup into `./backups`, while the daily copies go to
the disk you chose. Both directories exist and have files in them, and
nothing tells you they are different. The copy you would need after a bad
upgrade is the one in the place you have not been checking.

Set `BACKUP_DIR` to the same directory before running anything by hand:

    export BACKUP_DIR=/mnt/your-backup-disk/pathlms

Put that line in the shell profile of whoever runs the upgrade, so nobody has
to remember it.

---

## Then the ordinary setup

The administrator home screen shows a five step list. None of it is locked
and there is no wizard forcing an order; do them in whatever order suits you.

Roughly, most people go:

1. Set up your groups. The group someone belongs to can decide what they are
   given.
2. Add people, or connect company sign-in so they arrive on their own. The
   Sign-in tab in Settings sets that up, and speaks the two standards most
   company identity providers use, OIDC and SAML.
3. Set your brand colour and logo on the Appearance screen. Set one colour
   and the whole palette is generated from it. Use a PNG or WebP logo: SVG
   uploads are refused, because that format can carry running software
   inside it.
4. Write a course.
5. Enroll people. Pick people or a whole group, pick the courses, done. It
   then names anybody it could not enroll and why.

## Set up email early if real people will use this

Without it, nobody can reset their own password, including you.

There is no screen for this. Email is set in the same `.env` settings file as
everything else, and takes effect when you recreate the containers with
`docker compose up -d`. These are the lines:

    SMTP_HOST=smtp.example.com
    SMTP_PORT=587
    SMTP_SECURE=false
    SMTP_USER=your-account
    SMTP_PASSWORD=your-password
    MAIL_FROM=pathlms@example.com

`SMTP_HOST` and `MAIL_FROM` are always required. `SMTP_PORT` defaults to 587.
`SMTP_SECURE` is for a server encrypted from the first byte, usually port
465; leave it out otherwise. `SMTP_USER` and `SMTP_PASSWORD` go together: set
one and you must set the other, or leave both out for a relay that accepts
mail from this network without a password. The links inside the messages are
built from the address you already gave, so there is nothing else to set.

Leave every line out and PathLMS runs normally, telling anybody who has
forgotten a password to ask an administrator.

The full walkthrough, including how to test that mail actually sends and what
each provider needs, is on its own page: [Setting up email](deploy/EMAIL.md).

Set some of these lines and not the rest, and PathLMS will not start. A
half-configured mail server would accept every password reset and deliver
none of them. The refusal names the missing line, and it is only in the log:

    docker compose logs api

The same six lines are at the foot of the settings file the download came
with, commented out, so you can uncomment the ones you need instead of typing
them.

## What to read next

- [Deploying PathLMS](DEPLOYMENT.md), if something about the environment is
  still unsettled.
- [How updates work](UPDATES.md), before the first one arrives rather than
  after.
