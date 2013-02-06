# Package definition construction

## Module helper functions

First, I'll add a quick and dirty short-hand function to point at console.log
and save myself a lot of keystrokes on code. The ... dots is called a `splat`
and basically means we can feed it a variable length list of arguments (optional
arguments).

```coffee-script

    log = (args...) -> console.log args...

```

Behind the scenes, we'll find the console.log function being mapped to the
standard output stream, in case of server-side JavaScript that is, and it looks
like this:

```coffee-script

    console.log = (d) ->

        process.stdout.write d + '\n'

```

As with any variable in our dynamically type language, we may just override
these with our own values.


## Basic low-level information

As a small demonstration of some things that we are able to utilize straight
away, I'll add a few variables to store system information. The process object
is a global object and can be accessed from anywhere. It is an instance of
EventEmitter.

```coffee-script

    log process.execPath

    log process.title + " (" + process.arch + ") " + process.platform

    log "Found environment..." if process.env?

    log "This process is pid " + process.pid

    log 'Current directory: ' + process.cwd()

```

Nearly all of node.js capabilities can be utilized by functions and key:value
objects returned from the core library classes and methods (API). Additionally
the memory usage method returns a data object that requires the `util` core
library for its `util.inspect` method and I wanted to avoid dependencies until
now - to illustrate the presence of globals.


## Signal processing logic

Signaling was one of the earlier, more primitive, ways in which computer
processes and users we able to communicate. Later, this mostly became an
abstraction based on (near) legacy ways of working with computers. In many cases
low-level system signals for a powerful interface to running Unix processes. The
responsiveness to these signals is near excellent, and sometimes even special
(hidden) features have been hooked to `USR1` and `USR2` signals of many well
known applications in Linux and Mac OS X. Because often the first means of
leaving a running program instance for any user, is the key combination `Ctrl+C`
as soon as the program is up. Thats why any logic dealing with these low-level
kills, controls and so on should come first.

Start reading from stdin so we don't exit the console.

```coffee-script

    process.stdin.resume()

```

We will place a listener here, on the process, and as such it will catch any
events emitted when a low-level signal is received by the process(es).

```coffee-script

    process.on "SIGINT", () -> log "Got SIGINT. Press Control-D to exit."

```

The listener for `SIGINT` (Control+C on most computers) is just one of the many
signals we can possibly receive. Another signal is that of (modem) hang-ups.

```coffee-script

    process.on "SIGHUP", () -> log "SIGHUP. hang-up signal emitted."

```

For now, I'll sum up any more I find but at the moment, have no use for.

```coffee-script

    process.on "SIGUSR1", () -> log "SIGUSR1. user-defined signal 1."

    process.on "SIGUSR2", () -> log "SIGUSR2. user-defined signal 2."

    process.on "SIGQUIT", () -> log "Got SIGQUIT. quit signal."

    process.on "SIGKILL", () -> log "Got SIGKILL. kill signal."

```

There are many more but they are less frequently used, the ones above are still
commonly found in day-to-day BSD, Linux and Mac system operations.


## Dependencies

As nearly every software package developed under the node.js stack, we require
a few libraries to be present, functional and working, and to be imported in
our new program.


### Core modules

We import the native node.js libraries usually first. We can safely assume these
will always be in perfect working condition 999/1000 times.

We require access to the lower file system I/O architecture to perform
operations such as writing and reading of files. Both synchronous as
asynchronous methods are provided in the standard library. The `path` library is
supposed to tell us what the base name of this present working directory
(`$PWD`) is. This is assumed the name of our program later on.

```coffee-script

    try

        fs = require 'fs'

        path = require 'path'

```

We created a `try...catch` code structure to allow for some basic exception
handling. In case of any error, the `catch` statement will intercept the error,
and allow for us to custom deal with the way it is presented to the end-user.
This is a better method of exception handling than just dumping the raw stack -
for most normal people this is way too cryptic.

```coffee-script

    catch err

        throw new Error "Failed to import core libraries, nothing I can do " +
            "here: you're on your own. Bailing out!"

```

Ok. If all is well, then let's continue to import some of our other stuff.


### External modules

Technically, these titles are not a 100% accurate. It's not *always* modules
that we are importing but for arguments sake, let's just consider the methods
and properties which are exposed (using `module.exports` statements), and which
belong to a certain class or group of entities rather than really "belong" to
the module, to at least be a integrated part of it. Usually we will be depending
upon the entire package anyway, by use of the `require` keyword we may access
the externally defined sweetness.

