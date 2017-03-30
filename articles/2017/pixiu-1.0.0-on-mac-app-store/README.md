Pixiu 1.0.0 on Mac App Store
===
I'm glad to tell you that my first Apple platform app, Pixiu, has been released on the Mac App Store.

[![Mac App Store](https://devimages.apple.com.edgekey.net/app-store/marketing/guidelines/mac/images/badge-download-on-the-mac-app-store.svg)](https://geo.itunes.apple.com/app/id1195433805)

What's Pixiu? Well, the lexical meaning in Chinese is a mythical creature, but the app has nothing to with the name. It's a native Gmail inbox snippets and notifications macOS app based on Gmail API. You can learn more in the [intro page][].

In this post I would like to share some experience about the journey.

## Intention
Why did you build it?

Because I need one. I like the web version and it's full-fledged except a lack of a native notifications functionality which sometimes cause you trouble(e.g. browser not opened or account not signin).

Another reason lies in that I have been always wanting to try building something on the Apple platform.

It seems promising this time.

## Swift/Cocoa
Swift is a new language, and a *new* means powerful most of the time since it *stood on the shoulders of other languages*. It's also easy to learn especially you have other language(s) background. The *The Swift Programming Language* book by Apple is a good start.

Sometimes, a framework is more important than the language itself in a project's perspective. A framework is more difficult than a language, as of Cocoa, the macOS native application development framework, I just learned the tip of the iceberg.

## Development
The project is written in Swift plus some C code. Apart from fabric, a crash report framework, there is no third part libraries.

The actual development time(writing code) is relatively short. Most of the time was spent on digging into the [Gmail API][], the strict Google OAuth flow, and the hidden bug fixes, so on and so forth.

First, Gmail API is really great, however, it does have some defects. Take the inbox unread mails for example, you can not get a mail's title/sender/time/etc. directly in a list style API, instead have to request another API. For a user with a large number of unread mails, doing so is bound to trigger the API quota limit per user per second if performing many requests concurrently. You have to do extra job to keep things working.

Second, the Google OAuth flow is strict, plus a bit unclearness. Different platforms have different ways. For a mobile app is easy, not to mention Google provides libraries. For a desktop app, your choices are a redirecting authentication code to a local http server or letting user copy-paste code manually... The former is bad for sandbox if publishing to App Store. It turns out that you have workarounds, but Google's doc doesn't reveal.

Finally, a new language is not always good, it lacks some features or libraries you have been familiar with. Since I can only read Objective-C code instead of writing, I have to write some C code. Swift does have a good support for C except the wired long syntax.

There are does other development aspects to talk.

- [Lessons From Gmail API - a Batch Request Design Doc](../lessons-from-gmail-api-a-batch-request-design-doc)

## Review
First app and first submit, indeed, Apple's review is more than I thought. It took almost 3 weeks, with many rejections and submits.

- missing privacy policy
- crashed in old macOS. It's really hard to debug a native application in old OS.
- use a restrict C API. Like I said before, I wrote a small C http server for Google OAuth redirections, however Apple rejected for only server apps can use that API.
- network issue. Hard to find this bug, see my accepted [answer][] on stackoverflow.
- rejected for a crash but the crash report belongs to another app
- illegibility of a icon

A long way, reached the *ready for sale* state.

## One More Thing
Today is the Chinese traditional [Lantern Festival(元宵节)][Lantern Festival], 元宵节快乐🎉

Thanks for reading.

## EOF
```yaml
summary: Share the intention, learning, development, and submit experience of the Pixiu app
weather: fine
license: cc-40-by
location: 22,144
background: ../../../apps/pixiu/banner.jpg
tags:
  - apps
date: 2017-02-11T15:54:43+08:00
```

[intro page]: https://xiaolongtongxue.com/apps/pixiu/
[Gmail API]: https://developers.google.com/gmail/api/
[answer]: https://stackoverflow.com/questions/41461481/error-domain-nsposixerrordomain-code-100-protocol-error/41988623#41988623
[Lantern Festival]: https://en.wikipedia.org/wiki/Lantern_Festival

