---
layout: post
title: "How To Create an Ansible Playbook to Configure HAProxy"
date: 2013-10-21
# categories: [ haproxy ]
comments: true

---

This is the continuation for
[Setup a Simple HAProxy Config]({% post_url 2013-10-21-setup-a-simple-haproxy-config %})

It explains how to create an Ansible playbook to automate the haproxy
configuration.


If you'd like to find out more about Ansible you can read up on it on
their website: http://www.ansibleworks.com

``` yaml haproxy.yml
---
# Set up and configure an HaProxy server (Ubuntu flavor)
- name: haproxy
  hosts: all
  user: userwithsudoaccess
  sudo: True
  tags: haproxy

  vars_files:
    - "vars/main.yml"

  tasks:

    # haproxy package for Ubuntu
    - include: tasks/haproxy-apt.yml

    # Specific haproxy tasks follow here
    - name: Copy haproxy logrotate file
      action: >
        copy src=files/haproxy.logrotate dest=/etc/logrotate.d/haproxy
        mode=0644 owner=root group=root

    - name: Create haproxy rsyslog configuration
      action: >
        copy src=files/haproxy-rsyslog.conf
        dest=/etc/rsyslog.d/49-haproxy.conf
        mode=0644 owner=root group=root
      notify: restart rsyslog

    - name: Configure system rsyslog
      action: >
        copy src=files/rsyslog.conf
        dest=/etc/rsyslog.conf
        mode=0644 owner=root group=root
      notify: restart rsyslog

    - name: Create haproxy configuration file
      action: >
        template src=templates/haproxy.cfg.j2 dest=/etc/haproxy/haproxy.cfg
        mode=0644 owner=root group=root
      notify: restart haproxy

```


The following file that contains the variables needed for the haproxy playbook
it should located under vars (vars/main.yml)

``` yaml vars/main.yml
---
haproxy_port: 8080
haproxy_servers:
  - server1.mydomain.com
  - server2.mydomain.com
  - server3.mydomain.com

```

The following is the task/haproxy-apt.yml file that is used to
install haproxy on Ubuntu. If you are using CentOS or RedHat
you can use 'yum' instead of 'apt'

``` yaml tasks/haproxy-apt.yml
---
# TASK: Install and configure HAProxy - Ubuntu style
#

- name: Install HAProxy
  action: apt pkg=$item state=latest
  with_items:
    - haproxy
  
- name: Enable HAProxy service
  action: service name=haproxy enabled=yes
  
- name: Copy Ubuntu default file
  action: >
    copy dest=/etc/default/haproxy
    src=../packages/files/haproxy/default
    owner=root group=root mode=0444
  notify: restart haproxy
  # Note the notify clause is handled by a
  # Ansible handler (explained below)
```

The content for rsyslog.conf, haproxy.logrotate and 49-haproxy.conf can
be found in the previous
[blog]({% post_url 2013-10-21-setup-a-simple-haproxy-config %})

However, this time we are templating haproxy.cfg with jinja2 and the content is:
{% raw %}
``` jinja haproxy.cfg.j2
global
	log 127.0.0.1	local0
	log 127.0.0.1	local1 notice
	maxconn 4096
	user haproxy
	group haproxy
	daemon

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
	retries	3
	option redispatch
	maxconn	2000
	contimeout	5000
	clitimeout	50000
	srvtimeout	50000

	stats enable
	stats auth		admin:password
	stats uri		/monitor
	stats refresh	5s
	option httpchk	GET /status
	retries		5
	option redispatch
	errorfile	503	/etc/haproxy/errors/503.http
	errorfile	400	/etc/haproxy/errors/400.http
	errorfile	403	/etc/haproxy/errors/403.http
	errorfile	408	/etc/haproxy/errors/408.http
	errorfile	500	/etc/haproxy/errors/500.http
	errorfile	502	/etc/haproxy/errors/502.http
	errorfile	503	/etc/haproxy/errors/503.http
	errorfile	504	/etc/haproxy/errors/504.http
	balance roundrobin	# each server is used in turns, according to assigned weight

listen http-in
	bind :80
	monitor-uri   /haproxy  # end point to monitor HAProxy status (returns 200)
	# option httpclose
	{% for dmp_server in dmp_servers %}
	server {{ dmp_server }} {{ dmp_server }}:{{ dmp_port }} weight 1 maxconn 1000 check inter 4000
	{% endfor %}
	rspidel ^Set-cookie:\ IP=	# do not let this cookie tell our internal IP address
```
{% endraw %}

Include handlers at the end of the file:

{% codeblock lang:yaml haproxy.yml %}

  handlers:
    - include: handlers/main.yml
{% endcodeblock %}

The content of handlers/main.yml looks like this:

``` yaml handlers/main.yml
---
# Ansible Handlers

- name: restart haproxy
  action: service name=haproxy state=restarted
  
- name: restart rsyslog
  action: service name=rsyslog state=restarted
```

<h3>Optional</h3>
Include Scout (https://scoutapp.com) and Papertrail (https://papertrailapp.com)
More on this later...

``` yaml haproxy.yml
    # Scout
    - include: tasks/scout.yml
      when: env == 'prod'

    # Papertrail for logging
    - include: tasks/papertrail.yml
      when: env == 'prod'

```

Now run it:

``` bash
ansible-playbook -T 120 -i <inventory-file> haproxy.yml
```
