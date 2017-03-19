# How Does QQ Know Who I am

## Abstract
This post gives a brief intro how does QQ identifies who you are in the browser even you have no intention of login, plus some security and privacy considerations.

## Notice
- Only test on macOS
- No Cookie discussed, think of a pure incognito browser environment
- As of this writing, Feb. 27 2017, some information may be changed in the future
- Correct me if I am wrong

## Background
[Tencent QQ][] is the most popular and evergreen instant messaging(IM) software service in China since 1999. QQ provides a huge number of services like gaming, music, video, media, payment, so on and so forth. Most people rely on it and it's everywhere. At the end of June 2016, there are almost 900 million active accounts.

Presume you have a desktop QQ client opened or its running background service.

Have you ever notice that even though you **did not sign in webpages of Tencent**, but when you do, it knows who you are by showing your avatar and nickname, and you don't even need to enter your account information but a single confirm click.

How does it work?

**Note**: in my small test cases, apart from the [Qzone][], a Chinese largest SNS service like Facebook, does actively identifies you, all of their pages don't, until you do by clicking the signin buttons.

## Implementation
Thanks to the Web, an open platform, when in doubt we can dig into the source or dev tool to found what went on.

It's easy and let's start!

Take the Qzone for example. Open Chrome DevTools or other similar tools, then visit QZone, in the *Network* tab, we can find a series of requests whose URL path are `/pt_get_uins`. The request and response messages look like below, respectively. Note some identifications are faked.

```http
GET /pt_get_uins?callback=ptui_getuins_CB&r=0.558885784438953&pt_local_tk=690287468 HTTP/1.1
Host: localhost.ptlogin2.qq.com:4301
Connection: keep-alive
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36
Accept: */*
DNT: 1
Referer: http://xui.ptlogin2.qq.com/cgi-bin/xlogin?proxy_url=http%3A//qzs.qq.com/qzone/v6/portal/proxy.html&daid=5&&hide_title_bar=1&low_login=0&qlogin_auto_login=1&no_verifyimg=1&link_target=blank&appid=549000912&style=22&target=self&s_url=http%3A%2F%2Fqzs.qq.com%2Fqzone%2Fv5%2Floginsucc.html%3Fpara%3Dizone&pt_qr_app=%E6%89%8B%E6%9C%BAQQ%E7%A9%BA%E9%97%B4&pt_qr_link=http%3A//z.qzone.com/download.html&self_regurl=http%3A//qzs.qq.com/qzone/v6/reg/index.html&pt_qr_help_link=http%3A//z.qzone.com/download.html
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.6,en-US;q=0.4,en;q=0.2
Cookie: pac_uid=0_58b252d129ee3; qqmusic_uin=12345678; qqmusic_key=12345678; pgv_pvi=9429219328; pgv_si=s9007738880; tvfe_boss_uuid=cd2dd5384b16a00f; pgv_info=ssid=s70163163; pgv_pvid=6955230160; uin=; skey=; pt_login_sig=A1C6J6LdSRo8g*lqTJ1CMRgD3tLQDlVASw8UyAC4Kw1gxBrkmCXe2nRr1YphX0pq; pt_clientip=edf1b70e1f98492a; pt_serverip=43040a8187e43833; pt_local_token=690287468; uikey=bfa126f8ca6ec1a5c95fdc7ff4515a163fafa0b203e95c600f4e44b9010cc3d0; pt_guid_sig=e67e6dfb749a1680c4d57fb323fea6bdbbeb4648fce3f3fdfafaf5f0b33d5585; qqmusic_fromtag=30; qrsig=TJchgiXDAODVMNP8xxwJjQDy**TyRv2UmNmH0IIfJ-HGKZRYMh9FVZmgw0qDJITa; _qz_referrer=qzone.qq.com
```

