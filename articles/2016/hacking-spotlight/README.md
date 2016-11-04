Hacking Spotlight
===
If you're using Mac, this article may appeal to you.

## What's Spotlight
According to Apple,

> Spotlight helps you quickly locate files on your Mac, and more.

You should read [this][apple-desc] if you have no idea about it. In short, it improves your productivity **quicklly**, search files, open apps, lookup words, search online with wiki, twitter, etc.

Here is a showcase in action.

[![Spotlight Demo on Vimeo](showcase.png?imageView2/2/w/512/format/jpg)](https://vimeo.com/190262280)

Come on, try on your Mac, you would find it helpful. However, you may notice that your spotlight doesn't show any **online results** but local stuff. If you did see it, free feel to skip the following section and enjoy your day:)

Why I cannot see it? Well, because you're not int the [Apple's feature availability][avaiable-list] list.

But, how can I enable it?

## Disclaimer
- As of this writing(11/2016), things would change in the future and this article may not work for you.
- Only test on macOS 10.12.1, not sure other environment.

Don't worry, you can give it a try with hints providing in this post.

## Pre-requisite
- A little knowledge about network, i.e. HTTP.
- A proxy server lives in the [list][avaiable-list] and a client on your local Mac.

Actually, it's all about **how to use proxy application**.

## How It Works?
Since the feature only available in a certain of country, Apple must know where you are when you using Spotlight. As far as I know, there're three ways that one remote end can **guess** your location.

1. **Language & Region** based location.
2. **Geographic** based location via GPS.
3. **IP** based location.

Tried all of them and I suppose Apple use solution 2 and fallback to solution 3. No matter what methods it uses one things is for sure: **they talk via network**, i.e. HTTP protocol.

So, by disabling GPS when searching then letting the search requests go through a proxy, we're done!

## In Action
### Disable Location Based Suggestions
System Preferences -> Security & Privacy -> Privacy -> Unlock -> scroll to the end of the list -> Details... -> uncheck Location-Based Suggestions

### Proxy Rules
By watching the HTTP requests, the two **second level domain** Apple uses for Spotlight suggestion is,

- `configuration.apple.com`
- `ls.apple.com`

Hence, put this two domain to your proxy rule list(or global proxy which may cause network laggy).

Note, to be clear, any domain has these two **suffix** should go through proxy.

## Remaining Issues
It works for me very well, however, there're still some things should be taken into count:

- [ ] The Wikipedia not work in the *Dictionary* app
- [ ] Privacy, since all our search keywords go through Apple

For the former, if you make it work, please let me know. The second, I treat it as a reminder, protecting our privacy from time to time.



Thanks for reading.

### EOF
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
