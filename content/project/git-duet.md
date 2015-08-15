---
date: 2015-05-30
tags:
- golang
- git
- pairing
title: git-duet
download_url: https://github.com/git-duet/git-duet/releases
project_description: Productivity tool for pairing using Git without losing your identity
project_name: git-duet
project_url: https://github.com/git-duet/git-duet
release_date: 2015-05-30
version: 0.1.0

---

[`git-duet`](https://github.com/git-duet/git-duet) makes use of the committer
field of `git` commits to allow paired pairing while maintaining identity.

<!--more-->

Traditionally, pairing programmers have to settle for either:

- Only associating one of their names and email addresses with the commit
- Abusing `user.name` and `user.email` to create an amalgamation of their
  names and e-mails (this is what many other tools do, such as
  [git-pair](https://github.com/pivotal/git_scripts)). Resulting in commits
  that are attributed to a fictitious identity such as `Josh Susser & Sam
  Pierson <pair+jsusser+sam@pivotallabs.com>`

`git-duet` takes a different approach and sets the author and the committer on
a commit allowing two individuals to associate their real names and e-mails
with a commit resulting in commits like:

```
$ git show --pretty=fuller
commit 54486ba89913e209f00903bdef34fc38e61edeb0
Author:     Jane Doe <jane.doe@example.com>
AuthorDate: Sat Jun 20 17:55:19 2015 -0700
Commit:     John Doe <john.doe@example.com>
CommitDate: Sat Jun 20 17:55:19 2015 -0700

Paired commit

Signed-off-by: John Doe <john.doe@example.com>

diff --git a/foo b/foo
new file mode 100644
index 0000000..e69de29
```

This is a port of https://github.com/meatballhat/git-duet to Golang because
this was found to be more performant and have less issues during usage due to
Ruby version managers like `rbenv` and `rvm` that frequently complain that
`git-duet` is not installed in a given version of Ruby when not using the
system Ruby. Many kudos to [@meatballhat](https://github.com/meatballhat) for
the initial implementation.

## Installing

Download the binaries from https://github.com/git-duet/git-duet/releases and
place them in your `$PATH`.

## Usage

Create a `~/.git-authors` with your authors:

```yaml
authors:
  jd: Jane Doe; jane
  fb: Frances Bar
  email:
    domain: awesometown.local
```

Use `git duet <author initals> <committer initials>` to set the author and
committer.

Use `git duet-commit` to commit using the set author and committer (I recommend
aliasing this, e.g. `git dci`).

Use `git solo jd` to erase the committer and just set the author (for when you
aren't pairing).

Example:
```bash
$ mkdir your-repo
$ cd your-repo
$ git init
$ touch README.md
$ git add -A
$ git duet jd fb
$ git duet-commit -m 'Initial commit'
$ git show --pretty=fuller
```

See [README.md](https://github.com/git-duet/git-duet/) for more configuration
options.

Admittedly this is still a hack to fit the concept of pairing on code into the
constraints of the metadata `git` allows to be associated with a commit, but
I find that this hack is better than combining names and emails into the author
of the commit for the following reasons:

- Can easily search for commits that a given user has touched
- Tools like GitHub are able to hyperlink commits to users and notify the
  appropriate user when a commit has been commented upon.

Eventually `git` may support the notion of multiple authors (see:
https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=451880) at which point
`git-duet` will change to utilize this.
