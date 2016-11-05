Enable Spotlight Suggestions
===
If you're using Mac, this article may appeal to you.

## What's it?
According to Apple,

> Spotlight helps you quickly locate files on your Mac, and more.

You should read [this][apple-desc] if you have no idea about it. In short, it improves your productivity **quickly**, search files, open/switch apps, lookup words, do calculation, search online with wiki, twitter, etc.

Below is a video showcase in action(if you cannot open it, try [this][qq-video]).

[![Spotlight Demo on Vimeo](showcase.png?imageView2/2/w/512/format/jpg)](https://vimeo.com/190262280)

Come on, try on your Mac, you would find it helpful. However, some of you may notice that your spotlight doesn't show any **online results** but local stuff. If you did see it, free feel to skip the following section and enjoy your day:)

Why I cannot see it? Well, because you're not int the [Apple's feature availability][avaiable-list] list.

But, how can I enable it?

## Disclaimer
- As of this writing(**Nov 2016**), things would change in the future and this article may not work for you.
- Test on macOS 10.12.1 only, not sure other environments.

Don't worry, you can still give it a try with hints providing in this post.

## Pre-requisite
- A little knowledge about Web, i.e. HTTP.
- A proxy server lives in the [list][avaiable-list] and a client on your local Mac.

Actually, it's all about **how to use a proxy application**.

## How It Works?
Since the feature only available in a certain of country, Apple must know where you are when you using Spotlight. As far as I know, there're three ways that one remote end can **guess** your location.

1. **Language & Region** based location.
2. **GPS** based location.
3. **IP Geographic** based location.

Tried all of them and I suppose Apple use solution 2 and fallback to solution 3. No matter what methods it uses one things is for sure: **they talk via network**, i.e. HTTP protocol.

So, by disabling GPS when searching then letting the search requests go through an HTTP proxy, we're done!

## In Action
### System Settings
Sequentially,

1. System Preferences
2. Security & Privacy
3. Privacy
4. Unlock
5. scroll to the end of the list
6. Details...
7. uncheck Location-Based Suggestions
8. Done and Lock it

### Apply Proxy Rules
By watching the HTTP requests, the two **second level domain** Apple uses for Spotlight suggestion are,

- `configuration.apple.com`
- `ls.apple.com`

Hence, put this two domain to your proxy rule list(or global proxy which may cause network laggy).

To be clear, any domain has these two **suffix** should go through a proxy, i.e. `a.b.c.ls.apple.com`.

## Further Thoughts
It works for me very well, however, there're still some things should be taken into count:

* The Wikipedia not work in the *Dictionary* app
* Privacy, since all our search keywords exposed to Apple
* By disabling the GPS and IP rule proxy, I'm not sure it would break any functionality

For the former, it's not a big deal since wiki works in Spotlight but if you make it work, please let me know. The latter, I treat it as a reminder, protecting our privacy from time to time. The last, basically not, it only has something to do with the Location-Based Suggestions but it's all about assumption.



Thanks for reading.

## EOF
```yaml
date: 2016-11-04T23:14:40+08:00
summary: Enable Spotlight suggestions where not available in your country.
weather: a bit cold
license: cc-40-by
location: 22, 114
background: spotlight.png?imageView2/2/w/1024/format/jpg
tags:
  - Hacking
```

[avaiable-list]: http://www.apple.com/ios/feature-availability/#spotlight-suggestions-spotlight-suggestions
[apple-desc]: https://support.apple.com/en-us/HT204014
[qq-video]: https://v.qq.com/x/page/m03438kx2xx.html

