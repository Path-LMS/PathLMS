# Everything PathLMS can do

This page lists what PathLMS does today, grouped by what you are trying to get
done. The [README](README.md) has the short version.

---

## Build courses

- Write and publish courses in the browser, with no separate authoring tool.
- Break a course into modules, sections, lessons and quizzes.
- Add topics anywhere in a course: inside a lesson, before one, or inside another topic. Each topic holds its own content, and learners get credit for finishing it.
- Build a whole course using only the keyboard.
- Run quizzes and keep each score on the learner's record.
- Give out printable certificates for any course that has them turned on.
- Put courses in order as a learning path, and choose which ones must be done first.
- Add a word list to a course or learning path. Learners see what a word means in that course, right where it appears.
- Copy a word list in from a Moodle file in a few minutes. You see every word before anything is added.

## Get training to people

- Put people into groups, and groups inside groups, to match your teams and departments.
- Enroll people or whole groups in several courses at once. PathLMS tells you about anyone it could not enroll, and why.
- Let people sign themselves up from the catalog.
- Hide a course from the catalog while it stays published.
- Stop people signing themselves up for a course, so only an administrator or manager can enroll them.

## Keep required training current

- Set up training that must be done again, such as every year. PathLMS gives it out again by itself, to everyone or to one team. You see who will get it before you save.
- Warn people once before their training goes out of date, at a normal hour where they live.
- Set a finish-by day. When it passes, an unfinished attempt closes.
- Let a learner whose attempt closed press Start again.
- See every required training on the People page, with who is up to date, who is due again and who is out of date.

## Know where things stand

- Open reports that start with a question: who finished what, who is overdue, how learners are getting on, and how each course and group is doing.
- List the people who did not finish in time.
- Export reports as CSV, Excel or PDF. Names and email addresses are left out unless you ask for them.
- Keep a record of important actions that shows if anyone tampers with it.

## Look after people's data

- Delete a person's data from inside PathLMS, and see what was removed and what was kept.
- Keep all learner data on servers you choose.

## Make it yours

- Add your logo and brand color. PathLMS keeps the rest of the screen readable.
- Choose a background for the sign-in page.
- Add a browser tab icon, shown in tabs, bookmarks and on phone home screens.

## Sign-in and security

- Let people sign in with the company account they already have, through OIDC or SAML.
- Give each person one of five roles: administrator, manager, instructor, author or learner.

## Run it yourself

- Install it with Docker Compose on Intel or ARM servers.
- Back up automatically, and restore from a backup.
- Update when you choose, from a button.
- Use your own web proxy for HTTPS, or let PathLMS handle it.

---

## What PathLMS does not do yet

- Learners cannot hand in work for an assignment. An assignment can hold instructions only.
- Course packages made in tools such as Articulate or Captivate cannot be used in a standard installation.
- PathLMS runs as one copy on one server. It cannot be spread across several servers.
- There is no built-in second sign-in step, such as a code from a phone. Use OIDC or SAML with a sign-in service that has one.
- PathLMS is built to meet WCAG 2.2 AA, the usual accessibility standard. It has not been certified.

These are limits of the product today. None of them is held back for a paid edition.
