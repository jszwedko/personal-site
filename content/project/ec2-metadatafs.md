---
date: 2016-02-14
tags:
- golang
- aws
- fuse
title: ec2-metadatafs
download_url: https://github.com/jszwedko/ec2-metadatafs/releases
project_description: EC2 metadata and tags as a filesystem
project_name: ec2-metadatafs
project_url: https://github.com/jszwedko/ec2-metadatafs
release_date: 2016-02-14
version: 0.2.0
---


[`ec2-metadatafs`](https://github.com/jszwedko/ec2-metadatafs) is
a [FUSE](https://github.com/libfuse/libfuse) filesystem that exposes AWS EC2
instance metadata and tags in the form of a directory tree.

<!--more-->

When writing software that runs on AWS instances and interacts with AWS
services, it is common to need metadata about the instance (e.g. the instance
ID, local IP address, or host region) to vary behavior about code on the
instance. This is provided by AWS via an internal HTTP service at accessible at
a static IP address which you can execute HTTP GET requests against. In
addition, often custom information is encoded as tags on the instance (e.g. the
role of the instance or the user that created the instance).

AWS provides SDKs for many languages that can access this information, but for
bash scripts, languages without support, or simply when debugging via SSH from
within the instance itself you are left either:

0. `curl`ing the information which means:
  * trying to remember the special static IP address or [looking it
    up](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
  * trying to remember the path of the information or [looking it
    up](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
0. using the [ec2-metadata tool](https://aws.amazon.com/code/1825) which:
  * provides only a limited subset of the metadata
  * requires use of `cut` to extract the field without the label

Additonally, the metadata endpoint (and thus the `ec2-metadata` tool) [does not
contain instance
tags](https://forums.aws.amazon.com/thread.jspa?threadID=88389) which requires
further queries using the [AWS CLI tool](https://aws.amazon.com/cli/) to
discover.

To simply all of this, I took a page one of the tenants of the Unix philosophy,
["In UNIX Everything is
a File"](https://web.archive.org/web/20120320050159/http://ph7spot.com/musings/in-unix-everything-is-a-file),
and decided to expose all of this information as a filesystem which can be
easily explored and interrogated using traditional UNIX tooling such as `grep`,
`cat`, and `ls`. This is done via
[`libfuse`](https://github.com/libfuse/libfuse) which exposes an interface for
writing filesystems in userspace (I use [go-fuse](github.com/hanwen/go-fuse)
which exposes Go bindings for `libfuse`).

## Installing

Install `fuse` using your *nix distro's package manager.

Download the binaries from the [releases
page](https://github.com/jszwedko/ec2-metadatafs/releases) and place them in
your `$PATH`.

Add to `/etc/fstab` if you'd like to mount the filesystem on boot via:

```
ec2-metadatafs /aws fuse _netdev,allow_other,tags 0 0
```

Leave out the `tags` option if you'd like to only mount the metadata
filesystem. Note that the `tags` option requires valid AWS credentials (see
`ec2-metadatafs --help`).

See the project
[README.md](https://github.com/jszwedko/ec2-metadatafs/blob/master/README.md)
for the most up-to-date usage information and additional options.

## Example

```
$ mkdir /tmp/aws
$ ec2-metadatafs --tags /tmp/aws
$ tree /tmp/aws
/tmp/aws
├── dynamic
│   └── instance-identity
│       ├── document
│       ├── dsa2048
│       ├── pkcs7
│       ├── rsa2048
│       └── signature
├── meta-data
│   ├── ami-id
│   ├── ami-launch-index
│   ├── ami-manifest-path
│   ├── block-device-mapping
│   │   ├── ami
│   │   └── root
│   ├── hostname
│   ├── iam
│   │   ├── info
│   │   └── security-credentials
│   │       └── test
│   ├── instance-action
│   ├── instance-id
│   ├── instance-type
│   ├── local-hostname
│   ├── local-ipv4
│   ├── mac
│   ├── metrics
│   ├── network
│   │   └── interfaces
│   │       └── macs
│   │           └── 06:5e:69:f7:53:ed
│   │               ├── device-number
│   │               ├── interface-id
│   │               ├── local-hostname
│   │               ├── local-ipv4s
│   │               ├── mac
│   │               ├── owner-id
│   │               ├── security-group-ids
│   │               ├── security-groups
│   │               ├── subnet-id
│   │               ├── subnet-ipv4-cidr-block
│   │               ├── vpc-id
│   │               └── vpc-ipv4-cidr-block
│   ├── placement
│   │   └── availability-zone
│   ├── profile
│   ├── public-keys
│   │   └── 0
│   │       └── openssh-key
│   ├── reservation-id
│   ├── security-groups
│   └── services
│       └── domain
│           └── amazonaws.com
├── tags
│   ├── createdBy
│   ├── name
│   └── role
└── user-data

16 directories, 42 files
$ cat /tmp/aws/meta-data/instance-id
i-123456
$ cat /tmp/aws/user-data
#! /bin/bash
echo 'Hello world'
$ cat /tmp/aws/tags/name
My Instance Name
```

The mounting of tags requires AWS API access (and valid credentials) so is not
mounted by default, but can be mounted via the `--tags` parameter.