```http
HTTP/1.1 200 OK
Date: Sun, 26 Feb 2017 18:29:35 GMT
Accept-Ranges: bytes
Content-Length: 188
Content-Type: Application/javascript

var var_sso_uin_list=[{"account":"my-qq-id-12345678","face_index":-1,"gender":0,"nickname":"咸鱼。","uin":"my-qq-id-12345678","client_type":66818,"uin_flag":8388608}];ptui_getuins_CB(var_sso_uin_list);
```

That's it. It sets up an HTTP server in `localhost.ptlogin2.qq.com:4301`, later their web page requests that endpoint for identifications.

The most interesting thing told by Chrome is the remote address, `127.0.0.1:4301`. **The remote end is the local**. Indeed, when I queried the DNS, I never thought of a public domain could be configured this way. 

```sh
$ host localhost.ptlogin2.qq.com
localhost.ptlogin2.qq.com has address 127.0.0.1
```

Double check the TCP port 4301, note other unrelated outputs are trimmed.

```sh
$ lsof -n -P -i TCP -s TCP:LISTEN
QQ        33616 longkai   35u  IPv4 0xaa38583952446abf      0t0  TCP 127.0.0.1:4300 (LISTEN)
QQ        33616 longkai   36u  IPv4 0xaa385839490683b7      0t0  TCP 127.0.0.1:4301 (LISTEN)
```

## Security
We just landed the surface. For a product like this, there must be many security considerations in their design. Since I am not good at security, I can only do some small hacks.

It turns out **any client can issue the identity requests**. Note `4300` also appeared in the `LISTEN` port list.

```sh
$ curl -v http://127.0.0.1:4300/pt_get_uins\?callback\=hello
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 4300 (#0)
> GET /pt_get_uins?callback=hello HTTP/1.1
> Host: 127.0.0.1:4300
> User-Agent: curl/7.51.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Mon, 27 Feb 2017 06:33:18 GMT
< Accept-Ranges: bytes
< Content-Length: 188
< Content-Type: Application/javascript
<
* Curl_http_done: called premature == 0
* Connection #0 to host 127.0.0.1 left intact
var var_sso_uin_list=[{"account":"my-qq-id-12345678","face_index":-1,"gender":0,"nickname":"咸鱼。","uin":"my-qq-id-12345678","client_type":66818,"uin_flag":8388608}];ptui_getuins_CB(var_sso_uin_list);
```

Same result.

So, if you want to know the QQ number of the end user, issue the requests in your native client or even the web pages like Tencent does.

**Knowing the QQ number does NOT mean your account is stolen**, however, think of somebody knows your mail address and it's the first phase of doing bad things.

The question is: are you willing to tell your QQ number or something like this to someone you don't even know or malicious?

## Privacy
For the majority, this feature is fine, since it provides a better user experience. Now that I have logged in desktop, the extra login flow on the Web is unnecessary. Moreover, Tencent does passively, only you, the user trigger the identify flow.

Two things should be taken into account, however:

First, as the preceding, you don't want to expose your identity to others, potentially.

Second, for some people with a sense of privacy against being tracked, or potentially in such a situation. As you can see the request message, the Cookie, contains many service identifications. Supposing you just want to view some pages **incognito**, which breaks it.

## How to Fix It?
The good news is you **CAN** control it, certainly, since you already know how it works.

Closing your QQ client is the most effective way. For power users, you can alter the DNS record of that specific host, or ultimately, a proxy in between performing filtering.


---

Thanks for reading.

## EOF
```yaml
date: 2017-02-26T21:12:17+08:00
summary: This post gives a brief intro how does QQ identifies who you are in the browser even you have no intention of login, plus some security and privacy considerations.
weather: cold 😳
license: cc-40-by
location: 22,144
background: qq.jpg
tags:
  - Hacking
  - Security
  - Privacy
```

[Tencent QQ]: https://en.wikipedia.org/wiki/Tencent_QQ
[Qzone]: https://en.wikipedia.org/wiki/Qzone
