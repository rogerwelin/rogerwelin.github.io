---
layout: post
comments: true
title:  "Develop custom Ansible modules"
date:   2016-04-25 18:43:43
categories: ansible
---

## Introduction

One thing that I really think is neat with Ansible is that when you need functionality that are not part of the Ansible core (let's say you need to integrate Ansible with bigip F5 load balancer) you can just turn to Python (or any other language for that matter) to write your own modules that you later can use in the regular playbooks. This is a really neat functionality, you are not tied down to the dsl (such as in the case of Puppet). DSL:s are in my experience very limited and stupid; you would rather want turn to a real general purpose programming language.

Here are some basic considerations of writing Ansible modules:

* Ansible will look in the library directory relative to the playbook, for example: `playbooks/library/your-module`
* You can also specify the path to your custom modules in ansible.cfg, for example: `library = /usr/share/ansible`
* Ansible expects modules to return JSON, for example: `{'changed': false, 'failed': true, 'msg': 'could not reach the host'}`
* If you write your module in Python (it can be written in any language as long as it returns json) you can import Ansible helper libraries
* If are using Python try to limit the libraries you are using to the standard library. If that's not possible you need a strategy to install those libraries everywhere you will run the playbooks (preferably with your Linux distributions package manager and not pip ;) )
* Good practice is to write a multi-line at the start of your module describing parameters and how to run it. This is especially important if you want to open source it.

# A trivial example

Let's say we would want to write a custom module that will check if a program is running on the target host. From our playbook we will call it like this: `is_running: name=consul`

While this is a very trivial example (you would surely not need to write a custom module for this since it's supported in ansible core) this will give you the needed boilerplate code needed to write a custom module, just re-implement that and focus on the real business logic you want to solve.

{% highlight python %}
#!/usr/bin/python
import subprocess

DOCUMENTATION = '''
module: is_running
short_description: "Checks if a process is running on a target machine"
author:
  - your name
requirements:
  - only standard library needed
options:
  name:
    description:
      - the name of the process you want to check
      required: true
      default: null
example:  is_running: name=etcd
'''

def is_running(name):
  process = subprocess.Popen("ps -ef | grep -i " + name + " | grep -v grep", shell=True, stdout=subprocess.PIPE)
  exists = process.communicate()[0]

  if exists:
    return True
  else:
    return False

def main():
  # The AnsibleModule provides lots of common code for handling returns, parses your arguments for you, and allows you to check inputs
  module = AnsibleModule(
    argument_spec=dict(
      name=dict(required=True, type='string')
    ),
    supports_check_mode=True
  )

  # in check mode we take no action
  # since this module actually never changes system state we'll just return False
  if module.check_mode:
    module.exit_json(changed=False)

  name = module.params['name']

  if is_running(name):
    module.exit_json(changed=False)
  else:
    msg = "Program %s is not running on this host" % (name)
    module.fail_json(msg=msg)

from ansible.module_utils.basic import *
if __name__ == '__main__':
  main()
{% endhighlight %}




