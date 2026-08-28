# Reporting a security problem

**Please report it privately, not in a public issue.** Somebody's live
installation should be able to be fixed before the details are public.

Use GitHub's private vulnerability reporting on this repository: open the
**Security** tab and choose **Report a vulnerability**. That opens a private
conversation visible only to the maintainer.

If that is not available to you, open a public issue saying only that you have
found a security problem and asking for a private channel. Do not put the details
in it.

## What helps

- **What an attacker could do**, in plain terms. That is the part that decides how
  urgent this is.
- **How to reproduce it**, in enough detail to see it happen.
- **The version** you found it on.
- **Whether it needs an account**, and if so which role.

Take your time over the report rather than rushing it. A clear one is worth more
than a fast one.

## What to expect

This is a small project with one maintainer and **no support contract and no
guaranteed response time.** That is stated plainly here because it is better to
know than to wait.

What you can expect is that a report will be read, that you will be told what is
being done, and that you will be credited when a fix ships unless you would
rather not be.

## Which versions are looked at

**The most recent release.** There is no long-term support branch, so the answer
to almost every security problem is a new release, and the way you get it is to
move to it. [How updates work](UPDATES.md) covers that.

## Things that are already known and are not findings

These are documented limits rather than discoveries. Reporting them is welcome,
and you will get this page back.

- **Traffic is not encrypted** until somebody sets that up. A fresh installation
  holds no certificate. Either something in front of it holds one, or you add the
  one extra file a release attaches and PathLMS holds it itself.
- **The session token is readable in browser storage.** This is a consequence of
  the point above; on a deployment reachable over an unencrypted connection, it
  is also readable in transit.
- **PathLMS runs as one copy** and cannot be scaled out.
- **A password check reaches a public breach service.** The first five characters
  of a hash are sent and nothing else.
- **A version check reaches a public container registry**, carrying nothing about
  the installation, and it can be switched off.
- **Packaged courses from another authoring tool do not play** on a release
  installation. The route that would serve them checks the administrator's
  setting before anything else and answers nothing while it is off.

All of these are covered in more detail in the [README](README.md).

## Please do not

- Test against somebody else's installation.
- Run automated scanners against infrastructure you do not own.
- Publish the details before there is a fix people can install.
