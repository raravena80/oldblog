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

``` yaml scout.yml
---
PLAYBOOK: Install scout in Ubuntu
---
# Create a subset of users
- name: scout
  hosts: all
  user: user-with-sudo
  sudo: True

  vars:
    scout_key: YourScoutAPIKeyFromTheirWebsite

  tasks:
    - include: tasks/scout.yml
```

We start by defining a "task" file:

``` yaml tasks/scout.yml start:1
---
# TASK: ScoutApp Monitoring (https://scoutapp.com) 

# Separate task to install Ruby
- include: ruby.yml

- name: Install scout + dependencies
  shell: >
    executable=/bin/bash source /etc/profile.d/rvm.sh;
    gem install scout scout_api --no-rdoc --no-ri

- name: Create scout home directory
  file: >
    dest=/root/.scout state=directory
    owner=root group=root mode=0700

```

In the same file add the crontab entry
and logrotate entry for Scout.

``` yaml tasks/scout.yml start:20

- name: Scout cron script crontab
  template: >
    dest=/etc/cron.d/scout
    src=../packages/templates/scout/scout-crontab.j2
    owner=root group=root mode=0444

- name: Scout cron script logrotate
  copy: >
    dest=/etc/logrotate.d/scout
    src=../packages/files/scout/scout-logrotate
    owner=root group=root mode=0444
```

This is what scout-crontab.j2 looks like:

``` jinja templates/scout-crontab.j2
# crontab for Scout monitoring run by root
* * * * * root /bin/bash -l -c 'scout -n "{{ ansible_fqdn }}" {{ scout_key }}' >> /var/log/scout.log 2>&1
```

And this is what scout-logrotate looks like:

``` plain files/scout-logrotate
/var/log/admobius/scout.log
{
      rotate 7
      daily
      compress
      delaycompress
      missingok
      notifempty
}
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

and now run it.

``` bash run.sh
ansible-playbook -T 120 scout.yml
```
