---
title: "AWS Autoscaling Ansible Inventory"
tags:
- ansible
- aws
- autoscaling
date: "2015-08-15T13:16:22-07:00"

---

As described in a [previous post]({{<ref
"/post/ansible-aws-launch-configuration.md">}}), at
[Braintree](http://braintreepayments.com/), we use Ansible as the [user
data](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) in the
launch configurations (LC) of our autoscaling groups (ASGs) to configure
instances on boot via a bash script with a base64 encoded tarball of the
playbook directory. While we find this is a very effective way to structure the
configuration, testing new configuration can be time consuming as it requires a
new LC to be created and attached to the ASG followed by cycling an instance in
the ASG by terminating it so that a new one, with the new configuration, takes
its place.

However, another advantage of using Ansible for configuration is that we can
run the configuration locally and target the instances that are part of the ASG
(you'd want to use a non-production ASG, of course) to test new configuration
iteratively.  To do this, we need to create an inventory of these instances so
that Ansible knows which to run the configuration against.  Luckily, Ansible
allows you to write [dynamic
inventories](http://docs.ansible.com/ansible/intro_dynamic_inventory.html) that
are actually executables that will be run by Ansible to generate the inventory.

Here is an adaptation of the inventory script that we use (with Braintree
specific variables and other noise removed):

```python
#! /usr/bin/env python

import argparse
import boto.ec2.autoscale
import json
import os
import sys

def get_tag(tags, key):
    for tag in tags:
        if tag.key == key:
            return tag.value

    return None

region = os.environ.get('AWS_REGION', None)
if region is None:
    print "$AWS_REGION must be set"
    sys.exit(1)

parser = argparse.ArgumentParser(description='Dynamic inventory for autoscaling groups')
parser.add_argument('--list', help="list hosts", action="store_true")
parser.add_argument('--host', help="list host vars")
args = parser.parse_args()

if args.host:
  print "{}"

if not args.list:
  sys.exit(1)

autoscale = boto.ec2.autoscale.connect_to_region(region)
ec2 = boto.ec2.connect_to_region(region)

inventory = {"_meta": {"hostvars": {}}}
for autoscaling_group in autoscale.get_all_groups():
  instance_ids = [i.instance_id for i in autoscaling_group.instances]
  instance_dns_names = [i.public_dns_name for r in ec2.get_all_instances(instance_ids) for i in r.instances]
  name = get_tag(autoscaling_group.tags, 'Name')
  if name not in inventory:
      inventory[name] = { "hosts": [] }
  inventory[name]['hosts'] += instance_dns_names

print json.dumps(inventory)
```

This script will pull all of the ASGs in your region and return their instances
grouped by the `Name` tag on the autoscaling group. There exists a more full
featured [ec2
inventory](https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.py)
script, but we found this didn't work for our use cases because it doesn't
handle grouping instances by autoscaling groups.

The use of `boto` allows us to easily take AWS credentials from all of the
typical places (`~/.aws/credentials`, environment variables, and instance IAM
roles).

Example output of `AWS_REGION=us-east-1 ./asg-inventory --list | jq .`:
```
{
  "haproxy": {
    "hosts": [
      "ec2-55-1-114-78.us-west-1.compute.amazonaws.com",
      "ec2-51-9-184-33.us-west-1.compute.amazonaws.com"
    ]
  },
  "_meta": {
    "hostvars": {}
  },
  "web_servers": {
    "hosts": [
      "ec2-59-68-77-112.us-west-1.compute.amazonaws.com",
      "ec2-52-151-19-125.us-west-1.compute.amazonaws.com"
    ]
  }
}
```

In your playbook, it is a good idea to expose the target hosts as a variable,
e.g.:

```yml
---
- hosts: "{{ target }}"
  vars:
    target: 127.0.0.1
  tasks:
```

So that you now can then do something like: `AWS_REGION=us-east-1
ansible-playbook -i ./asg-inventory /path/to/playbook.yml -e target=haproxy` to
run your local playbook against the remote instances in the `haproxy`
autoscaling group as you make changes locally.

Once you are satisfied with your configuration, I still recommend going through
the process of updating the LC with your gzipped Ansible configuration (again
see my [previous post]({{<ref
"/post/ansible-aws-launch-configuration.md">}}) for a way to do this) and
spinning a fresh instance up in your ASG to ensure that your configuration
works when ran from scratch. I've often been bit by configuration that worked
while I was iteratively adding to it and mutating state where it did not work
on a fresh instance.

Happy Ansibling!
