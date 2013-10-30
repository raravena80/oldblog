---
layout: post
title: "Ansible Playbook for PaperTrail on Ubuntu"
date: 2013-10-29 11:54
comments: true
categories:
---

This posts describes how to create a simple
[Ansible](http://www.ansibleworks.com) task
on how to setup [PaperTrail](https://www.papertrailapp.com) on Ubuntu.
Because it's a `task` it needs to be included in an ansible playbook.

It's a follow up to a previous
 [blog]({% post_url 2013-10-21-how-to-create-an-ansible-playbook-to-configure-haproxy %})
 describing an Ansible Playbook to setup an HAProxy system. This Ansible task can
 be included in the HAProxy playbook as well as any other playbooks with something
 like this:

``` yaml papertrail.yml
---
PLAYBOOK: Install papertrail on Ubuntu
---
- name: scout
  hosts: all
  user: <user-with-sudo>
  sudo: True

  tasks:
    - include: tasks/papertrail.yml
```

Next, we define the `task` that includes installing the dependencies
rsyslog and libssl-dev. Also we copy a specific rsyslog configuration
for papertrail.

``` yaml papertrail.yml
---
# TASK: Papertrail log aggregation

- name: Install dependencies for Papertrail
  apt: pkg=$item state=latest
  with_items:
    - libssl-dev
    - rsyslog-gnutls

- name: Copy rsyslog.conf
  copy: > 
    src=files/rsyslog.conf
    dest=/etc/rsyslog.conf
    owner=root group=root mode=0444
  notify: restart rsyslog

```

And here's the content of rsyslog.conf:

<script
  src="https://gist.github.com/raravena80/7221713.js?file=rsyslog.conf">
</script>

Next you need to include the papertrail cerfiticate file if you want
to encrypt your connection from rsyslog to PaperTrail. 
The link to the certificate file is 
[here](https://papertrailapp.com/tools/syslog.papertrail.crt).
You also need to tell Ansible to restart rsyslog when it installs
this file using the `notify` keyword.

``` yaml papertrail.yml
- name: Papertrail certificate
  copy: >
    src=files/syslog.papertrail.crt
    dest=/etc/syslog.papertrail.crt
    owner=root group=root mode=0444
  notify: restart rsyslog

```
Here you include the specific papertrail configuration
for rsyslog.
``` yaml papertrail.yml
- name: Papertrail rsyslog config file
  copy: >
    src=files/papertrail.conf
    dest=/etc/rsyslog.d/70-papertrail.conf
    owner=root group=root mode=0444
  notify: restart rsyslog

```

The papertrail.conf file can be seen here:

<script
  src="https://gist.github.com/raravena80/7221713.js?file=papertrail.conf">
</script>

Optionally you can install the Ruby papertrail remote
syslog in case you'd like to send random logs from the machine
to PaperTrail.

``` yaml papertrail.yml
- name: Install Papertrail remote file logger
  shell: >
    executable=/bin/bash source /etc/profile.d/rvm.sh;
    gem install remote_syslog --no-ri --no-rdoc
```

Finally just run it: `ansible-playbook -T 120 -i inventory-file papertrail.yml`
