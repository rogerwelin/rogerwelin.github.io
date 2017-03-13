---
layout: post
comments: true
title:  "Testing Ansible Playbooks with Docker"
toc: true
date:   2016-07-04 18:43:43
categories: ansible docker
---

## Introduction

As long as I have been working with configuration management tools (puppet & ansible) there hasn't really been a good way to test the units you've been written. Up until recently my experience has been something like this: working on a feature on the master branch, in best case someone will lend their eyes to look at the changes, run changes directly on the target environment and hope everything works without spewing errors. Except being an embarrasing workflow this imposes a risk; there's no way of knowing that your changes won't set the target environment on fire and either way testing in production (or any other target environment for that matter) is unacceptable!

Operations have come a long way to embrace good software developer practices, so there should be a way for us to embrace this regarding writing and testing playbooks/modules/cookbooks as well. What I have in mind is hardly revolutionary (basically just a feedback-loop):

Working on a feature on a branch **->** Commit and push changes **->** Automatically trigger a run on the CI server **->** CI server checks syntax, lint and runs the actual playbook **->** If successful; branch can be merged to master, If failed; redo process until success

Let's look at a couple of alternatives how to set this up. As config management tools change state I ideally want something that I can spin up, test on and later tear down without changing state on something that is persistent.

## Alternatives

**Locally.** Running playbooks locally is indeed a possibility. However from my point of view it's not really a viable alternative as my host operation system need to be the same as the target environment OS, and it needs the same configuration and repos. And I don't want to mess around installing and configuring on my workstation.

**Vagrant.** The old workhorse. Vagrant is definitely a viable solution. The major selling point, as it's virtualization, is that you can mirror your target environment with Vagrant. The drawbacks are that it's pretty slow to spin up and I haven't found a good way to integrate Vagrant with CI servers (Jenkins).

**Docker.** Docker is an interesting alternative, containers are blazingly fast to spin up and tear down and integration to a CI server is really easy. Docker best practices is to run one process per container and if we're going to use Docker containers to test ansible playbooks we're going to violate that (since we need systemd as a init system and openssh server). With that in mind let's try it anyway!

## Dockerfile
I'll be using centos 7 here. The first problem is to find an image that has both systemd and sshd installed. I couldn't find a good image so we're going to build it ourselves. Take a look at the Dockerfile below:

