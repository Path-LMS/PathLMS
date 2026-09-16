![PathLMS. A learning path drawn as five courses in order: two finished, the third open and raised above the rest because it is the one being worked through now, and two still to come.](assets/banner.svg)

# PathLMS

**Learning management without the learning-management overhead.**

Modern LMS platforms are often built around the system: layers of configuration, rigid structures, administrative work, and pricing that grows with every learner.

PathLMS is built around the learning instead.

Create courses, organize them into learning paths, enroll people or groups, run quizzes, issue certificates, track progress, and answer the questions that matter. The model stays simple, the interface stays clear, and the administration stays out of the way.

You still get the things an organization needs: branding, reporting, audit history, privacy controls, SSO, backups, and control over where the platform runs and where learner data lives.

**Simple for learners. Efficient for administrators. Yours to run.**

<!-- version -->
**Version 0.119.0** · [Releases](https://github.com/path-lms/pathlms/releases)
<!-- /version -->

**[Download the latest release →](https://github.com/path-lms/pathlms/releases/latest)**

---

## Why PathLMS

PathLMS is designed around the problems that make modern LMS platforms harder than they need to be.

- **Clear structure.** Courses contain modules, sections, lessons, topics and quizzes. Learning paths put courses in order. There is nothing else to learn.
- **Less administration.** Organize people into groups, assign training in bulk, and let PathLMS explain anything it could not complete.
- **Useful reporting.** Reports start with the question you are trying to answer: who completed what, who is overdue, how learners are progressing, and more.
- **A better learner experience.** Prerequisites are clear, progress is easy to understand, and the interface stays focused on learning.
- **Your organization, not ours.** Add your logo, brand color and tab icon. PathLMS keeps the rest of the screen readable and easy to use.
- **Your environment and your data.** Run PathLMS on your own infrastructure, keep learner data local, and integrate with OIDC or SAML for company sign-in.
- **No per-user pricing.** Access does not become more expensive simply because more people need training.

## PathLMS in action

![The administrator home page, with a Get Started panel reading 0 of 5 complete above five unchecked setup steps, from setting up groups to enrolling learners.](images/the-first-screen.png)

*Start with a short guide or ignore it and go directly where you need to go.*

![The reports screen: seven cards, each headed with a question such as Who completed what? or Who's overdue?, each with a View Report button.](images/reports-are-questions.png)

*Reports begin with real questions, not a query builder.*

![The Enroll people dialog at step one of three, People then Courses then Result, with a toggle between choosing individual people and whole groups.](images/enrolling-a-group.png)

*Enroll individuals or whole groups in one pass.*

![A lesson page. The word Extraction is lightly underlined, and a small panel beside it, labeled What it means on this course, gives the meaning that applies here.](images/a-term-in-a-lesson.png)

*Course-specific glossary terms give learners the meaning that matters in context.*

*These are screenshots of the software running.*

## What you can do

- **Learning paths at the center.** Put courses in order, enroll a whole group once, and PathLMS keeps everyone's progress right when the path changes.
- **Your servers, your data, no per-user fees.** Install it yourself. Backups run on their own, and updates happen when you choose.
- **A record nobody can quietly change.** Important actions are logged, and the log shows if anyone tampers with it.
- **Repeating training that runs itself.** Yearly training goes out again on time. People are warned before it goes out of date. You can see who is out of date.
- **Reports that answer questions.** See who finished what, who is overdue, and how each course and group is doing. Exports leave out names and emails unless you ask.
- **Privacy you can act on.** Delete a person's data from inside PathLMS. See exactly what was removed and what was kept.
- **Enroll whole groups at once.** Groups sit inside groups to match your organization. Everyone gets all the courses they need in one step.
- **Build courses in the browser.** Add lessons, topics, quizzes and certificates. No separate authoring tool needed.
- **Company sign-in.** People use the account they already have, through OIDC or SAML.

See **[everything PathLMS can do](FEATURES.md)**.

## Who it is for

PathLMS is for organizations that need to deliver and track training without turning learning management into a separate discipline.

It fits onboarding, internal training, compliance programs, professional development, and other structured learning where the priorities are straightforward: publish good material, get it to the right people, understand progress, and keep a reliable record.

It is especially well suited to organizations that want to run their own platform, control their own data, and avoid per-seat licensing.

## Current boundaries

A few things PathLMS does not do yet:

- Learners cannot hand in work for an assignment. An assignment can hold instructions only.
- Course packages made in tools such as Articulate or Captivate cannot be used in a standard installation.
- PathLMS runs as one copy on one server. It cannot be spread across several servers.
- There is no built-in second sign-in step, such as a code from a phone. Use OIDC or SAML with a sign-in service that has one.
- PathLMS is built to meet WCAG 2.2 AA, the usual accessibility standard. It has not been certified.

These are product boundaries, not features held back for a paid edition.

## Run it your way

PathLMS is distributed as a Docker Compose deployment for Intel and ARM systems. A small installation is comfortable with two CPU cores and 4 GB of memory.

Backups run automatically, updates remain administrator-controlled, and HTTPS can be handled by your existing reverse proxy or by PathLMS itself.

See **[Deploying PathLMS](DEPLOYMENT.md)** for the full installation guide and **[After it starts](AFTER-IT-STARTS.md)** for first-run configuration.

[How updates work](UPDATES.md) covers what an update costs, how you are told one exists, and what the Updates section inside the product does.

## Under the hood

| Part | What it is |
| --- | --- |
| Application | Node.js 24 and TypeScript, serving a GraphQL interface |
| Browser | React 19, built with Vite |
| Web server | nginx with security headers and rate limits |
| Database | PostgreSQL 18 with enforced row-level security |
| Cache | Valkey 8 |
| Files | S3-compatible object storage on your own disk |
| Sign-in | Ed25519 tokens, Argon2id passwords, OIDC and SAML |
| Deployment | Docker Compose across isolated container networks |

Security controls are enforced below the interface as well as inside it. PathLMS checks its database access policies at startup and refuses to serve if those protections are missing or weakened.

## License

**PathLMS is not open source yet.**

You may install and run the published release, but the current license does not permit forking, redistribution, or derivative works. See [LICENSE.md](LICENSE.md).

## Security

Please report security issues privately rather than opening a public issue. See [SECURITY.md](SECURITY.md).

## Contributing

Bug reports and questions are welcome. Code contributions will open when the licensing decision is settled. See [CONTRIBUTING.md](CONTRIBUTING.md).
