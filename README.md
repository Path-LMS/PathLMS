![PathLMS. A learning path drawn as five courses in order: two finished, the third open and raised above the rest because it is the one being worked through now, and two still to come.](assets/banner.svg)

# PathLMS

**Learning management without the learning-management overhead.**

Modern LMS platforms are often built around the system: layers of configuration, rigid structures, administrative work, and pricing that grows with every learner.

PathLMS is built around the learning instead.

Create courses, organize them into learning paths, enroll people or groups, run quizzes, issue certificates, track progress, and answer the questions that matter. The model stays simple, the interface stays clear, and the administration stays out of the way.

You still get the things an organization needs: branding, reporting, audit history, privacy controls, SSO, backups, and control over where the platform runs and where learner data lives.

**Simple for learners. Efficient for administrators. Yours to run.**

<!-- version -->
**Version 0.102.7** · [Releases](https://github.com/path-lms/pathlms/releases)
<!-- /version -->

**[Download the latest release →](https://github.com/path-lms/pathlms/releases/latest)**

---

## Why PathLMS

PathLMS is designed around the problems that make modern LMS platforms harder than they need to be.

- **Clear structure.** Courses contain modules, sections, lessons, and quizzes. Learning paths put courses in sequence. No extra hierarchy to learn.
- **Less administration.** Organize people into groups, assign training in bulk, and let PathLMS explain anything it could not complete.
- **Useful reporting.** Reports start with the question you are trying to answer: who completed what, who is overdue, how learners are progressing, and more.
- **A better learner experience.** Prerequisites are clear, progress is easy to understand, and the interface stays focused on learning.
- **Your organization, not ours.** Add your logo and brand color while PathLMS keeps the surrounding visual system consistent and accessible.
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

- Create and publish courses directly in the browser.
- Build learning paths and control prerequisites where sequencing matters.
- Run quizzes and keep scores with the learner record.
- Issue printable completion certificates when enabled for a course.
- Organize people into nested groups that reflect real teams, departments, or cohorts.
- Enroll individuals or groups across multiple courses at once.
- Track completions, progress, overdue training, course performance, and group performance.
- Export reports with personally identifying information excluded by default.
- Keep a tamper-evident audit history of significant actions.
- Delete a person's data through the application and see what was removed or retained.
- Brand the experience with your logo and color without rebuilding the interface.
- Use OIDC or SAML with your existing identity provider.

## Who it is for

PathLMS is for organizations that need to deliver and track training without turning learning management into a separate discipline.

It fits onboarding, internal training, compliance programs, professional development, and other structured learning where the priorities are straightforward: publish good material, get it to the right people, understand progress, and keep a reliable record.

It is especially well suited to organizations that want to run their own platform, control their own data, and avoid per-seat licensing.

## Current boundaries

PathLMS is intentionally focused, and the current release has a few limits worth knowing up front:

- Completed courses do not yet reset, expire, or automatically reassign for recurring training.
- Assignment blocks can contain instructions, but learners cannot submit work against them yet.
- Imported course packages from tools such as Articulate or Captivate are not yet supported in a standard release deployment.
- PathLMS runs as a single application instance rather than a clustered service.
- Built-in MFA is not included; use OIDC or SAML with an identity provider that provides it.
- WCAG 2.2 AA is the accessibility target, not a certification claim.

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
