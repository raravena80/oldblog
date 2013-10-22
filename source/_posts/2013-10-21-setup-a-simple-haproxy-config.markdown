---
layout: post
title: "Setup a Simple HAProxy Config"
date: 2013-10-21
comments: true
categories: haproxy
---

Here's simple haproxy configuration to get you started,
you probably want to stick this under /etc/haproxy/haproxy.cfg

``` plain Simple HAProxy Config
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
	maxconn	4096
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
    server server1 server1.mydomain.com:8080 weight 1 maxconn 2000 check inter 4000
    server server2 server2.mydomain.com:8080 weight 1 maxconn 2000 check inter 4000
    server server3 server3.mydomain.com:8080 weight 1 maxconn 2000 check inter 4000
    rspidel ^Set-cookie:\ IP=	# do not let this cookie tell our internal IP address

```

You also want to setup logging using rsyslog,
you can syslog-ng or other loggers too as well,
but the configuration is different.
``` plain Rsyslog HAproxy config
# put this in /etc/rsyslog.d/49-haproxy.conf:
local0.* -/var/log/haproxy/haproxy_0.log
local1.* -/var/log/haproxy/haproxy_1.log
& ~
```

Now, setup logrotate (usually under /etc/logrotate.d/haproxy:
``` plain HAProxy logrotate config
/var/log/haproxy/haproxy*.log
{
    rotate 7
    weekly
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        reload rsyslog >/dev/null 2>&1 || true
    endscript
}
```

You can also use your favorite configuration management tool,
such as Puppet, Chef or Ansible, to parameterize and template
the configurations. I use Ansible (I'll explain in a different
blog)
