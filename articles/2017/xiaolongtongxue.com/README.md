# xiaolongtongxue.com

Turn your favorite markup git repository as a website, with Github favorite rendering layout, automatically.

## Features

- Layout with Github favorite rendering look and feel.
- Support or mix many markup formats, [markdown][], [org-mode][], [rst][], etc.
- The moment push your work to the Github, your website will sync changes automatically.
- Misc configuration like posting new documents to Medium, controlling which file to be rendered, skipped, hide, redirections and so forth.

## Document Format Requirement

### Each Doc Must...

- Resident in a directory, one per one.
- Have an `EOF` headline followed by a `YAML` code block.

For markdown:

    ## EOF
    
    ```yaml

    ```

For org-mode:

    ** EOF
    
    #+BEGIN_SRC yaml
    
    #+END_SRC

For rst:

    EOF
    ---
    
    .. code-block:: yaml

Other markup format should have similar syntax, there must be an `yaml` source type indicator.

### YAML Fields for Documents

Note: the `date` is the only required and **MUST place as the last one**.

```yaml
summary: # summary for this article
hide: false # if true this doc won't appear in the list, can still accessed by URL, however.
title: # force the doc's title, in case the parsed title not your want.
weather: # hey, what's the weather like?
license: # "all-rights-reserved", "cc-40-by", "cc-40-by-sa", "cc-40-by-nd", "cc-40-by-nc", "cc-40-by-nc-nd", "cc-40-by-nc-sa", "cc-40-zero", "public-domain". The default is "all-rights-reserved".
location:  # where did you wrote this?
background: # banner image for this doc.
tags: [tag1, tag2, ...]
date: 2016-01-07T02:50:41+08:00 # required, must be this format(i.e., RFC3339)
```

There is even a command-line tool for auto-generating this format, go [get it](cmd/newdoc)!

## Run with Docker

Run `docker run -d -p 1217:1217 -v /path/to/repo:/repo -v /path/to/conf.yml:/conf.yml:ro longkai/xiaolongtongxue.com` Don't forget to replace your volumes.

Or, if you prefer `docker-compose`, [modify](docker-compose.yml) for your environment:

```yaml
essays:
  image: longkai/xiaolongtongxue.com:latest
  ports:
    - 1217:1217
  volumes:
    - $PWD/log.txt:/log.txt
    - $PWD/conf.yml:/conf.yml:ro
    - $HOME/src/github.com/longkai/essays:/repo
  restart: always
```

Run `docker-compose up -d`.

## Build Manually

### Pre-requisite

- [golang][go] >= 1.7
- [bower][bower]

### Building

1. `go get github.com/longkai/xiaolongtongxue.com && rm $GOPATH/bin/xiaolongtongxue.com`
2. `cd $GOPATH/src/github.com/longkai/xiaolongtongxue.com`
3. `./build.sh`
4. `./xiaolongtongxue.com [/path/to/conf.yml]`

## Configuration

Note wildcard `**` is not supported by Golang.

```yaml
port: 1217 # Server listen port.
repo_dir: /repo # Local Github documents repo location.
medium_token: # Medium Self-issued access tokens, optional.
glob_docs: # The markup file name wilcards.
  - README.* # Any file start with README.xxx would be parse as an articles.
skip_dirs: # Skip parse dirs.
  - .* # Skip hidden dirs
  - assets # assets dir is resered as front-end resources.
github:
  user: # Github user name
  repo: # Github repo name
  hook_secret: # Github WebHook secret
  access_token: # Github Personal access token
meta: # optional
  ga: GA tracker ID
  gf: false # Use Google Fonts, check `templ/include.html`
  origin: https://your-domain.com # required only if medium posting service enabled.
  bio: something about you
  link: other link about you
  name: your name
  title: page title
  mail: you@somewhere
  github: your Github link if nay
  medium: medium repo if any
  twitter: twitter link if any
  instagram: ins link if any
  stackoverflow: stackoverflow link if any
redir: # Redirection mapping, i.e., HTTP 301
  "/path/to/old": "/path/to/new"
```

Remember in the `assets/images/` directory there are some placeholder images, you would like to replace them with yours.

## Front-end template

I am not good at front-end development, so there is only one template site I grab from the Internet. If you would like to replace your site's design, you need to understand a little about [template usage][go-template] of Golang, it's not hard.

## Github Hook Service

1. Obtain your Github *personal access token* from your settings
2. Go to your repo settings, in *Webhooks & services* Tab add a webhook with **Payload URL** `your-domain.com/api/github/api`, then set the **Secret**, note the *push* event is required. 

Don't forget to set this information in your `conf.yml`!

## Medium Support

The [official Medium API][medium] only allows posting new stuff, i.e., editing or deleting are not supported.


## 

Happy hacking.

## EOF
```yaml
hide: false
summary: Turn markdowns into website with Github, Docker, and more.
weather: hey, what's the weather like?
license: cc-40-by
location: somewhere 
background: blue-sky.jpg
tags:
  - Leaning
  - Hacking
date: 2016-09-12T21:01:17+08:00
```

[markdown]: https://guides.github.com/features/mastering-markdown/
[org-mode]: http://orgmode.org
[rst]: http://docutils.sourceforge.net/docs/ref/rst/restructuredtext.html
[go]: https://golang.org/
[bower]: https://bower.io/
[medium]: https://github.com/Medium/medium-api-docs/issues/52
[go-template]: https://golang.org/pkg/text/template/

