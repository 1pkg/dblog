---
author: 'Kostiantyn Masliuk'
title: 'Private repository dependency management with gomod'
date: '2021-03-20'
---

According to [go survey 2020](https://blog.golang.org/survey2020-results), gomod has finally won long disputes around dependency management in go: "Go modules adoption is nearly universal with 77% satisfaction, but respondents also highlight a need for improved docs". This is not surprising at all, as it is indeed a solid tool, but even more importantly, included into default go distribution as the official dependency management tool. With that said, gomod is not ideal (as any other complex tool) sometimes you can stumble upon some [controversial decisions](https://github.com/golang/go/issues/26366) or need to know some [internal implementation details](https://proxy.golang.org/) to use it properly in some less common setups.

I had to configure some new gomod environment for a private repository recently. I vaguely remembered that simply using gomod out of the box is coupled with some source code security concerns. After a quick research, I found a kind of [confirmation](https://goproxy.io/docs/GOPRIVATE-env.html) of my assumption: you can also read the official answer to - what will happen if you do not set `GOPRIVATE` env variable properly [here in FAQ](https://proxy.golang.org/).

```
If I don't set GOPRIVATE and request a private module from these services, what leaks?

The proxy and checksum database protocols only send module paths and versions to the remote server.
If you request a private module, the mirror will try to download it just as any Go user would and fail in the same way.
Information about failed requests isn't published anywhere.
The only trace of the request will be in internal logs,
which are governed by the [privacy policy](https://proxy.golang.org/privacy).
```

As goproxy.io states "gomod defaults work well for publicly available source code", though this behavior might be far from ideal for your private repository. In my specific case, gomod default behavior was not really a big deal, even if `GOPRIVATE` is not set properly. I also believe that most of the private projects can tolerate unwanted request trace appearance in internal google logs, but there are obvious exceptions to this if you require to have source code protection in place.

One extra interesting aspect that might hurt a private go module is proxy immutability. If private module gets to proxy somehow (even by mistake) there is no simple turning back [see FAQ again](https://proxy.golang.org/).

```
I removed a bad release from my repository but it still appears in the mirror, what should I do?

Whenever possible, the mirror aims to cache content in order
to avoid breaking builds for people that depend on your package,
so this bad release may still be available in the mirror even if it is not available at the origin.
The same situation applies if you delete your entire repository.
We suggest creating a new version and encouraging people to use that one instead.
```

From that point in time, I became curious: if there any simple way how to check if proxy contains my private module "x". Unfortunately, gomod doesn't have anything builtin for this. But proxy provides API "index.golang.org" that contains feed of all added modules. So in theory, we can use this API to create teeny-tiny cli tool to query gomod and cache modules feed locally for the faster access. I've decided to go ahead and create such tool, plus add some extra semver filtering, modules sorting, and other options to make the tool more generic and easy to use. The tool can be found here on github [gomer](https://github.com/1pkg/gomer).

```
gomer -h
Usage of gomer: [-f val] [-f val] <module_path_regexp>
  -constraint string
        cli semver constraint pattern; used only if non empty valid constraint specified
  -format string
        cli printf format for printing a module entry; \n is auto appended (default "%s %s %s")
  -index string
        cli go module index database url; golang.org index is used by default (default "https://index.golang.org/index")
  -nocache
        cli modules no caching flag; caching is enabled by default (default false)
  -timeout int
        cli timeout in seconds; only considered when value bigger than 0 (default 0)
  -verbose
        cli verbosity logging flag; verbosity is disabled by default (default false)
```

P.S.

While using [gomer](https://github.com/1pkg/gomer) I've also made one observation. Cached modules feed (reused for fast modules search) is partitioned by original module upload time (more precisely by original upload month). Looking at created partitioned cache files, we can see that size of cached files is growing superlineary from the start of gomod official launch till now (which also means that number of go modules growing in the similar fashion).

```
ls -lh | awk '{print $5,$9}'
1.6M page_2019-04-10T....json
1.2M page_2019-05-10T....json
2.2M page_2019-06-09T....json
9.7M page_2019-07-09T....json
4.3M page_2019-08-08T....json
14M page_2019-09-07T.....json
18M page_2019-10-07T.....json
7.1M page_2019-11-06T....json
18M page_2019-12-06T.....json
9.2M page_2020-01-05T....json
9.8M page_2020-02-04T....json
22M page_2020-03-05T.....json
15M page_2020-04-04T.....json
14M page_2020-05-04T.....json
14M page_2020-06-03T.....json
12M page_2020-07-03T.....json
12M page_2020-08-02T.....json
12M page_2020-09-01T.....json
11M page_2020-10-01T.....json
21M page_2020-10-31T.....json
47M page_2020-11-30T.....json
35M page_2020-12-30T.....json
55M page_2021-01-29T.....json
```

This is expected as more and more people starting to use gomod as primary dependency management tool and nicely corresponded with results of [go survey 2020](https://blog.golang.org/survey2020-results).
