---
layout: default
title: Contributing to Gemini ecosystem
---

# {{ page.title }}

*{{ page.date | date_to_string }} on [Peter Vernigorov's blog](/)*

About a year ago I found out about [Gemini protocol](https://gemini.circumlunar.space/). For those who aren't familiar with it, it's a new Internet protocol (which together with its accompanying gemtext format) aims to build an alternative "web, stripped right back to its essence". It appealed to my love for minimalism and I built a few things for its ecosystem.

Regardless of your views of Gemini, or familiarity, **the greater lesson here is that even with a simple protocol, useful programs almost never are**. Similar messages are echoed by others such as [Text Editing Hates You Too (2019)](https://lord.io/text-editing-hates-you-too/).

This post requires no knowledge of Gemini protocol specifications. Each section focuses on one project, explaining why and how it was built. Feel free to skip "How" sections as it can go deep into technical details.

## iOS Browser

**What** [Elaho](https://github.com/pitr/gemini-ios) is a fully featured iOS browser with features people are used to, such as tabs, bookmarks, history, etc. This is the first project I worked on. While mostly a backend engineer by trade, I've delved into the client side of things quite a few times. But this was my first published iOS app.

![Elaho screenshots](/images/elaho.png)

**Why** At the time, most clients for Gemini were terminal based, with only one or two focusing on Desktop GUI. I realized that more so than the web, Gemini is almost exclusively meant for consuming content. There is only one way to send information from the client, using something not unlike a one textbox form. Today, around 60% of content consumers online are on mobile/tablet devices, and it's growing with every year [[source]](https://www.perficient.com/insights/research-hub/mobile-vs-desktop-usage). Within those, while Android has a larger market share than iOS, I've chosen the latter as I have the most experience with it.

**How** Currently, Gemini-specific code accounts for 2% (1k of 41k) of lines of code. This includes Gemini protocol, certificate helpers, gemtext rendering and ansi escape codes. Most of the other code deals with UI for features such as tabs, session restoration, bookmarks, share extension etc. Foreseeing this, I decided to use an existing browser that already has most of this functionality included - [Firefox for iOS](https://github.com/mozilla-mobile/firefox-ios). It is licensed under Mozilla Public License 2.0, which allows me to fork it as long as copyright and license notices are preserved. Of interest is [my first commit](https://github.com/pitr/gemini-ios/commit/a3c3bc1e), which removed 30k lines worth of telemetry/tracking code: Adjust, Leanplum, Sentry.

Most of the code is responsible for UI. While I could benefit from view layer code responsible for managing bookmarks, history, and others, the underlying storage layer which relied on Rust-based [Firefox Application Services](https://github.com/mozilla/application-services) had to be replaced with [Realm](https://realm.io/).

Other mobile browsers for Gemini took a different route. [Ariane](https://oppen.digital/software/ariane/) and [Phaedra](https://oppen.digital/software/phaedra/) target Android devices, [Deedum](https://github.com/snoe/deedum) uses Flutter framework, [Rocketeer](https://git.shadowfacts.net/shadowfacts/Gemini) used SwiftUI, and [Lagrange](https://gmi.skyjake.fi/lagrange/) is built on a custom-made cross-platform framework written in C. All were built from scratch and (apart from Lagrange, author of which has put in an impressive amount of time into the project) do not support most browser features expected by users. Examples include things like session restoration, URL bar with autocomplete, custom keyboard and without spell check, and horizontal slide gesture to go back/forward. The reason is that the sheer amount of work needed to build all of this auxiliary functionality seems to result in most of these projects being abandoned by their authors.

**Result** Since it's been in the app store, I've seen about 20 weekly installs, and after releasing a new version I get about 3000 updates. It has been mentioned on [AppleVis](https://www.applevis.com/apps/ios/utilities/elaho) (online resource for blind and low vision users of Apple products).

## Server Framework

**What** [Gig](https://github.com/pitr/gig) is a Gemini framework written in Go that helps quickly build dynamic Gemini servers. It is very similar to [Gin](https://github.com/gin-gonic/gin), [Echo](https://echo.labstack.com/) and other projects for HTTP servers.

**Why** At the time, apart from a couple capsules online (search engine and a gardening experience game), all resources were static. Most servers at the time (and this is still true as of 2021) only supported serving static files. I had a few ideas for dynamic resources (see other projects below) and a simple framework would minimize work required later.

**How** Borrowing from other Go frameworks, this project has a zero-allocation router, extensive documentation, lots of rendering helpers, a few builtin middlewares such as `Logger`, `Recover`, `ValidateHasCertificate` and support for custom ones. While HTTP frameworks can rely on `net/http` package, Gig needed to implement Gemini protocol specification. Each route handler gets a Context (an interface containing request/response helpers) similar to Gin and Echo, and after the request is done the Context is later reused thanks to `sync.Pool`.

As I used it to build other projects, I was able to iterate on the framework itself. Context interface was improved, helpers to return an error to client were simplified, a middleware for login/password authentication was added.

**Result** In the beginning Gig was primarily used by me. However, in the last few months a number of servers came up that imported it (see [Who uses Gig](https://github.com/pitr/gig#who-uses-gig)). I even found a [live stream on YouTube](https://www.youtube.com/watch?v=lO51xCvZ3GI) that features Gig being used to build a microblogging app.

## Link Aggregator

**What** [Geddit](https://github.com/pitr/geddit) is similar to Hacker News, Reddit.

**Why** In late 2020 new capsules were popping up weekly if not daily, and some were usually posted to the mailing list. These emails were in-between other discussion threads, such as protocol finalization, protocol ideas, questions, etc. I came to a conclusion that a different mechanism to discover and share new capsules/pages was needed. Link aggregation services work relatively well for this in my experience, so I set out to build one for Gemini.

**How** Thanks to Gig, it was quite simple to build using ModelViewController pattern. Controller layer, including CRUD for posts and comments, contains about [155 sloc](https://github.com/pitr/geddit/blob/master/main.go). Model layer, built on GORM+SQLite, is another [100 sloc](https://github.com/pitr/geddit/blob/master/db/db.go). Views use `text/template` format. The project compiles into a small static binary that can be rsync'ed to the server.

**Result** As of May 2021 there were 160 links submitted and 250 comments posted. There are at least a few submissions a week, and it has been mentioned on the mailing list many times. I continue to check it myself every few days. Other similar aggregators and forum-like capsules went live but none have maintained a consistent usage by the community.

## Wikipedia Proxy

**What** An interface to Wikipedia from Gemini, available at `gemini://wp.glv.one`.

**Why** Capsules are usually built around content and functionality. Most content is produced by authors, with blogs as a prime example. However, there is a lot of content already available elsewhere on the Internet, and Gemini could be a good interface to consume it. Wikipedia content is licensed under [Creative Commons Attribution-ShareAlike 3.0 Unported License](https://en.wikipedia.org/wiki/Wikipedia:Text_of_Creative_Commons_Attribution-ShareAlike_3.0_Unported_License) and it can be remixed (in this case, converted to Gemini text) and shared (distributed at wp.glv.one). Gemtext format is a simplified version of Markdown, so the challenge would be to simplify an already content-rich format. Another challenge is to parse the Wikitext format.

**How** [Backend](https://github.com/pitr/wp) is built on my Gig framework, [go-mwclient](cgt.name/pkg/go-mwclient) for Wikipedia client, and [github.com/d4l3k/wikigopher/wikitext](https://github.com/d4l3k/wikigopher) for Wikitext parsing.

Wikitext package is part of a similar project for gopher space. It contains a PEG grammar and a parser. It was extremely easy to incorporate this into my project. Once an AST (Abstract Syntax Tree) is generated, my code walks the tree and generates Gemtext.

The first challenge was links in text. Gemtext allows text OR link per line, but not both. Wikipedia articles usually contain a LOT of inline links to related articles so it's important that these links are kept close to the text. The solution I came up with was to list all links directly AFTER the paragraph, by remembering links as they are parsed and storing them in a Links structure. Once a paragraph break was detected, it was a matter of simply flushing those links and resetting.

```go
type Links struct {
	buf strings.Builder
}

func NewLinks() *Links {
	return &Links{}
}

func (f *Links) Add(name, href string) {
	href = strings.TrimSpace(href)
	f.buf.WriteString(fmt.Sprintf("=> %s %s\n", href, name))
}

func (f *Links) String() string {
	return f.buf.String()
}

func (f *Links) Reset() {
	f.buf.Reset()
}
```

![Wikipedia screenshot](/images/wikipedia.png)

Anything that is not a paragraph, heading text, list or link is skipped. This includes footnotes, tables, images, complex structures like {Infobox}, etc.

**Result** While there is an active usage of it, I consider this project a failure. Simply looking at the amount of content that is skipped due to inability to represent it in Gemtext format renders it quite useless.

But perhaps the bigger reason for its failure is the performance factor. This app constantly uses most of the memory and cpu on my server, which profiling showed is due to the parsing library. Latency is also quite large, especially for large pages, again due to slow parsing. Most articles take from a few seconds to more than a minute to load. There are [quite a few Wikitext parsers](https://www.mediawiki.org/wiki/Alternative_parsers), but unfortunately none that are written in Go.

Another Wikipedia Proxy project by [Alex Schroeder](https://alexschroeder.ch/) hosted at `gemini://vault.transjovian.org/`. It is built in Perl and uses [a lot of regular expressions](https://alexschroeder.ch/cgit/phoebe/tree/contrib/wikipedia.pl?id=c536e0fc9175e33a3b002953019f7d3763356d4b#n163) to convert Wikitext to Gemtext. In the end its latency is quite good.

On a related note, a [few other proxy capsules](https://portal.mozz.us/gemini/gempaper.strangled.net/mirrorlist/) have come up for resources like Reddit, Hacker News, and newspapers such as Guardian, Deutsche Welle, CNN, NPR. Much of this content is licensed less permissively, and I think it does more damage than good to the Gemini community.

## PaaS

**What** glv.one is a Platform-as-a-Service, similar to Heroku. Deployment is done by pushing a new docker image to a private registry, and the administration interface is available over Gemini at `gemini://glv.one`.

**Why** Having built a few capsules using Gin, I wanted to be able to quickly create new ones. Deploying a new capsule with glv.one is as easy as clicking "New" in the admin interface, and then pushing the image to `deploy.glv.one/$USER/$APP`. I also saw a potential need in the community to have a place to publish dynamic capsules without paying for your own server. Free static hosting is offered by many capsules such as `gemini.circumlunar.space`, `flounder.online`, `gemlog.blue` and `srht.site`, but there was no place to host dynamic content.

**How** This is the only project that I did not open source, mostly because there isn't much to open source. It is built by glueing different parts together.

1. Docker images are pushed to a private [Docker Registry](https://hub.docker.com/_/registry). Only requests from `docker login` and `docker push` are allowed. Registry is password protected with `htpasswd`, and has a webhook setup to notify a little Go server of pushes:

	```
auth:
  htpasswd:
    path: /.htpasswd
notifications:
  events:
    includereferences: true
  endpoints:
    - url: http://172.17.0.1:8080/event
      ignore:
        actions:
           - pull
	```

2. The tiny Go server parses the webhook event, makes sure that the $USER pushed the new image to `/$USER/$APP`, an app that he owns. Then it runs `docker stop $APP && docker rm $APP` (to stop the previous version), and `docker run -d --restart=always --name $APP $IMAGE`. There are a few more arguments to limit cpu/memory resources and to mount a volume.

3. Each app gets its own local port, which is mapped to port 1965 in the container.

4. Nginx runs on port 1965 and streams requests to the correct capsule's port based on the request's [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) (Server Name Indication):

	```
stream {
    map $ssl_preread_server_name $upstream {
        include /etc/nginx/stream.d/*.conf;
    }

    server {
        listen      1965;
        proxy_pass  $upstream;
        ssl_preread on;
    }
}
	```

5. When creating a new app, it simply needs to write `"$APP.glv.one" 127.0.0.1:$PORT;` to a file in /etc/nginx/stream.d/ and request nginx to reload its configurations.

6. Admin interface offers additional functionality such as viewing logs (by sending back `docker logs $APP`) and deploying an old version (by sending a fake push event with that image ID).

**Result** While there was no community adoption (every new user must be manually whitelisted by emailing me, I received 0 emails), I have used GLV.One for all my apps. Despite being quite hacky (and probably not secure), it works surprisingly well.

## Summary

There were a few smaller projects I built in the Gemini universe that I won't go into, as this article is long enough. Overall I enjoyed being part of the community and building projects that try to solve real problems.