These packages however, are not something we can always just rely upon. The
built-in, internal node.js logic, dictates for a major part some of its
strengths and weaknesses but sadly, if a package is not found in the
`node_modules` tree of directories, node does nothing to offer us a download.
Instead we get the very unfriendly stack dumps.


#### Improved, colorful exception handling and error tracebacks

But we can do better than that! There is a really nice package written by
[DennisKehrig](https://github.com/DennisKehrig) which is known under the name
[ANSInception](https://github.com/DennisKehrig/ANSInception). This program offers
colorful exception handling, adds CoffeeScript support and improved nodemon /
supervisor compatitibility.

```coffee-script

    require('ansinception') ->

```

Now, everything that gets defined/called under this function, will be subject to
improved error handling. Isn't that just sweet? That outer `try` is there as
extra safety net, in case our end-user doesn't have these packages set-up yet.

Now let's keep (s)mashing-up our stack like this and see how we can continue this
line.


#### Reusable, structured user inquiries and dialog persistence

Prompter is a node package that is written to create json files, prompt for
input (over the console) and evaluates expressions which are defined in a .json
file. This is very nice as it allows us to think-out our common work patterns,
properly define the elements of the pattern/workflow/procedure. The literate
programming style that I exhibit here, further enables us to write out a more
elaborate, documented architecture; especially with dependencies it can be good
to be able and later recollect why certain - possibly fatal or essential for
survival - are made.

```coffee-script

        prompter = require 'prompter'
```

#### Adding the foundations of smart-interacting applications

There is simply no way to begin describing the vast amount of features that we
are able to unlock when tapping into the riches of linguistic computation and
phonetics. With the trainable classifiers, deductive reasoning, persistence and
machine learning capabilities, we are lucky to have the fabulously made
[NodeNatural](https://github.com/NaturalNode/natural) program to our disposal.

```coffee-script

        natural = require 'natural'

```

Also I downloaded a local copy of the `Wordnet` application slash dictionary.
This can be useful when we want to be lazy/smart and get automatic synonyms of
works as, for example, `package.json` field values for the `keywords` array.
That is, once the term is no longer open for interpretation, the reality of
[ambiguous](http://en.wikipedia.org/wiki/Ambiguity) terms is often one obstacle.

Fortunately, wordnet has a extensive database of different `senses` of words and
may allow for easier deduction of the sense through associated contextually
important neighbouring words. To further enhance our Artificial Intelligence
capabilities, I'll add-in a second tool, this one capable to break-up English
[Part of Speech or POS.js](https://github.com/fortnightlabs/pos-js) tagged
sentences.

```coffee-script

        pos = require 'pos'
```

Being able to tell *what the subjects of our talks are*, *when a
question is being asked* or being able to tell that *we are getting a direct
call for action (command)* will come in handy when dealing with (command line)
programs of semi-intelligent (code bot) nature.

Wordnet is a seperate package and it holds the data we can reference and query.

```coffee-script

        wordnet = require 'WNdb'
```

Now we can continue with the pieces that deal with our package definition. Here
we resolve the structured / uniform file path through a convention I just made:

```coffee-script

        pkgFile = __dirname + "/etc/npm.d/default.package.prompted.json"
```

Normally, we wouldn't dare to do synchronous opertions inside Node. This because
there is a performance penalty for mixing the two; but since we read only 1 file
and we know this for certain:

```coffee-script

        src = fs.readFileSync pkgFile, "utf8"
```

Furthermore, we can pass contextual data from inside here, into the dynamic
Question and Answer JSON script file.

```coffee-script

        ctx = basename: path.basename process.cwd()
```

Finally, we call the prompter in asynchronous fashion, passing in the custom
data that is obtained elsewhere.

```coffee-script

        s = prompter src, ctx, (err, output) ->

            log output

            process.stdin.pause()
```

Pipe the result of the prompter process to standard output, so we see the
package data written to our console.

```coffee-script

        s.pipe process.stdout
```

Also we tunnel any input through the standard input interface `stdio`, to our
prompter instance.

```coffee-script

        process.stdin.pipe s
```

When done, we can resume on standard input.

```coffee-script

        process.stdin.resume()
```
