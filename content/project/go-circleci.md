---
date: 2015-08-15
tags:
- golang
- circleci
title: go-circleci
download_url: https://github.com/jszwedko/go-circleci
project_description: Go library for interacting with CircleCI
project_name: go-circleci
project_url: https://github.com/jszwedko/go-circlecli
release_date: 2015-08-15
version: 0.1.0

---

[`go-circleci`](https://github.com/jszwedko/go-circleci) is a Go library that
I wrote to wrap [CircleCI's API](https://circleci.com/docs/api). It can be used
by other Go projects that wish to interact with the API.

<!--more-->

[CircleCI](https://circleci.com) is a continuous integration service that you
can use with your GitHub projects to automatically build and test your code.

The library API itself mirrors closely the endpoints exposed by CircleCI, you
can see the [GoDoc](http://godoc.org/github.com/jszwedko/go-circleci) page for
the latest documentation.

## Installing

- Set up your [Go build environment](https://golang.org/doc/code.html)
- `go get github.com/jszwedko/go-circleci`
- In your project, import the library via `import "github.com/jszwedko/go-circleci"`