{% highlight bash %}
FROM centos:7
MAINTAINER "Roger Welin" <roger.welin@icloud.com>
ENV container docker
RUN (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == systemd-tmpfiles-setup.service ] || rm -f $i; done); \
rm -f /lib/systemd/system/multi-user.target.wants/*;\
rm -f /etc/systemd/system/*.wants/*;\
rm -f /lib/systemd/system/local-fs.target.wants/*; \
rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
rm -f /lib/systemd/system/basic.target.wants/*;\
rm -f /lib/systemd/system/anaconda.target.wants/*;
RUN yum -y update; yum clean all
RUN yum -y install openssh-server openssh-clients passwd sudo; yum clean all
RUN systemctl enable sshd.service
ADD ./start.sh /start.sh
ADD ./run.sh /run.sh
RUN mkdir /var/run/sshd
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key -N ''
RUN sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config && sed -ri 's/#UsePAM no/UsePAM no/g' /etc/ssh/sshd_config
RUN chmod 755 /start.sh && chmod 755 /run.sh
RUN ./start.sh
VOLUME [ "/sys/fs/cgroup" ]
ENV AUTHORIZED_KEYS nil
EXPOSE 22
CMD ["/run.sh"]
{% endhighlight %}

**Explanation**

* We base this on the [official centos7 image](https://github.com/docker-library/docs/tree/master/centos)
* The ugly hack starting at the RUN command and below is needed to get systemd working, [docs here](https://github.com/docker-library/docs/tree/master/centos#systemd-integration)
* We install openssh-server and passwd and sudo (this enables us to create a ssh user)
* We enable sshd service with systemctl so it starts at boot
* We run a script called start.sh (see below) that will create a user, called user, and add it to sudoers
* We create an environment variable, `AUTHORIZED_KEYS`, which we will inject our public ssh key when the containers starts that ansible will use to authenticate
* run.sh (see below) checks if `AUTHORIZED_KEYS` environment variable is set, if true it takes the value and popluate authorized_keys file. Then it runs `exec /usr/sbin/init`

**start.sh**
 
{% highlight bash %}
#!/bin/bash
__create_user() {
  # Create a user to SSH into as.
  useradd user
  SSH_USERPASS=newpass
  echo -e "$SSH_USERPASS\n$SSH_USERPASS" | (passwd --stdin user)
  echo ssh user password: $SSH_USERPASS
}

__add_user_to_sudoers() {
  echo "user ALL=(ALL:ALL) NOPASSWD: ALL" | (EDITOR="tee -a" visudo)
}

# Call all functions
__create_user
__add_user_to_sudoers
{% endhighlight %}

**run.sh**

{% highlight bash %}
#!/bin/bash

if [ "${AUTHORIZED_KEYS}" != "nil" ]; then
  mkdir -p /home/user/.ssh
  chmod 700 /home/user/.ssh
  touch /home/user/.ssh/authorized_keys
  chmod 600 /home/user/.ssh/authorized_keys
  echo ${AUTHORIZED_KEYS} > /home/user/.ssh/authorized_keys
  chown -R user /home/user/.ssh
fi

exec /usr/sbin/init
{% endhighlight %}


## Putting it all together
Our project should look something like this now:

{% highlight bash %}
├── ansible
│   ├── env
│   │   └── local_docker
│   ├── httpd.yml
│   ├── roles
│   │   └── httpd
│   │       └── tasks
│   │           └── main.yml
│   ├── site.yml
│   └── ssh
│       ├── id_rsa
│       └── id_rsa.pub
├── container-start-and-playbook-run.sh
└── docker
    ├── Dockerfile
    ├── run.sh
    └── start.sh
{% endhighlight %}

First things first, lets build our docker image (I won't bore you with the output): `docker build -f docker/Dockerfile -t local/centos7-systemd .`

Then let's create a ssh key-pair to use for ansible to ssh into the container. `ssh-keygen -t rsa`, then put them in the ssh dir (I chose to put the ssh dir in the ansible dir, but you could place them somewhere else)

Let's fix the ansible part of it. I'm going to create a very simple httpd role that installs httpd and check that it's running. But first we need to add details to the inventory file (`env/local-test`) to let ansible know how it can access the container:

{% highlight bash %}
[httpd]
localhost ansible_ssh_port=5000 ansible_ssh_user=user
{% endhighlight %}

Now lets take a look at our httpd role at `roles/httpd/tasks/main.yml`:


{% highlight yml %}
---
- name: install httpd
  yum: name=httpd state=installed

- name: check httpd running and enabled
  service: name=httpd state=running enabled=yes

- name: check port 80 answering
  wait_for: port=80
{% endhighlight %}


As you can see, nothing revolutionary is going on here, it's just instructions to install httpd and enable it and see that port 80 is open. Now let's take a look at `httpd.yml`:


{% highlight yml %}
---
- hosts: httpd
  become: yes
  become_user: root
  roles:
    - httpd
{% endhighlight %}


Again, this is nothing out of the ordinary. An important thing to note is that we're setting up privilegie escalation so we can execute command as root.

Cool, now we're ready to start a container from the image we created and run `ansible-playbook` against it! For convenience I'll put all the commands in a shell script, `container-start-and-playbook-run.sh`, that way it's easy to chain everything together. Here's the content of the shell script:

{% highlight bash %}
#!/bin/bash

DCKER_CONTAINER_NAME="ansible-role-test"
SSH_PUBLIC_KEY_FILE=ansible/ssh/id_rsa.pub
SSH_PUBLIC_KEY=$(cat "$SSH_PUBLIC_KEY_FILE")

docker run -ti --privileged --name $DOCKER_CONTAINER_NAME -d -p 5000:22 -e AUTHORIZED_KEYS="$SSH_PUBLIC_KEY" local/centos7-systemd

cd ansible && ansible-playbook -i env/local_docker site.yml --private-key ssh/id_rsa

{% endhighlight %}

This should be pretty self-explanatory by now. We map port 22 inside the container to port 5000 on the host which we also defined in ansibles inventory file. Lets go ahead and run the script:

{% highlight bash %}
[root@localhost docker-ansible]# bash container-start-and-playbook-run.sh
589656a2eb6c754f11869bd20fab117fb4a2ee28ff1b7572eecf82e88c64dc70

PLAY [httpd] *******************************************************************

TASK [setup] *******************************************************************
ok: [localhost]

TASK [httpd : install httpd] ***************************************************
changed: [localhost]

TASK [httpd : check httpd running and enabled] *********************************
changed: [localhost]

TASK [httpd : check port 80 answering] *****************************************
ok: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=4    changed=2    unreachable=0    failed=0
{% endhighlight %}

It worked! **Great success! :)**

## Pro's and Con's

**Pro's**

* Fast and flexible
* Easy integration with CI servers so setup can establish a nice workflow
* Very little extra configuration needed; just ssh-keys and another environment file, otherwise playbook can be run as is

**Con's**

* The container needs to mount systemd cgroups on the host as a volume so it cannot run on distos using another init system or OSX
* Setup feels a bit hackish
