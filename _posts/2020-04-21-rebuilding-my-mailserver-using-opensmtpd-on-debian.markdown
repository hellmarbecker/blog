---
layout: post
title:  "Rebuilding my mail server using OpenSMTPD on Debian"
categories: mail linux blog
---
So, I had been running my own mail server for more than 10 years. The
configuration had degraded over time and been propagated through several
major OS upgrades. Security features were not up to date and I had lost
track of some of the changes I made years ago. It was time to tear down
the entire server and rebuild it from scratch.

A while ago I came across [this article](https://poolp.org/posts/2019-08-30/you-should-not-run-your-mail-server-because-mail-is-hard/) by Gilles Chehade (I think it was featured on [Hacker News](https://news.ycombinator.com/news)).
In his blog, Gilles included [a step-by-step explanation](https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/) how to set up a mail server using OpenSMTPD, which he maintains.

[OpenSMTPD](https://www.opensmtpd.org/) is really great, it offers up-to-date
security features and its configuration is much simpler than for other SMTP servers. Alas,
things are not as easy as they look. OpenSMTPD has been developed fo OpenBSD, and while
there are ports to Linux, getting the latest stack up and running is not totally trivial.

## The First Attempt

I thought that CentOS would be a good platform to deploy to. However, the latest
version my hosting provider supports is CentOS 7. And for CentOS 7 there is no
prebuilt package with the latest version of OpenSMTPD. Also, there is no supported
upgrade path from CentOS 7 to 8.

Fine, so I will build it myself. I installed all the usual development tools, cloned the repo
and did the usual incantations. Turns out I needed to install some missing Perl packages, so I needed
CPAN too.

And then the build failed because OpenSMTPD 6.6 requires OpenSSH 1.1.1, which is not included with 
CentOS 7. So I started building that one from source, too. There is an instruction
[here](https://www.hostnextra.com/kb/how-to-install-openssl-1-1-1d-in-centos/) but then of course, 
version 1.1.1d has a bug that makes the tests fail, so I had to upgrade to 1.1.1e.

After all the binaries has been built, it still didn't work. I needed to manually create
some specific users and groups for OpenSMTPD.

Fixed some default paths and it worked - kind of. The entire thing felt a bit like a Frankenstein
setup. Also, for each upgrade I would have to do the same steps again, and maybe I would
have another broken dependency ...

That's where I decided to scrap it and try again. Time to move to the [Debian](https://www.debian.org/)
platform. Maybe that's going to work better?

## Switching to Debian

The starting situation seemed not to be much better. My provider offers Debian 9 Stretch, but
I need at least Debian 10 Buster. There is, however, a defined path to upgrade
Debian to the next major version, and it is as simple as changing the list of
repositories in `/etc/apt/sources.list`, then upgrade the system. This part 
went smooth and easy.

Alas, the OpenSMTPD package that comes with Debian Buster is version 6.0.0, but I wanted 6.6.0.
Some Googling revealed that you can get newer versions of packages by adding the
`buster-backports` repository. So I added the line

    deb http://deb.debian.org/debian buster-backports main

to my `sources.list` and then installed OpenSMTPD using

    apt-get -t buster-backports install opensmtpd

And voilà, I have a nice, working mail server. I used Gilles's instructions for the rest, so I got

- dovecot for my IMAP server
- rspamd to filter spam

The walkthrough in Gilles's blog is really detailed and can easily adapted to Linux. Really great work!

In addition, I installed `ufw` for a firewall, because I like things easy.

## Learnings

  - I really like OpenSMTPD. I used exim4 before and I find that OpenSMTPD is much easier to configure.
  - Debian over CentOS. The ease of upgrade convinced me once again. And I haven't even touched upon the
    little things like the much better management of virtual Apache servers in Debian.

