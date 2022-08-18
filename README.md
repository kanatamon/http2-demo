# HTTP/2 Demo

Welcome to my HTTP/2 Demo where this is a simple experiment as part of my hight-level
experiment on the reproduction a failure while using server-sent event over
multi-hops network.

## How to run locally

### System Requirements

- [Caddy](https://caddyserver.com/docs/install)

### Getting started

To start Caddy as a daemon, run:

```bash
caddy run
```

After the starting Caddy:

- Visit `http://dev.localhost` to inspecting the network over HTTP/1.1
- Visit `https://dev.localhost` to inspecting the network over HTTP/2

## Intro

Recently I did research on the real-time mechanism topic and then reached to the
[server-sent events or sse](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
with [the issue on older networks](https://dev.to/miketalbot/server-sent-events-are-still-not-production-ready-after-a-decade-a-lesson-for-me-a-warning-for-you-2gie).
In the blogpost where the issue addressing, they said about [_old proxies_](https://dev.to/dunglas/comment/103cg)
which might causing the sse to not work as expect. I think it gonna fun to reproduce
the issue on a local machine and see if it's possible.

But before that reproduction, I also realized, once I did heard about [_sse should be with HTTP/2_](https://www.youtube.com/watch?v=5vyKhm2NTfw&list=WL&index=8&ab_channel=Front-EndEngineer).
To be honest, My familiarity to HTTP/2 is very shallow. So, this should be a good
opportunity to do some experiment to create a strong mental model about HTTP under
the wires. Let's drive into it!.

## Story

At first, I would create a scope about this HTTP/2's experiment. And seem that
some comparison between HTTP/1.x to HTTP/2 should be a good starter point.
So, I need to design a workflow and think about a measurement for the experiment.

> DISCLAIMER: I would NOT drive into details like what is multiplexing,
> HTTP pipelining, etc. It would take too long and too specific, which should be
> investigated by a topic instead of an overview like what I do here. [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTPA)
> has very great articles about each specific topic over HTTP, if you would like to
> investigate more deeper.

In this experiment, I reached for [this mention on MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Connection_management_in_HTTP_1.x#domain_sharding:~:text=sending%20parallel%20requests.%20Default%20was%20once%202%20to%203%20connections%2C%20but%20this%20has%20now%20increased%20to%20a%20more%20common%20use%20of%206%20parallel%20connections.)
, which said about the limitation for number of parallel requests made by a browser
to a domain on the connection management in HTTP/1.x. And the limitation could obsolete
when migrating to HTTP/2, which has the Multiplexing's feature. The number of
parallel requests seem to be a point where I looking for making the scope.

I came up with a idea about _just simple transferring static files over the wire_
should be enough to inspecting the performance of HTTP/2. The next thing to do is
to list some general requirement about the experiment:

1. HTTP/1.x and HTTP/2 on my machine
2. A static website and a bunch of resource's request to simulate multiple TCP connections
3. A measurement, and I think just the networks'tab on Google Chrome's Devtools should be enough

And then as deep as I investigating, I found that HTTP/2 need to be on TLS which
is HTTPS. As the general requirement, I want run both HTTP/1.x and HTTP/2 on my
machine. I need a solution to enable HTTPS on my local machine. This should be a
subtask to investigate. Then for a little while I found [Caddy](https://caddyserver.com/docs/)
, a web server that come with automatic HTTPS and HTTP/2 by default with very less
configuration and also do serving static file too. Thanks to [Skilltrive](https://www.youtube.com/watch?v=327shuxREVQ&ab_channel=Skillthrive)
and lovely web community on our planet. This tool helps me refine the general
requirement into the specific requirement.

When come to a static website I need for the experiment, I found [a demon by Hussein Nasser](https://www.youtube.com/watch?v=fVKPrDrEwTI&ab_channel=HusseinNasser).
He demonstrated by letting a browser do multiple request that kick-off at the most
same time. So If I do what the demo did that should be enough, right?. And the
next thing I considering is what kind of static file to be requested by the website.
At first, I think it doesn't matter what type of static file to be served, the
matter should be just only the size. But let's think about it, I would do some
visualization on the screen to reflect what state of each request is performing.
And each request that download a resource should be independent to each other,
then my visualization of each request gonna be independent to each other too.
A type of static file that I need should be kind of the non-blocking render
resource where it means a moment during downloading that resource would let a browser
continuos rendering instead of freezing the screen until all downloads are done.
So, I would go for [the lazy-loading's strategy](https://developer.mozilla.org/en-US/docs/Web/Performance/Lazy_loading)
that let whatever static file requesting with this strategy gonna be non-blocking
by it own nature. And I would go for just _style.css_ with some long comment without
minifying to simulate variant of file size.

At this point, there is no need to add some visualization on the screen. The
requests gonna be fired on background and also be inspected by a devtools.
Absolutely, No-room for any visualization. But just for fun, I would add some
simple visualization represent which request is responded successfully or it's
pending.

So, the specific requirement for the experiment is going to be the following:

1. Setup Caddy server to serve both HTTP/1.x and HTTP/2 on my local machine
2. A static website that request a number of _style.css_
3. Render boxes each represent a request's file-size and also state whether it's responded successfully or pending

## Inspecting the result

TODO: add results

## Visual improvement using `Fetch` and other alternatives during my experiment

> The purpose of using `Fetch` is just to make HTTP request.

There are other APIs that can do what `Fetch` do with especially less code comparing
to `Fetch`. Before I came up with `Fetch`, I did use the lazy-loading using `<link />`
and then the XMLHttpRequest's API. The following is the story for why

> Nothing relating to HTTP/2 and can be considered as an out-of-context for this
> experiment. Just some visual improvement.

### Lazy-loading

Actually, we can manually type `<link />` into _index.html_ as much as we want,
right?. But that's quite pain fo us to do the duplication. So, I end-up to let
javascript populating `<link />` to what number I want. And also add handler to
trigger when the resource was successfully loaded.

```js
const link = document.createElement('link');
link.href = 'style.css';
link.rel = 'stylesheet';
link.type = 'text/css';
link.onload = () => {
  // Render whatever indicating the resource was successfully loaded
  box.style.backgroundColor = 'lime';
};

// This kick-off the request
document.querySelector('head').append(link);
```

Just using this API, every think look fine when I'am inspecting on Google Chrome's
DevTools. But would it be better to have a nicer visualizing while a particular
request is performing like, which state it is? _request is queued_,
_connection is created_, etc.

This should extend the specific requirements:

- Each particular request should be visually indicating to the state it is performing
  1. When a request is queued, let leave it
  2. When a request is connected over TPC, let displaying outline indicating it's performing download the resource
  3. When a request is downloading the resource, also display some progress
  4. When a request is closed, let remove the outline

To get progress update during downloading the resource. It's impossible to do with
just bare `<link />`, after some research I found the MDN mentioned about
[ProgressEvent](https://developer.mozilla.org/en-US/docs/Web/API/ProgressEvent).
That look like what I want, and as the examples demo to use XMLHttpRequest's
API.

### XMLHttpRequest

With the power of this APIs bringing mine to reach as the specific requirement
needed. This is how I using the APIs:

```js
const xhr = new XMLHttpRequest();
xhr.open('GET', 'style.css');
xhr.onprogress = (pe) => {
  if (!pe.lengthComputable) return;
  if (pe.loaded > 0) {
    box.style.outline = '1px solid deeppink';
  }
  const percentage = Math.floor(100 * (pe.loaded / pe.total));
  for (const unitAsIdx in Array.from({ length: percentage })) {
    const unitElement = box.childNodes.item(unitAsIdx);
    unitElement.style.backgroundColor = '#dedede';
  }
};
xhr.onloadend = () => {
  box.textContent = '';
  box.style.backgroundColor = 'lime';
  box.style.outline = '0';
};
xhr.send();
```

The XMLHttpRequest's APIs is used for more than a decade. But this is seem to be
a history of the Web. Nowadays, we tended to use `Fetch`s APIs as the replacement
of the XMLHttpRequest's APIs. So, It gonna be better to go with the modern web
standard.

To replace implementation of XMLHttpRequest, there is no mention about the
ProgressEvent and `Fetch` along together on the MDN. And I found a [demo](https://javascript.info/fetch-progress)
so, I decided to use it as the replacement.

But the implementation using `Fetch` is very low-level, that make code a little
bit ugly and not clean as XMLHttpRequest did. So, I decided to abstract that away
by introducing this function:

```js
async function loadContentProgressively(response, stepCallback) {
  const reader = response.body.getReader();
  const contentTotal = +response.headers.get('Content-Length');
  let done = false;
  let contentLoaded = 0;
  while (!done) {
    const result = await reader.read();
    const stepLoaded = result?.value?.length ?? 0;
    contentLoaded += stepLoaded;
    stepCallback({ loaded: contentLoaded, total: contentTotal });
    done = result.done;
  }
}

// Then on `Fetch` would be just this

fetch(URL).then(async (response) => {
  ...
  await loadContentProgressively(response, renderProgressFunction);
  ...
});
```

## Bottom-line

TODO: what I just know during the experiment and how do I can use it?

## References

- [How to Set Up HTTPS and Subdomains on Localhost With Caddy Server](https://www.youtube.com/watch?v=327shuxREVQ&ab_channel=Skillthrive)

- [How HTTP/2 Works, Performance, Pros & Cons and More](https://www.youtube.com/watch?v=fVKPrDrEwTI&ab_channel=HusseinNasser)

- [That's so fetch](https://jakearchibald.com/2015/thats-so-fetch/)
