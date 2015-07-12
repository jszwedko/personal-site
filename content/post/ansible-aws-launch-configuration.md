---
title: "Using Ansible to provision AWS autoscaling instances"
date: "2015-07-11T19:15:44-07:00"
tags:
- aws
- ansible
- autoscaling

---

## Problem statement:

If you are running your infrastructure on EC2 in AWS, you are probably using
[AWS's autoscaling service](http://aws.amazon.com/autoscaling/) to manage your
instances. If you are not, you should be. Even if you don't plan to have your
instances scale up and down based on traffic patterns, you'll want to use
autoscaling for the simple case of keeping a certain number of instances
running if one of them stops responding or is decommissioned by AWS -- without
requiring manual intervention by an engineer to spin up and configure a new
server.

The tricky bit is making sure that the new instances that are created are
configured properly for their purpose. For Cassandra, for example, this might
mean pointing the configuration at the seed nodes for your cluster. For a web
server this may mean configuring which database to point to and credentials for
connecting. Either way it is rare that your server will need no configuration
at launch time to operate properly (even though you should attempt to embed as
much configuration as possible into your AMIs). AWS's solution for this is
user-data, a script (or cloud-init file) that is executed on first boot.

The simplest thing you can do is provide a bash script, e.g.

```bash
#!/bin/bash

aws ec2 describe-instances \
  --filters "Name=tag:Database,Values=true" \
  --query 'Reservations[*].Instances[*].PrivateDnsName' \
  --output text \
  > /etc/databases

systemctl start my-webservice
```

This is a contrived example, but I hope it is illustrative.

At Braintree, as our user-data became more complex (things like configuring
firewall rules, setting up certificates and configuring services like
exhibitor), we made the decision to migrate from bash scripts to
[Ansible](http://www.ansible.com/home) configuration to better organize it and
take advantage of [jinja2](http://jinja.pocoo.org/docs/dev/) templating.

If you are familiar with Ansible, you can probably guess that the difficulty
with using Ansible as user-data is that Ansible configuration typically
consists of a playbook and all of its supporting assets (templates, static
files, roles, etc.) which doesn't neatly fit into a single file (AWS also
limits user-data to 16 KB) so it may seem like your only option is to store the
configuration elsewhere (like S3) and retrieve it in your user-data or bake it
into your image (slow turn around time for changes), but below I'll describe
another option that ended up working for us.

## Solution:

Compress and base64 encode your Ansible configuration! It may seem like
a simple idea, but it didn't immediately occur to us so I thought I would share
it here. I also think this technique could be used for other standalone
configuration management tools like
[`chef-solo`](https://docs.chef.io/chef_solo.html) or even just modularized
bash scripts.

We use Terraform to manage our infrastructure, so we have this compression and
base64 encoding happen as a preprocessing step when planning (using a wrapper
script around `terraform plan`) that is used by our launch configuration
resources, but you could fit it into your workflow however works best.

Example script you can use to automate creation of the user-data script:

```sh
#! /bin/sh

ansible_configuration=$1
shift

cat <<-EOF
#! /bin/bash

set -o errexit

mkdir -p /tmp/ansible
echo '$(tar c -C "$ansible_configuration" . | gzip -n | base64 -w 0)' | base64 -d | tar xz -C /tmp/ansible
cd /tmp/ansible
EOF

echo -n '/usr/local/bin/ansible-playbook playbook.yml --connection=local -i localhost, -e target=localhost'
```

Note that this script assumes that you have a variable called `target` that you
use for your `hosts` declaration in your playbook.

Example of generated user-data (base64 encoded directory shortened for brevity):

```bash
#! /bin/bash

set -o errexit

mkdir -p /tmp/ansible
echo 'H4sIAAAAAAAAA+w9aXPbRpb+[...]+CV+qgxijv8ggqkPOrhZngzP99N/Tv0z+6bZbGw3u9QXa2v1ovOfw0nfP/yPxjlBAKBQCAQCAQCgUAgEAgEAoHg+eEnXr53cwCgAAA=' | base64 -d | tar xz -C /tmp/ansible
cd /tmp/ansible
/usr/local/bin/ansible-playbook playbook.yml --connection=local -i localhost, -e target=localhost
```

This is what you would use for the user-data in your launch configuration
whereby, on launch, the encoded Ansible configuration is decoded, written
out to `/tmp/ansible`, and `ansible-playbook` executed.
