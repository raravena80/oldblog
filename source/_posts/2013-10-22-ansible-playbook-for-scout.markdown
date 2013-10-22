---
layout: post
title: "Ansible Playbook for Scout"
date: 2013-10-22
comments: true
# categories: scout ansible

---

This is a sample Ansible task (http://www.ansibleworks.com) 
on how to setup Scout (https://www.scoutapp.com)
It needs to be included in an ansible playbook.

It's a follow up to a previous
 [blog]({% post_url 2013-10-21-how-to-create-an-ansible-playbook-to-configure-haproxy %})
 describing an Ansible Playbook to setup an HAProxy system. This Ansible task can
 be included in the HAProxy playbook as well as any other playbooks with something
 like this:

``` yaml Any playbook
---

- include tasks/scout.yml
```

``` yaml scout.yml
---
# TASK: ScoutApp Monitoring (https://scoutapp.com) 

- include: ../tasks/ruby-apt.yml

- name: Install scout + dependencies
  shell: >
    executable=/bin/bash source /etc/profile.d/rvm.sh;
    gem install scout scout_api --no-rdoc --no-ri

- name: Create scout home directory
  file: >
    dest=/root/.scout state=directory
    owner=root group=root mode=0700

- name: Copy over scout public key
  copy: >
    src=../packages/files/scout/scout_rsa.pub
    dest=/root/.scout/scout_rsa.pub
    owner=root group=root mode=0400

- name: Creating admobius directories
  file: >
    dest=/usr/share/admobius/bin
    state=directory owner=root group=ambot mode=0755

- name: Scout cron script crontab
  template: >
    dest=/etc/cron.d/scout
    src=../packages/templates/scout/scout-crontab.r2
    owner=root group=root mode=0444
  when_string: $env == 'prod'

- name: Scout cron script logrotate
  copy: >
    dest=/etc/logrotate.d/scout
    src=../packages/files/scout/scout-logrotate
    owner=root group=root mode=0444
```
