![PathLMS. A learning path drawn as five courses in order: two finished, the third open and raised above the rest because it is the one being worked through now, and two still to come.](assets/banner.svg)

# PathLMS

**A learning path is an ordered list of courses. A course is a tree of modules,
sections and lessons, with quizzes where you want them. That is the whole
model.**

<!-- version -->
**Version 0.71.0** · [Releases](https://github.com/path-lms/pathlms/releases)
<!-- /version -->

Order is off until you ask for it. When you do, a locked lesson names what opens
it: *finish Fire Safety Basics first, and this opens.*

An organization can change ten things about how PathLMS looks, and cannot change
eleven others: the type scale, the spacing, the shadows, the motion, the focus
rings, which color means danger. You give it one brand color and it builds the
rest, then turns down any color it cannot make readable. Nobody on your side has
to have taste for the result to hold together.

It ships as a Docker Compose stack, so it runs anywhere containers do: a cloud
instance, a rented virtual machine, a server in the corner of an office, a NAS.
Images are published for Intel and ARM.

**[Download the latest release →](https://github.com/path-lms/pathlms/releases/latest)**

---

## What it does

- **Courses and lessons.** Write course material in a browser, arrange it into
  modules and lessons, publish it when it is ready.
- **Learning paths.** Put several courses in an order and enroll somebody in the
  whole thing at once.
- **Quizzes.** Ask questions, mark them automatically, record the score against
  the person's enrollment.
- **People and groups.** Groups nest. Give a group a folder of courses and
  everyone in that group gets them.
- **Enrolling a group.** Pick people or a whole group, pick the courses, and it
  enrolls them. Then it names anybody it could not enroll, and why.
- **Reports as questions.** Pick the question you were asked, narrow the answer,
  export it. The export asks first whether the file should carry people's names.
- **An activity record.** Significant actions go into a log the application
  itself cannot alter or delete, each entry linked to the one before.
- **Erasing somebody.** When a request to be forgotten arrives, erase the person
  from the People screen. It tells you what went and what was kept.
- **Your own look.** Set one brand color and the whole palette is generated from
  it. Upload a logo. Light and dark are designed separately rather than one being
  an inversion of the other.

## PathLMS in action

![The administrator home page, with a Get Started panel reading 0 of 5 complete above five unchecked setup steps, from setting up groups to enrolling learners.](images/the-first-screen.png)

*The first screen after setup. Five steps, and you can ignore all five: nothing
behind them is locked and no wizard decides what you do first. Your own first
screen is emptier than this one, which has the optional example data loaded.*

![The reports screen: seven cards, each headed with a question such as Who completed what? or Who's overdue?, each with a View Report button.](images/reports-are-questions.png)

*Reports are questions, not a query builder. Pick the one you were asked, narrow
the answer, export it. The export asks first whether the file should carry
people's names, and the answer is no until you change it.*

![The Enroll people dialog at step one of three, People then Courses then Result, with a toggle between choosing individual people and whole groups.](images/enrolling-a-group.png)

*Pick people or a whole group, pick the courses, done. Then it names anybody it
could not enroll and why, with a link straight to that person. A number tells you
something went wrong. A name tells you who to go and see.*

![A lesson page. The word Extraction is lightly underlined, and a small panel beside it, labeled What it means on this course, gives the meaning that applies here.](images/a-term-in-a-lesson.png)

*The same word means different things to different teams. A word's meaning
belongs to the course it appears in, so a reader gets the meaning that fits what
they are reading. Where nobody has written one, the word is left plain, because
showing somebody the wrong meaning is worse than showing none.*

*These are photographs of the software running, not drawings of it.*

## Who it is for

The person this was built for is the one who ends up owning training without
having applied for the job: an operations manager, an office manager, a safety
coordinator, somebody in a small internal IT team. You need to know who still has
not done the thing, you need the list ready when a client or an auditor asks, and
you need to do it again next quarter for everybody who has joined since.

It suits an organization that would rather run software itself than rent it, and
that has somebody who can use a terminal for about half an hour, once. There is
no per-person charge and no license fee, so nobody gets left out of the fire
safety training to save money.

## Where this is the wrong answer

Reading this before you install is the point of it. Every product named here is a
good product that sensible people choose for good reasons.

- **You need annual or recurring refresher training.** Completion here is final
  and cannot yet be reset, so "everybody does this again every year" is not
  something PathLMS can do. Moodle, or any commercial platform.
- **You are a school, college or university.** There are no terms, no timetables
  and no LTI. Canvas, Moodle, Blackboard or D2L Brightspace.
- **You run classroom or scheduled training.** There are no sessions, rooms,
  attendance registers or waiting lists.
- **You want somebody else to run the server.** This is self-hosted only. Docebo,
  TalentLMS or hosted Canvas.
- **You need a support contract.** There is nobody to escalate to.
- **You are selling courses to the public.** Nothing here takes a payment.
  Thinkific or Teachable.
- **You need a license you can fork and pass on.** See [License](#license) below.
  Moodle, Canvas or Open edX.
- **You want a marketplace of extensions.** There is no plugin system, on
  purpose.

## What it does not do

These are the limits worth knowing before you commit, not after.

**Training cannot repeat.** Completion is final. Enrolling somebody again on a
course they have finished returns the record they already have rather than
starting them over, and the expiry date an enrollment can carry is stored and
never acted on. Due dates are shown and nothing chases anybody. **If your
requirement is annual refresher training, this is the limit that matters most to
you, and it is why the list above starts where it does.**

**One copy on one machine.** PathLMS cannot be scaled out. Report exports, their
download links and one of the rate limits live in a single running process's
memory, so a second copy behind a load balancer would break downloads and
multiply that limit. The stack names its containers explicitly, which prevents
scaling outright rather than letting somebody try it and find out.

**Encryption is something you set up, not something you get.** A fresh
installation serves unencrypted traffic on one port, and you choose one of two
ways to change that. Most deployments put a reverse proxy or a load balancer in
front holding the certificate, and nothing in the PathLMS settings changes when
you do. Or PathLMS holds the certificate itself and answers browsers directly,
using one you generate or upload from the Settings screen, which takes one extra
file that comes with the release. There is nothing to build or compile either
way, and the Network tab inside the product records which you chose so the rest
of the software stops guessing.

**Updates happen when you decide.** Nothing deploys itself. PathLMS tells you
when a newer release exists, runs its own safety checks, takes a backup and
records every step. The last step, replacing the running containers, is yours: it
deliberately will not do that, because anything able to create a container on a
machine can do anything else on that machine too, and the part of PathLMS facing
the internet is the last thing that should hold that power. In practice it is
three lines in a file and one command. See [Updates](UPDATES.md).

**There is no two-step sign-in.** Nothing in the browser reaches it and there is
no database behind it, so nobody can turn it on. Signing in with an account
people already have, through the two standards most company identity providers
speak, does have screens for setting it up. Test it against your own provider on
a deployment you can throw away before you depend on it.

**Assignments cannot be handed in.** A course can contain an assignment block and
an author can write one. There is no way for a learner to submit anything against
it and nothing that records a submission. Treat it as a place to put
instructions, not as a thing people turn in.

**Lessons are saved by hand.** There is no automatic save in the course builder,
so treat writing a lesson the way you would treat writing a document.

**Courses packaged by another authoring tool do not play.** Packages exported
from Articulate, Captivate or similar will not run on an installation built from
a release, even with the administrator's switch turned on. Everything else works.
This is a known gap and not a security hole: the route that would serve a
package's files checks the administrator's setting before it looks at anything
else, and answers nothing while that setting is off.

**One of the seven reports is not written yet.** "How are quizzes performing?"
opens and tells you so in as many words. Your data is not missing; that
particular report has not been built.

**No accessibility audit and no screen reader testing has been recorded.** What
is checked automatically, on every change, before anything ships: text against
its background in six brand colors in both light and dark, with the build
stopping on a pair nobody could read; every control large enough to tap; a focus
outline you can see when you tab to something, including under Windows High
Contrast; and motion that stops for anybody whose computer asks for less. Nobody
has sat down with a screen reader and no outside expert has reviewed it, so
WCAG 2.2 AA is the target rather than a claim.

**Two things leave the machine on their own, and nothing else does.** No learner
data, no names, no email addresses and no identifier of any kind is sent
anywhere.

- **A breached-password check.** When somebody sets a password, the first five
  characters of its hash go to a public service that reports whether the password
  appears in known breaches. The full password is never sent, and the service
  cannot tell which password was asked about. There is no setting to switch this
  off. On a network with no route out it times out and lets the password through.
- **A check for newer versions.** No more than once every six hours, the
  deployment asks a public container registry which versions of PathLMS have been
  published. What leaves the machine is a request for a public list of version
  numbers, carrying nothing about this installation. Nothing is downloaded and
  nothing is installed. Set `PATHLMS_UPDATE_CHECK=off` to stop it, and it fails
  quietly on a network with no route out rather than complaining at anybody.

One more thing can leave the machine, and only because somebody asked it to: when
an administrator types this deployment's own address into the Settings screen,
PathLMS tries to reach that address to tell them whether it works.

**Losing the only administrator's password locks you out, and nothing in the
product will let you back in.** The step that creates the first administrator
runs only while there are no accounts at all, and resetting a password needs
email, which is not configured by default. Getting back in from there means
somebody with access to the database writing a new password directly into it.
Create a second administrator on your first day. [After it
starts](AFTER-IT-STARTS.md) says this again, louder, because it is the one
mistake with no button to undo it.

## What you need

- A machine running **Docker** and the **Docker Compose plugin**. That is the
  only requirement. There is no source code to download, no `git`, no Node.js and
  no build step on the server.
- **Two processor cores and 4 GB of memory** is a comfortable starting point for
  a small organization. The application container is capped at 2 cores and 512 MB
  and the object store at 1 core and 512 MB by the stack itself. The database is
  not capped and will use what it needs.
- **Disk space for your uploads and your backups.** Everything lands in one
  directory you choose, so you can put it on whichever disk has room.
- Images are published for **Intel and ARM**, so a Graviton, Ampere or Apple
  silicon host works the same as an x86 one.

## Install

The full walkthrough for every environment is in **[Deploying
PathLMS](DEPLOYMENT.md)**. The short version:

1. Download the files attached to the [latest
   release](https://github.com/path-lms/pathlms/releases/latest) into an empty
   directory on your server.
2. Follow `INSTALL.md`, which came with them. One command generates every
   password and key, inside the image you are about to run, so nothing has to be
   invented or pasted from anywhere. Three things are yours to decide: the
   address people will type, your email address, and a password you choose.
3. Start it, open your address, and sign in.

About fifteen minutes, most of which is the machine downloading things.

### What a release contains

| File | What it is |
| --- | --- |
| `docker-compose.yml` | The whole stack: seven containers on segmented networks. |
| `pathlms.env` | Every setting you must supply, with this release's three image addresses already filled in and everything else deliberately empty. |
| `INSTALL.md` | Numbered steps from an empty directory to a signed-in administrator. |
| `docker-compose.encryption.yml` | Optional. Add it if you want PathLMS to hold your certificate and answer browsers directly. Leave it out if something in front of it already does. |

Alongside those, a release attaches one short page for each environment that
decides something its own way, so you can read it without leaving the download.

Each release also publishes three container images, each pinned to an exact
cryptographic fingerprint rather than to a tag. A tag can be moved to point at
different bytes after you have read it. A fingerprint cannot, so two
installations of one version are running the same software and can be shown to
be.

**A fresh installation contains nothing.** No sample company, no demo accounts,
no branding but your own. It creates exactly one account, from the email address
and password you choose, and only while there are no accounts at all.

## After it starts

[What to do once it is running](AFTER-IT-STARTS.md) covers the first
administrator, telling PathLMS its own address, and deciding what happens about
encryption.

## Updates

[How updates work](UPDATES.md). A release page, three image addresses, and a
section inside the product that checks, backs up, and records what it did.

## Backups

Backups run on their own from the moment you start it. One a day by default, kept
for fourteen days, into a directory you choose, which is deliberately a separate
setting from the data directory so you can point it at a different disk. The
first one runs immediately rather than after a day, so you find out on day one if
the directory is not writable.

**A backup is only a backup once you have restored one somewhere.** Try it before
you need it.

## Under the hood

| Part | What it is |
| --- | --- |
| Application | Node.js 24 and TypeScript, serving a GraphQL interface |
| Browser | React 19, built with Vite |
| Web server | nginx, holding the security headers and the rate limits |
| Database | PostgreSQL 18, with row-level security enabled and forced on every table |
| Cache | Valkey 8 |
| Files | An S3-compatible object store, running on your own disk |
| Sign-in | Ed25519 signed tokens, Argon2id password hashing, and the two company sign-in standards, OIDC and SAML |
| Running it | Docker Compose. Seven containers on three networks, two of which cannot reach the internet or be reached from it |

The database itself decides who may read which rows, by a large set of rules the
application checks at every start. If any of them is missing or has been
weakened, it refuses to boot rather than serving somebody else's data quietly.
Nobody but the person themselves can rewrite a completion record, and nothing but
a maintenance task can delete one.

## License

**No license has been granted yet, so all rights are reserved.**

The intention is to release PathLMS as open source, and that decision has not
been made. Until it is, nobody has permission to fork this, redistribute it, or
build on it. You may install and run it.

If you need software you can fork and pass on, that is a genuine reason to choose
Moodle, Canvas or Open edX instead, and this page would rather tell you now.

See [LICENSE.md](LICENSE.md).

## Security

If you find a security problem, please report it privately rather than in a
public issue. [SECURITY.md](SECURITY.md) says how.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Bug reports and questions are welcome
now. Code contributions have to wait until the license question above is settled.
