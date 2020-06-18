Setup Your ownCloud Service
===
> Note: This is just a breif intro, which needs other tech skills to get things done.

There're a lot of cloud based services you can choose from like Dropbox, Google Driver, iCould, BaiduPan, many more. I would like suggest you try to use this service because they're really good. However, there're always some reasons you won't use these provider(I suppose is rare), like privacy, unwanted terms of services, leak data, regulation, security, lack of functionality, etc.

Last month, 360 YunPan, one of the largest cloud stroage provider announced their service will be shutdown in mouths. So it's not always safe to put your important data to cloud. I usually like to backup my very important files to my local storage device.

So, why I'm setup my own cloud service? Because I want to sync Omnifocus with WebDAV. For many reasons, Omni sync server's performance is laggy. Sometimes you make changes in your Mac, the iPhone won't get synced. So I [mentioned their Twitter][twitter-status] account and got reply that I can use WebDAV server for syncing.

So what's WebDAV? You can find it in [Wikipedia][wiki]. In short, you can use it to sync contents for different clients.

At first I search that in China only JianGuoYun provides WebDAV service but it's has some quota restrictions. Sicne I have my own server, why no setup my own?

## Fail on Nginx
Since my server uses Nginx for a lot of web based services, I chose it for the WebDAV server. However, it's doesn't support WebDAV well and lacks functionality for Omni, after hours works I gave it up.

Then I find [ownCloud][ownCloud] in Github issues indicates supporting Omni. Like its name, it's an open source cross platfrom cloud storage application runs right on your server. It seems pretty good so I decide try it.

## Setup ownCloud
It requires a bunch of dependencies to run it since it's [LAMP][LAMP] based. Mysql, php, apache, it's a not easy and time comsuming task. What's worse it may conflict your server. Docker it's an killer for this situation. If you know the basic usage, life is easy!

Assuming you know the basis and has Docker installed, go to the [documentation][ownCloud-docker], it contains all the instruction for getting started. Here it's a brief steps.

1. pull the image with `docker pull owncloud`
2. create `docker-compose.yml` file specify your port and volumes(which contains your saved data!)
3. run ownCloud with `docker-compose up -d`
4. nginx ssl and forwarding support

below is my `docker-compose.yml`

```yaml
owncloud:
  image: owncloud
  ports:
    - 8888:80
  volumes:
    - ~/owncloud/apps:/var/www/html/apps
    - ~/owncloud/config:/var/www/html/config
    - ~/owncloud/data:/var/www/html/data
```

Then reload or restart your nginx server, go to the brower with your url and setup admin account. You will find the web UI its pretty stright forward. On the left bottom is the WebDAV address, copy and paste to your Omni prefences, the sync speed would much better and your data will more secure.

Happy Hacking.

### EOF
```yaml
date: 2016-11-08T21:44:11+08:00
summary: setup your own cloud service for sync OmniFocus WebDAV, and many more.
weather: slightly cold
license: cc-40-by
location: 22,114
background: # banner image for this article, or RGBA hex color value(i.e. starting with '#')
tags:
  - Hacking
```

[twitter-status]: https://twitter.com/xiaolongtongxue/status/795789662065262592
[wiki]: https://en.wikipedia.org/wiki/WebDAV
[ownCloud]: https://owncloud.org
[LAMP]: https://en.wikipedia.org/wiki/LAMP_(software_bundle)
[ownCloud-docker]: https://hub.docker.com/_/owncloud/
