Intro xiaolongtongxue v2.0
===
Turn markdowns into website with Github, Docker, and more.

It builds upon **Github Fav Markdown API**, rendering from a plain markdown repo to a nice website. Moreover, it supports **auto update** when you push commits to Github.

It's highly **customizable** and even has a docker image for build-run-ship easily.

The source code and docker image available on [Github][repo], [Docker Hub][docker], respectively.

## Features
- **Auto** Using Git/Github to keep your writing workflow, when you push your work to Github, your website will sync changes automatically
- **Standard** [Github Fav Markdown][github fav md] rendering style and API
- **Docker** Run right from Docker
- **Fast** With Non-blocking architecture, not really a static but dynamical
- **Configurable** You can modify for your needs
- **Support CDN** Put all your static stuff to *CDN*(Only tested qiniu)
- **Support Medium** Auto posting your new works when pushing to Github

## Markdown format Requirement
### Each Doc Must...
1. resident in a directory, one per one
2. file name ends with `.md`, prefer `README.md`
3. have an `EOF` [Fenced code block][Fenced code block], all the rest has no restricts

Note the format is(at least one `#`),

```md
### EOF
{{yaml fenced code block}}
```

```yaml
--- sample, only `date` is required
title: # only required if not specify in markdown
date: 2016-01-07T02:50:41+08:00 # required, must be this format(i.e., RFC3339)
hide: false # if true this article won't appear in the list
summary: summary for this article
weather: hey, what's the weather like?
license: # "all-rights-reserved", "cc-40-by", "cc-40-by-sa", "cc-40-by-nd", "cc-40-by-nc", "cc-40-by-nc-nd", "cc-40-by-nc-sa", "cc-40-zero", "public-domain". The default is "all-rights-reserved".
location: somewhere 
background: banner image for this article
tags:
  - tag1
  - tag2
  - ...
```

Take a look at a full [sample][sample]. There is even a command-line tool for auto-generating this format, checkout the [source](cmd/newmd) or download [here][dl].

## Run with Docker
Run `docker run -d -p 1217:1217 -v /path/to/repo:/repo -v /path/to/env.yaml:/env.yaml:ro longkai/xiaolongtongxue.com` Don't forget to replace your volumes.

Or, if you prefer `docker-compose`, modify for your needs,

```yaml
sakura:
  image: longkai/xiaolongtongxue.com
  ports:
    - "1217:1217"
  volumes:
    - /path/to/env.yaml:/env.yaml:ro
    - /path/to/repo:/repo
```

then run `docker-compose up -d`

## Build Manually
### Pre-requisite
- [golang][go] >= 1.7
- [bower][bower]

### Building
1. `go get github.com/longkai/xiaolongtongxue.com && rm $GOPATH/bin/xiaolongtongxue.com`
2. `cd $GOPATH/src/github.com/longkai/xiaolongtongxue.com`
3. `./build.sh`
4. `./xiaolongtongxue.com [/path/to/env.yaml]`

## Configuration
```yaml
--- env.yaml
port: 1217
repo: /repo
hook_secret: Github WebHook secret
access_token: Github Personal access token
#medium_token: Medium Self-issued access tokens
meta:
  ga: GA tracker ID
  gf: false # Use Google Fonts, check `templ/include.html`
  #cdn: CDN origin # currently only tested qiniu
  origin: https://your-domain.com # (i.e. `window.location.origin` in JS) required only if using CDN or medium posting service
  bio: something about you
  link: other link about you
  lang: zh
  name: your name
  title: page title
  mail: you@somewhere
  domain: domain.com # optional, for multiple sub-domain tracking
  github: your Github link if nay
  medium: medium repo if any
  twitter: twitter link if any
  instagram: ins link if any
  stackoverflow: stackoverflow link if any
ignores:  # NOTE: the path is **HTTP RequestURI** format
  - '^/[^/]+\.md$' # ignore *.md in root dir
```

Remember in the `assets/images/` there are some placeholder images, you would like to replace with yours.

Note if you use docker image with which container has a mounted repo, the `repo` in the `env.yaml` and the docker mount pointer MUST be same.

## Github Hook&Markdown Support
1. obtain your github *personal access token* from your settings
2. goto your markdown repo settings, in *Webhooks & services* Tab add a webhook with **Payload URL** `your-domain.com/api/github/api`, then set the **Secret**, note the *push* event is required. 

Don't forget to set this information in `eny.yaml`!

## CDN Support
I only tested *qiniu CDN* which can fetch then cache your site stuff for a given url. You must set your site url with prefix `/cdn/` to qiniu, then specify the CDN domain in `env.yaml`.

## Medium Support
The [official Medium API][medium] only allows posting new stuff to their side(e.g., editing or deleting are not supported). Note the limitation before you plugin it.


Happy hacking.

### EOF
```yaml
date: 2016-09-12T21:01:17+08:00
hide: false
summary: Turn markdowns into website with Github, Docker, and more.
weather: hey, what's the weather like?
license: cc-40-by
location: somewhere 
background: blue-sky.png
tags:
  - Leaning
  - Hacking
```


[repo]: https://github.com/longkai/xiaolongtongxue.com
[docker]: https://hub.docker.com/r/longkai/xiaolongtongxue.com/
[github fav md]: https://guides.github.com/features/mastering-markdown/
[Fenced code block]: https://help.github.com/articles/creating-and-highlighting-code-blocks/
[sample]: https://raw.githubusercontent.com/longkai/xiaolongtongxue.com/master/render/testdata/normal.md
[dl]: https://dl.xiaolongtongxue.com/newmd/
[go]: https://golang.org/
[bower]: https://bower.io/
[medium]: https://github.com/Medium/medium-api-docs/issues/52
