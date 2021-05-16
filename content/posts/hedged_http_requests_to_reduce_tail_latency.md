---
author: 'Kostiantyn Masliuk'
title: 'Hedged http requests to reduce tail latency'
date: '2021-05-16'
---

Recently, I stumbled upon [an article from Google engineers](https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext) that describes ways of reducing tail latency used inside Google. Among obvious methods like: managing background activities and adding priorities queues on various system levels. There were also listed multiple methods, efficiency of which might look less evident at the first glance. Notably, in "within request short-term" section:

```
Hedged requests. A simple way to curb latency variability is to issue the same request
to multiple replicas and use the results from whichever replica responds first. ...
The client cancels remaining outstanding requests once the first result is received.
Although naive implementations of this technique typically add unacceptable additional load,
many variations exist that give most of the latency-reduction effects while increasing load only modestly. ...
```

_and_

```
Tied requests. ... The simplest form of a tied request has the client send the request to two different servers,
each tagged with the identity of the other server ("tied"). When a request begins execution, it sends a
cancellation message to its counterpart. The corresponding request, if still enqueued in the other server,
can be aborted immediately or deprioritized substantially. ...
```

But examples for BigTable provided further in that article, make it clear that both hedged and tied requests are really efficient in reducing tail latency at scale. While tied requests technique demands cooperation between both client and server sides and isn't so trivial to implement in many situations. Hedged requests on another hand is the technique that purely resides in the client-caller code and can be rather easily implemented to be useful for any generic http request. In result, I spent some time creating my own generic [hedgehog library](https://github.com/1pkg/hedgehog) for http hedged transport in go. It provides http hedged transport wrapper for go std http client and multiple resource types to flexibly configure hedged behavior. There are 3 types of resources provided by [hedgehog](https://github.com/1pkg/hedgehog):

- _static_ that always waits for static specified delay before starting hedged request(s).
- _average_ that dynamically adjusts wait delay based on received successful responses average delays before starting hedged request(s).
- _percentiles_ that dynamically adjusts wait delay based on received successful responses delays percentiles before starting hedged request(s).

Main idea behind those dynamically adjusted resources - to better accommodate balance between reducing tail latency and growing related server load, quoting the article on this:

```

One such approach is to defer sending a secondary request until the first request
has been outstanding for more than the 95th-percentile expected latency for this class of requests.
This approach limits the additional load to approximately 5% while substantially shortening the latency tail.

```

Instead of a conclusion, I strongly suggest to read full [Google's paper](https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext) as it also shows other e.g. "cross-request long-term adaptations" techniques not covered here. Fun fact: this paper from Google is dated "February 2013". It's remarkable that Google cared about tail latency for many years, unfortunately not every company is Google, and even now in 2021 not so many other companies care about such important metric in their products.
