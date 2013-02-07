# Exploring Node.js, the event model inside and upside

## Introduction

In this literate CoffeeScript code file, I will further explore the Node.js
event model. In particular, we're interested in finding out how we can deal with
events in a generic, but in a more user-centric, rich fashion. As we'll find
out, parts of Node.js have - including the event model - severe limitations
imposed, drawbacks that come out of the design paradigm, or just plain old
design flaws in the range of "not thought out well enough". Luckily the node
activity is extremely active in producing ready-to-use snap-in packages that
cover any kind of problem from multiple ranges. Although I feel it is good to be
diverse in the landscape, I also feel there usually is a 'best approach',
especially for the tasks specific, and as such we'll often just conform to the
'industry (de-facto) standards and practices'.

## Event-driven design

From the Node.js 0.8.x reference manual we learn the following principle:

> Many objects in Node emit events: a `net.Server` emits an event each time a peer
connects to it, a `fs.readStream` emits an event when the file is opened. All
objects which emit events are instances of `events.EventEmitter`. You can access
this module by doing: require("events");*
<br><small>**API stability: 4 (frozen)</small>

Taking the evented I/O (input/output) approach, this design allows for us
to cheaply put any function in a condition known as "event loop". To better
understand the reasoning behind this architectural choice, first please consider
the following:

> Functions can then be attached to objects, to be executed when an event is
emitted. These functions are called listeners.

and

> Node.js uses an event-driven, non-blocking I/O model. The first basic
thesis of node.js is that I/O is expensive.

Take a look at the following to get a basic idea of how expensive I/O really is.
!["The cost of I/O"](http://blog.mixu.net/files/2011/01/io-cost.png)

From [Mixu][02] we learn the following:

So the largest waste with current programming technologies comes from waiting
for I/O to complete. There are several ways in which one can deal with the
performance impact (from [Sam Rushing][01]):

* **synchronous**: handle one request at a time, each in turn: this is most
simple. The drawback is that any one request can hold up (block) all the other
requests.

* **fork a new process**: every request we spawn a new subprocess for it to
handle, this does not scale well. Hundreds of connections means hundreds of
processes.

* **threads**: start a new thread to handle each request. It's easy and more
genttle to the kernel compared to using forks, since threads usually have much
less overhead. The downside is that the machine CPU may not support threads, and
threaded programming can get very complicated very fast, with worries about
controlling access to shared resources.

Any of these concepts by itself constitute a library full of documentation and
thoughts, experiences and use-case scenatios so it's beyond the scope of this
article to further elaborate on these subjects. It's ok if you don't know what
they mean or only vaguely grasp the concept.

The second basis thesis is that:

```md
the thread-per-connection approach is memory-expensive
```

Again from [Mixu][02]:

```md
Apache is multithreaded: it spawns a thread per request (or process, it depends
on the conf). You can see how that overhead eats up memory as the number of
concurrent connections increases and more threads are needed to serve multiple
simulataneous clients. Nginx and Node.js are not multithreaded, because threads
and processes carry a heavy memory cost. They are single-threaded, but event-
based. This eliminates the overhead created by thousands of threads/processes by
handling many connections in a single thread.
```

## Not everything is as it seems

Node.js keeps a single thread for your code. It really is a single thread
running: you can’t do any parallel code execution; doing a “sleep” using `while`
and `1000` for example will block the server for one second:

```js
while(new Date().getTime() < now + 1000) {
   // do nothing
}
```

So while that code is running, node.js will not respond to any other requests
from clients, since it only has one thread for executing your code. Or if you
would have some CPU -intensive code, say, for resizing images, that would still
block all other requests.

…however, everything runs in parallel *except* your code

There is no way of making code run in parallel within a single request. However,
all I/O is evented and asynchronous, so the following won’t block the server:

```js
(function() { var time; time = process.hrtime();

  setTimeout(function() {
var diff;
    return diff = process.hrtime(time);
  });

}).call(this);
```

Having asynchronous execution is a good thing because it simplifies writing code
compared to (multi)threads, where [concurrency][w1] issues have a tendency to
result in WTFs) but with CoffeeScript we can improve on that situation! The JS
that you normally would write contains tons brackets and function keywords, it
can get a little a distracting and I already have enough of that myself as it
is. So we're fortunate that the syntactic sugar CoffeeScript provides, is
extremely well suited for asynchronous programming tasks, as you can see by the
rewritten code below:

```coffee-script

    time = process.hrtime()
    console.log time
    setTimeout () ->

      diff = process.hrtime time
```

