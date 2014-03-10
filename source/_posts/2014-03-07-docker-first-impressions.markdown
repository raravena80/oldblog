---
layout: post
title: "Docker First Impressions"
date: 2014-03-07 16:28
comments: true
categories: docker
keywords: docker Ricardo Aravena
description: A first impressions take on Docker.
---

For the last few days I've been taking at crack at using the recent [Docker](https://www.docker.io/) container deployment tool that I've been
hearing a lot buzz about.
In essence, it's a wrapper on top of Linux [LXC containers](https://linuxcontainers.org/),  writen in the new friendly and not so popular yet [Go](http://golang.org/) language developed at Google.

Just a little bit of background, for those of you not familiar with LXC containers,
they are pretty much defined as `chroot` on steroids. Basically, you can run isolated virtual
environments in a single Linux machine and make it look like that they are different machines.
These environments give you the advantage of being isalated and at the same they are able to use
the same Linux exectutables and memory space to improve speed and footprint size.

Docker is pretty easy to try from its [website](https://www.docker.io/), you can just click on the [Get Started](https://www.docker.io/gettingstarted/) 
link and a UI terminal shows up where you can type Docker commands. It's CLI feels a lot like a CLI writen in Python or Ruby, yet it still doesn't
have the nice subtleties such as using a `?` to get help. To get help you simply type `docker help` or `docker command help` 

Setup varies depending on your platform. It can be a bit confusing if you work with multiple say, Linux platforms but from personal experience
most people stick to a more `favorite` Linux platform. Docker is also available for MacOS and Windows, but in reality 
it doesn't run over either one since `LXC` is only available for Linux. They way they mention that it can be run on MacOS or Windows is by running 
it on a lightweght VM. 

As of this writing there's a message on each one of the platform installation pages that says: `Docker is still under heavy development! We don’t recommend using it in production yet, but we’re getting closer with each release. Please see our blog post, “Getting to Docker 1.0”`

For me, it wasn't that difficult to setup, I just followed the steps on the Wiki. I don't think the steps
are that difficult but it just requires basic knowledge of the Linux command line.
Docker requires the Linux 3.8 kernel and in my case, I tried Docker on Ubuntu 13.10 so I didn't have to install an extra kernel package.

However, if you are running say Ubuntu 12.04 LTS (Precise) you are going to have to install the update kernel packages and reboot the machine 
before you can use Docker:


``` bash linenos:false
$ # install the backported kernel from raring
$ sudo apt-get update
$ sudo apt-get install linux-image-generic-lts-raring linux-headers-generic-lts-raring
$ sudo reboot
```

In my case, for Ubuntu 13.10 it says that if you want AUFS (AnotherUnionFS) support you need to install the `linux-image-extra` package:

``` bash linenos:false
$ sudo apt-get update
$ sudo apt-get install linux-image-extra-\`uname -r\`
```

It turns out that my Ubuntu 13.10 already had the `linux-image-extra` package, so I didn't have to make any changes. 


Next I had to run:

``` bash linenos:false
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
$ sudo sh -c "echo deb http://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"
$ sudo apt-get update
$ sudo apt-get install lxc-docker
```

But if you prefer,  there's also a curl script that simplifies the previous steps.

``` bash linenos:false
$ curl -s https://get.docker.io/ubuntu/ | sudo sh
```

All in all pretty easy.  One last thing is that if you have the Ubuntu Firewall enabled (ufw) you need
to allow it to forward traffic in the `/etc/default/ufw` file:

```
# Change:
# DEFAULT_FORWARD_POLICY="DROP"
# to
DEFAULT_FORWARD_POLICY="ACCEPT"
```
and then run:

``` bash linenos:false
$ sudo ufw allow 4243/tcp
```

Now we are all dandy and ready to try Docker commands. The first one that you want to run is
the one to create a container:

``` bash linenos:false
$ sudo docker run -i -t ubuntu /bin/bash
```

This command will automatically download all the default ubuntu container images that you can use to run your Ubuntu container. It does take a while but then again it's downloading full container images each about 60Mb compressed. Keep in mind that the `-i` option means "interactive" and the `-t` means allocate a pseudo tty.

This the output of the command:

``` text linenos:false
Unable to find image 'ubuntu' locally
Pulling repository ubuntu
5ac751e8d623: Download complete
9cd978db300e: Download complete
9cc9ea5ea540: Download complete
9f676bd305a4: Download complete
eb601b8965b8: Download complete
511136ea3c5a: Download complete
f323cf34fd77: Download complete
7a4f87241845: Download complete
1c7f181e78b9: Download complete
6170bb7b0ad1: Download complete
321f7f4200f4: Download complete
```

After it's complete, it will display a `bash` shell in the container with the prompt displaying the container hash id. For example, `root@3b667578ce4f:/#`. You can run almost any linux command that your Ubuntu distro supports, including something like: `apt-get update; apt-get install apache2`. 

I ran `apt-get upgrade` and somehow it couldn't finish the installation saying some of the packages were missing dependencies, so in essence I fried the container.

I said no problem, I'll just get rid of it. First hit the `ctrl-p ctrl-q` keys to detach from the container. Then, run these Docker commands to find out the container id (if you somehow don't remember it), stop the container and then delete the container.

