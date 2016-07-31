---
layout: post
title: Signing Commits on Github with key from Keybase.io (OS X)
date: '2016-07-30T23:47:00.001-05:00'
author: Phillip Green II
tags:
- security
- keybase.io
- github
- signature
- osx
modified_time: '2016-06-03T23:47:00.001-05:00'
---

I start this post of with a confession.  I have never used [GPG][gpg].  I never discounted its usefulness, but never took the time start using it.  I recently was sent my invite for [Keybase.io][keybase-io] and I want to start using it.  In this post, I am describing what I had to do in order to start signing my git commits and have them show up correctly in [Github][github].

## Pre-requisites

To get started, you need some pre-requisites:

 * A [Github account][github-join]
 * A [Keybase.io Account][keybase-io]
   * At the time of this writing, there is a limited access, [message me on twitter][twitter-phillipgreenii] and I'll send you one of my invites.
   * When setting up your account, be sure to include your email addresses you have [added in github][github-associated-email-addresses].
 * [GPG][gpg] installed
   * I'm running El Capitan, so I went with [GPG Tools][gpg-tools].
 * [pinentry][pinentry] (optional)
   * I use [MacPorts][macports] to install `pinentry-mac`.

## Instructions

### Add keybase key to GPG keychain

I first followed the steps from [How to import Keybase private key to use locally with GPG][import-keybaseio-gpg] to export my private key from keybase and import it into my gpg keychain.  I cannot confirm step 3 because I used the GUI instead of the command line, but I expect both are the same.  The end goal is to have your GPG key from keybase installed locally.  Once it is in there, be sure to set the `ownertrust` to `Ultimate`.  I'm sure there is a command to do this, but I did it from the GUI under details.  If this is not set, git will print warnings for your signatures:
```
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
```

### Add Key to github

Next I jumped over to [Github GPG + Keybase PGP][github-gpg-keybase-pgp].  If you already added your email addresses to your keybase key during creation, you can skip the first three steps and jump right into adding your GPG key into github.  

### Enable sign for all commits

I then followed the steps to associate my key with my repository and set it to always sign during commits:

```
git config user.signingkey phillip.green.ii@gmail.com
git config commit.gpgsign true
```

I used the "per repository" configuration, but like most everything in git, you can use the `--global` flag to set those values globally.

### Configure GPG Agent to manage the passphrase

If you stop now, you will be prompted to type in your keybase.io passphrase each time you sign a commit.  However, you can configure the `gpg-agent` to handle the passphrase.  I followed the steps from [Automatic Git commit signing with GPG on OSX][atomatic-git-sign-with-gpg-osx] and had only one problem.

I used [MacPorts][macports] to install `pinentry-mac` and unlike what is specified in the gist, the install path for me was different.  I used the following configuration for `gpg-agent.conf`

```
default-cache-ttl 600
max-cache-ttl 7200

pinentry-program /Applications/MacPorts/pinentry-mac.app/Contents/MacOS/pinentry-mac
```

## Try it out

To ensure a valid test, I first ran `killall -9 gpg-agent` to ensure no pre-existing agents were running.  Then I opened a fresh terminal, to ensure the scripts would run.  Finally, I made my first commit.  The agent prompted me for my passphrase.  I only need to type it in once because it will be managed by my keychain now.  To ensure things were well, I ran `git log --show-signature`.

```
commit e1f0697c2181dc33c5cb18df871384f795c1286b
gpg: Signature made Sat Jul 30 21:12:25 2016 EDT using RSA key ID 31F122EF
gpg: Good signature from "Phillip Green II <phillip.green.ii@gmail.com>" [ultimate]
gpg:                 aka "Phillip Green II <deathcheese@yahoo.com>" [ultimate]
Author: Phillip Green II <phillip.green.ii@gmail.com>
Date:   Sat Jul 30 21:11:18 2016 -0400

    add keybaseio
```

After pushing to github, I was met with the following:

![Verified Github Commit][img-verified-github-commit]

All in all, there isn't  much to it.  I followed a couple blogs with only minor tweaks and now I can sign all of my git commits.

## References
* [How to import Keybase private key to use locally with GPG][import-keybaseio-gpg]
* [Github GPG + Keybase PGP][github-gpg-keybase-pgp]
* [Automatic Git commit signing with GPG on OSX][atomatic-git-sign-with-gpg-osx]


[atomatic-git-sign-with-gpg-osx]: <https://gist.github.com/bmhatfield/cc21ec0a3a2df963bffa3c1f884b676b> "Automatic Git commit signing with GPG on OSX"
[github]: <https://github.com/> "GitHub"
[github-associated-email-addresses]: <https://github.com/settings/emails> "Associated Email Addresses"
[github-join]: <https://github.com/join> "Join GitHub"
[github-gpg-keybase-pgp]: <https://www.ahmadnassri.com/blog/github-gpg-keybase-pgp/> "Github GPG + Keybase PGP"
[gpg]: <https://www.gnupg.org/> "GnuPG"
[gpg-tools]: <https://gpgtools.org/> "GPG Tools"
[import-keybaseio-gpg]: <http://www.keybits.net/2016/02/import-keybase-private-key/> "How to import Keybase private key to use locally with GPG"
[keybase-io]: <https://keybase.io/> "Keybase.io"
[macports]: <https://www.macports.org/> "MacPorts"
[pinentry]: <https://www.gnupg.org/related_software/pinentry/index.en.html> "Gnu Pinentry"
[twitter-phillipgreenii]: <https://twitter.com/phillipgreenii> "phillipgreenii (twitter)"


[img-verified-github-commit]: <{{ site.baseurl }}/images/signing-commits-on-github-with-keybaseio/verified-github-commit.png> "Verified Github Commit"
