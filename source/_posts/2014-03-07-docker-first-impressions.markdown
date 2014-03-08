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
In essence is wrapper on top of Linux [LXC containers](https://linuxcontainers.org/)  writen in the new friendly and not so popular yet [Go](http://golang.org/) language developed at Google.

Just a little bit of background, for those of you not familiar with LXC containers,
they are pretty much defined as `chroot` on steroids. Basically, you can run isolated virtual
environments in a single Linux machines and make it look like that they are different machines.
These environment give you the advantage of being isalated and at the same they are able to use
the same Linux exectutables and memory space to improve speed and footprint size.

Docker is pretty easy to try from its [website](https://www.docker.io/), you can just click on the [Get Started](https://www.docker.io/gettingstarted/) 
link and a UI terminal shows up where you can type Docker commands. It's CLI feels a lot like a CLI writen in Python or Ruby, yet it still don't
have the nice subtleties such as using a `?` to get help. To get gelp you simply type `docker help` or `docker command help` 


Setup varies depending on your platform. It can be a bit confusing if you work with multiple say, Linux platforms but from personal experience
most people stick to a more `favorite` Linux platform. Docker is also available for to be run MacOS and Windows, but in reality 
it doesn't run over either one since `LXC` is only available for Linux. They way they mention that it can be run on MacOS or Windows is by running 
it on a lightweght VM. 

As of this writing there's a message on each one of the platform installation pages that says: `Docker is still under heavy development! We don’t recommend using it in production yet, but we’re getting closer with each release. Please see our blog post, “Getting to Docker 1.0”`

For me, it wasn't that difficult to setup, I just followed the steps on the Wiki. I don't think the steps
are that difficult but it would be nice to have a single command to setup up, a la Ruby rvm. 

Docker requires the Linux 3.8 kernel and in my case, I tried Docker on Ubuntu 13.10 so I didn't have to install an extra kernel packages.
However, if you are running say Ubuntu 12.04 LTS (Precise) you are going to have to install the update kernel packages and reboot the machine 
before you can use Docker:


``` bash
# install the backported kernel from raring
sudo apt-get update
sudo apt-get install linux-image-generic-lts-raring linux-headers-generic-lts-raring
sudo reboot
```

In my case for Ubuntu 13.10 it says that if you want AUFS (AnotherUnionFS) support you need to install the kernel extra packages:

``` bash
sudo apt-get update
sudo apt-get install linux-image-extra-`uname -r`
```

However, it turns out that my Ubuntu 13.10 already had the linux image extra package.





