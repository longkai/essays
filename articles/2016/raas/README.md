Raspberry Pi as a Service
===
Borrowing from those popular expressions, e.g. 'XaaS', this article gives a brief stories about what I did some of the services right from a tiny Pi.

## Local DNS Mapping
A server usually has a static IP address or more for providing a sustainable and convenient service for the users, which is absolutely fine in the public network. What's more, people don't get well with numbers but text so DNS comes into play. Our Pi lives in the local or private network, a **static numeric** IP would be in a mess. Supposing you set Pi's IP to `192.168.1.233`, all of the network(i.e. TCP/IP) based service needs an IP address, e.g. ssh, http(s), ftp, etc. Plus sometimes you need to specify a port associate with the service. What a mess.

We need to set up a local DNS service. Say, `raspberrypi.local` points to `192.168.1.233`, then everything is fine. Besides, a sub-domain may more helpful to escape from numeric **port**. Say `git.raspberrypi.local` points to the local git service, `www.raspberrypi.local` points to local web, etc.

There're many ways to reach this goal. For example, you can setup your local DNS server, modifying your `/etc/hosts` derectly, or using [Multicast DNS][mDNS]. The first one requires some network experience which may suit for those power users. The middle one is the most effective, and the last one is the most flexible, set once, apply every where(the whole local network).

If your OS is Unix based, you can try dial `raspberrypi.local`, which may have already been set up!

## Local Git Service
How about host your own Github like service locally? Github is great and you can always use it. However, for some reason, maybe sensitive or private, you can't push your work. In this case, a local git service is awesome, you have a full control your work, plus a replica bonus.

There're many open source git service lets you deploy locally, like [gitlab][gitlab], [gogs][gogs] even git itself!

For a tiny Pi, I suppose the lightweight **gogs** it's enough. The good news, its deployment is fair simple, no other dependencies required. By the way, you can use docker for a quick start.

## Local Cloud Storage
Like the previous section, what about making your personal cloud storage service? There're a lot of reasons, security, privacy, terms of service etc. It's always a good idea that only you have the full access control your data.

I choose [ownCloud][ownCloud], an open source providing cross platform client support(i.e. Web, Android, iOS). Plus, it offers some rich features like [WebDAV][CalDAV], [CalDAV][CalDAV], [CardDAV][CardDAV], with which sync and restore your contacts, calendar, etc.

## Full Stack Proxy Server
With Pi's wireless support and built-in Linux OS, there's no reason you can't control your network.

You know it.


Happy hacking.

### EOF
```yaml
date: 2016-11-17T18:29:55+08:00
summary: Host some aswsome services right in your Pi.
weather: fine
license: cc-40-by
location: 22,114
background: docker-raspberry-pi.jpg
tags:
  - Hacking
```
[mDNS]: https://en.wikipedia.org/wiki/Multicast_DNS
[gitlab]: https://about.gitlab.com
[gogs]: https://github.com/gogits/gogs
[ownCloud]: https://owncloud.org/
[WebDAV]: https://en.wikipedia.org/wiki/WebDAV
[CalDAV]: https://en.wikipedia.org/wiki/CalDAV
[CardDAV]: https://en.wikipedia.org/wiki/CardDAV

