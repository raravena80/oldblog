---
layout: post
title: "Ansible Playbook for Scout on Ubuntu"
date: 2013-10-22
comments: true
# categories: scout ansible

---

This is a sample Ansible task (http://www.ansibleworks.com) 
on how to setup Scout (https://www.scoutapp.com) on Ubuntu.
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

``` yaml tasks/scout.yml
---
# TASK: ScoutApp Monitoring (https://scoutapp.com) 

# Separate task to install Ruby
- include: ../tasks/ruby.yml

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
```

In the same file add the crontab entry
and logrotate entry for Scout

``` yaml tasks/scout.yml start:24

- name: Scout cron script crontab
  template: >
    dest=/etc/cron.d/scout
    src=../packages/templates/scout/scout-crontab.j2
    owner=root group=root mode=0444
  when_string: $env == 'prod'

- name: Scout cron script logrotate
  copy: >
    dest=/etc/logrotate.d/scout
    src=../packages/files/scout/scout-logrotate
    owner=root group=root mode=0444
```

Now to install ruby using RVM, if you don't want to use
the system ruby (most of the times you don't).

``` yaml tasks/ruby.yml
---
# TASK: Install Ruby on Ubuntu

- name: Install Ruby dependencies
  apt: pkg=$item state=latest install_recommends=no
  with_items:
    - autoconf
    - automake
    - bison
    - build-essential
    - curl
    - libc6-dev
    - libgdbm-dev
    - libffi-dev
    - libncurses5-dev
    - libreadline6
    - libreadline6-dev
    - libsqlite3-dev
    - libssl-dev
    - libtool
    - libyaml-dev
    - libxml2-dev
    - libxslt1-dev
    - openssl
    - pkg-config
    - sqlite3
    - subversion
    - zlib1g
    - zlib1g-dev

- name: Install RVM
  shell: curl -L get.rvm.io | bash -s stable

- name: Install Ruby 2.0.0
  shell: >
    executable=/bin/bash source /etc/profile.d/rvm.sh;
    rvm install 2.0.0

- name: Set default ruby version
  shell: >
    executable=/bin/bash source /etc/profile.d/rvm.sh;
    rvm --default use 2.0.0
```