In node.js, you aren’t supposed to worry about what happens in the backend: just
use callbacks when you are doing I/O; and you are guaranteed that your code is
never interrupted and that doing I/O will not block other requests without
having to incur the costs of thread/process per request (e.g. memory overhead in
Apache).

Broadly speaking, there are two ways to handle concurrent requests to a server.
Threaded servers use multiple concurrently-executing threads that each handle
one client request, while evented servers run a single event loop that handles
events for all connected clients. To chose between the threaded and evented
approaches you need to consider the [load profile of the server][03].

## The Node Restaurant: Great Staffing, Low Prices

I'm taking a little side-step here to further elaborate on these concepts. It's
very likely that, as progressing through this material, you become increasingly
confused. This is rather likely as it has happend to the most of us when
thinking and trying to understand these abstract concepts, I found the following
analogy a great way of representing these concepts in a fashion that we
humans/laymen can comprehend:

```md
Web application should be as a restaurant. You have waiters (web server) and
cooks (**workers**). Waiters are in contact with clients and *do simple tasks*
like providing menu or explaining if some dish is vegetarian. On the other hand
they delegate harder tasks, such as getting all the ingredients and preparing
the meal, to the kitchen. Because waiters are doing only simple things they
respond quick, and cooks can concentrate (allocate resources) on their intensive
job.

Node.js here would be a *single but very talented waiter* that can process **many
requests at a time**, and Apache would be a gang of dumb waiters that just process
one request each.

If this one Node.js waiter would begin to cook, it would be an immediate
catastrophe. Still, cooking could also exhaust even a large supply of Apache
waiters, not mentioning the chaos in the kitchen and the progressive decrease of
responsitivity.
<br><small>Thanks to [mbq][04]</small>
```

### Parties, side-dishes and take-away

I could add to that the you also need to consider the fact that, without help of
an external party, you have no clue which order will come back from the kitchen
first. Obviously, the easier to prepare dishes (computation, allocation of
resources is few) are done first. Heck, some might even take no preperation at
all, like the bread or salad, because we serve it so often it's always ready
(caching, pre-compiled code, optimized binaries).

Normally, this wouldn't be much of a problem: some people just get server before
others eventhough they came in later simply because their dish is prepared
faster. Our super-waiter can't wait for every dish to return and once the
kitchen is done, has to serve the meal or it will go cold. But when we deal with
parties, this becomes a problem. People expect they can eat together at the same
time. Some might actually have arrangements with other party-members to share or
trade (parts) of their dishes. Also there is etiquette: it can be insulting to
people to serve children before grandparents; any good host should consider this
a mortal sin! A node.js way of solving this would be summarized [here][06]. In
another, future, article I'll try and elaborate on this concept further.

Oh. Our restaurant also has cases where it just gives the guest a recipe (code
logic) and the ingredients (data, REST) to have them prepare the meal at home
([client-side rendering][05]) so it doesn't cost our own cooks and waiters any
effort at all. Thats also a whole different cookie and food left for a later
time.

## Conlusion

To conclude this introduction, the following should be kept in mind:

```md
Having asynchronous I/O is good, because I/O is more expensive than most code
and we should be doing something better than just waiting for I/O.
```

and

```md
All objects which emit events are instances of events.EventEmitter.
Functions can then be attached to objects, to be executed when an event is emitted. These functions are called listeners.
Listeners can be added to the end of the listeners array for the specified event. They then fire when the event takes place.
```

This idea of "events" are now nothing more than the subjects we spoke about in
this article. Listeners are also a well known pattern, mostly made popular by
jQuery and other DOM libraries. A Listener might look like this:

```js
finder.on('done', function (event, records) {
  ..do something
});
```

We call a function on an object that adds a listener. In that function we pass
the name of the event we want to listen to and a callback function. ‘on’ is one
of many common name for this function, other common names you will come across
are ‘bind’, ‘listen’, ‘addEventListener’, ‘observe’. You should make yourself
familiar with the ideas expressed here. [This article][07] is a nice starting
point for some of these and other related programming practices.


[01]: <http://www.nightmare.com/medusa/async_sockets.html>
[02]: <http://blog.mixu.net/2011/02/01/understanding-the-node-js-event-loop/>
[03]: <http://mmcgrana.github.com/2010/07/threaded-vs-evented-servers.html>
[04]: <http://stackoverflow.com/a/3491931/2004521>
[05]: <http://engineering.linkedin.com/frontend/client-side-templating-throwdown-mustache-handlebars-dustjs-and-more>
[06]: <https://github.com/jprichardson/node-nextflow>
[07]: <http://sporto.github.com/blog/2012/12/09/callbacks-listeners-promises/>
[w1]: <http://en.wikipedia.org/wiki/Concurrent_computing>
