
---
date: 2015-08-15
tags:
- golang
- circleci
- cli
title: circleci-cli
download_url: https://github.com/jszwedko/circleci-cli/releases
project_description: CLI tool for interacting with CircleCI
project_name: circleci-cli
project_url: https://github.com/jszwedko/circleci-cli
release_date: 2015-08-15
version: 0.0.1

---

[`circleci-cli`](https://github.com/jszwedko/circleci-cli) is a CLI too that
allows you to interact with CircleCI without leaving the comfort of your
terminal.

<!--more-->

[CircleCI](https://circleci.com) is a continuous integration service that you
can use with your GitHub projects to automatically build and test your code.
To help you automate tasks, they expose an [HTTP + JSON
API](https://circleci.com/docs/api) that you can interact with. `circleci-cli`
wraps this API in a nice interface so that you can view, retry, cancel builds
and more!

## Installing

Download the binaries from https://github.com/jszwedko/circleci-cli/releases and
place them in your `$PATH`.

Generate an API token via the [settings page](https://circleci.com/account/api).

## Examples

```
$ export CIRCLE_TOKEN=XXXXX # token generated above
$ circleci projects -v
jszwedko/go-circleci
master*  success

jszwedko/circleci-cli
master*  failed

$ circleci recent-builds -a
jszwedko/circleci-cli/2    failed     master    Fix wording in README
jszwedko/go-circleci/3     success    master    Fix link
jszwedko/go-circleci/2     success    master    Add build badge
jszwedko/circleci-cli/1    failed     master    Add link to releases page
jszwedko/go-circleci/1     success    master    Update README for new location

$ circleci show -p jszwedko/go-circle-ci -n 2 # try running with -v
Subject           Add build badge
Trigger           github
Author            Jesse Szwedko
Committer         Jesse Szwedko
Status            success
Build Parameters
                  None
Started           2015-08-14 21:33:44.678 +0000 UTC
Duration          28.604s

Build 0
* Starting the build (success) (770ms)

* Start container (success) (5.289s)

* Enable SSH (success) (1.468s)

* Restore source cache (success) (1.448s)

* Checkout using deploy key: 37:27:f7:68:85:43:46:d2:e1:30:83:8f:f7:1b:ad:c2 (success) (2.378s)

* Configure the build (success) (80ms)

* Restore cache (success) (3.783s)

* rm -rf $HOME/.go_workspace (success) (726ms)

* Save cache (success) (674ms)

* go vet ./... (success) (1.033s)

* go test ./... (success) (1.428s)

* Collect test metadata (experimental). (success) (1.568s)

* Collect artifacts (success) (6.515s)

* Disable SSH (success) (8ms)

$ circleci -h
NAME:
   circleci - Tool for interacting with the CircleCI API

USAGE:
   circleci [global options] command [command options] [arguments...]

VERSION:
   0.0.1-2-gcad9be7 (cad9be7f983569548d473ef5335a3a25fe2e425b)

COMMANDS:
   projects                     Print projects
   recent-builds, recent        Recent builds for the current project
   show                         Show details for build
   list-artifacts, artifacts    Show artifacts for build (default to latest)
   test-metadata                Show test metadata for build
   retry-build, retry           Retry a build
   cancel-build, cancel         Cancel a build
   build                        Trigger a new build
   clear-cache                  Clear the build cache
   add-env-var                  Add an environment variable to the project (expects the name and value as arguments)
   delete-env-var               Add an environment variable to the project (expects the name as argument)
   add-ssh-key                  Add an SSH key to be used to access external systems (expects the hostname and private key as arguments)
   help, h                      Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --host, -H "https://circleci.com"    CircleCI URI [$CIRCLE_HOST]
   --token, -t                          API token to use to access CircleCI (not needed for displaying information about public repositories) [$CIRCLE_TOKEN]
   --help, -h                           show help
   --version, -v                        print the version
```

The exact commands and output may very with updates, so please see the
[README](https://github.com/jszwedko/circleci-cli) for the most up-to-date
documentation.