``` bash list-containers linenos:false
$ docker ps
```

then:

``` bash linenos:false
$ docker stop 3b667578ce4f
$ docker rm 3b667578ce4f
```
and we are back to square one so we can start with a clean sheet. You can notice the all the containers will be stored here: `/var/lib/docker/containers` in a directory that matches the container's specific hash id. You'll also notice that after you run the `docker rm` command the directory for the specific container is deleted.

There are other command that are useful. For example:

``` bash linenos:false
$ docker images
```
will display all the downloaded images that can be used as a container. In my case, it's a list of Ubuntu distros:

``` bash linenos:false
root@host:~# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu              saucy               9f676bd305a4        4 weeks ago         182.1 MB
ubuntu              13.10               9f676bd305a4        4 weeks ago         182.1 MB
ubuntu              13.04               eb601b8965b8        4 weeks ago         170.2 MB
ubuntu              raring              eb601b8965b8        4 weeks ago         170.2 MB
ubuntu              12.10               5ac751e8d623        4 weeks ago         161.4 MB
ubuntu              quantal             5ac751e8d623        4 weeks ago         161.4 MB
ubuntu              10.04               9cc9ea5ea540        4 weeks ago         183 MB
ubuntu              lucid               9cc9ea5ea540        4 weeks ago         183 MB
ubuntu              12.04               9cd978db300e        4 weeks ago         204.7 MB
ubuntu              latest              9cd978db300e        4 weeks ago         204.7 MB
ubuntu              precise             9cd978db300e        4 weeks ago         204.7 MB
```

So now let's say you want to use the `quantal` image you can run:

``` bash linenos:false
$ docker run -t -i ubuntu:5ac751e8d623 /bin/bash
```

Although, out of the scope of this specific post, there are many other features that you can use including creating images and publishing them to a public repository. 

### Conclusion ###

Docker is a very useful tool for people wanting to deploy application in an isolated way so that it doesn't interfere with the major functions of a particular server. It allows for easy creation an deletion of containers in case something goes wrong or simply when ops guys are trying to deploy something quickly already prebaked into a Docker image. You can even deploy Docker containers in AWS EC2 instances to even further compartamentalize your application. 

However, if you are concerned about Docker being in its early development stage (as of this writing) and if you don't care about costs to a certain extent, the Docker/LXC approach is not very different from say using Amazon EC2 prebaked AMIs. LXC containers are pretty light weight but whatever it is that you are running will still be CPU, Disk I/O and Network contrained on the same physical or virtualized machine.

Also, there are also several tools available to create AMIs including the infamous [Aminator](https://github.com/Netflix/aminator) developed at Neflix. And if you happen to be a fan of [Ansible](http://www.ansible.com/home) like me, you can use the [Ansible Aminator playbook](https://github.com/Answers4AWS/netflixoss-ansible/) from [AnSwers](http://answersforaws.com/).

