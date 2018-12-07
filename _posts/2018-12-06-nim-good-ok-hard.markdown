---
layout: post
title:  "Nim: The good, the OK, and the hard"
date:   2018-12-06 23:34:30 +0200
categories: nim
---

## Background

I'm a software engineer at [ThreeFoldTech](https://threefold.io) and the author of [Nim Days](https://xmonader.github.io/nimdays)

One of the projects we develop at ThreeFoldTech is [Zero-OS](https://github.com/threefoldtech/0-core) a stateless Linux operating system designed for clustered deployments to host virtual machines and containerized applications.
We wanted to have a CLI (like docker) to manage the containers and communicate with zero-os instead of using Python client.


## Application requirements
- single binary
- zos should be like docker for dockerd
- commands to interact with zero-os (via redis)
- subcommands to interact with containers on zero-os
- documentation (soft documentation, hard documentation)
- tabular output for humans (listing containers and such) 
- support json output when needed too (for further manipulation by tools like jq)

Sounds simple enough. Any language would do just fine


## Choosing Nim

From [Nim](https://nim-lang.org) website

> Nim is a systems and applications programming language. Statically typed and compiled, it provides unparalleled performance in an elegant package.
- High-performance garbage-collected language
- Compiles to C, C++ or JavaScript
- Produces dependency-free binaries
- Runs on Windows, macOS, Linux, and more



In the upcoming sections, I'll talk about the good, the okay, and the hard points I faced while developing this simple CLI application with the requirements above.


## The good

### Static typing
Nim eliminates a whole class of errors by being statically typed

### Expressiveness 
Nim is like python (whitespace sensitive language) and there's even a guide on the official repo [Nim for Python programmers](https://github.com/nim-lang/Nim/wiki/Nim-for-Python-Programmers). Seeing some of Pascal concepts in Nim gets me very nostalgic too.

{% highlight nim %}
import strutils, strformat, os, ospaths, osproc, tables, parsecfg, json, marshal, logging
import net, asyncdispatch, asyncnet, streams, threadpool, uri
import logging
import algorithm
import base64

import redisclient, redisparser
import asciitables
import docopt
proc checkContainerExists*(this:App, containerid:int): bool=
  ## checks if container `containerid` exists or not
  try:
    discard this.containerInfo(containerid)
    result = true
  except:
    result = false
{% endhighlight %}


I find UFCS (Uniform Function Call Syntax) really great too [excellent nim basics](https://narimiran.github.io/nim-basics/)

{% highlight nim %}
proc plus(x, y: int): int =  # <1>
  return x + y

proc multi(x, y: int): int =
  return x * y

let
  a = 2
  b = 3
  c = 4

echo a.plus(b) == plus(a, b)
echo c.multi(a) == multi(c, a)


echo a.plus(b).multi(c)  # <2>
echo c.multi(b).plus(a)  # <3>
{% endhighlight %}

Also case insensitivity `toUpper` `toupper` `to_upper` is pretty neat
> I don't use the same identifier with different cases in the same scope 


{% highlight nim %}
type ContainerInfo* = object of RootObj
  id*: string
  cpu*: float
  root*: string
  hostname*: string
  name*: string
  storage*: string
  pid*: int
  ports*: string
{% endhighlight %}

I like the way of defining types, enums and access control `*` means public.


### Developing sync, async in the same interface

> Pragmas are Nim's method to give the compiler additional information/commands without introducing a massive number of new keywords. Pragmas are processed on the fly during semantic checking. Pragmas are enclosed in the special {. and .} curly brackets. Pragmas are also often used as a first implementation to play with a language feature before a nicer syntax to access the feature becomes available.

I'm a fan of `multisync` pragma because it allows you to define procs for async, sync code easily

{% highlight nim %}


proc readMany(this:Redis|AsyncRedis, count:int=1): Future[string] {.multisync.} =
  if count == 0:
    return ""
  let data = await this.receiveManaged(count)
  return data

{% endhighlight %}
Basically in sync execution multisync will remove Future, and await from the code definition and will leave them in case of async execution

### The tooling

#### vscode-nim
[vscode-nim](https://github.com/pragmagic/vscode-nim) is my daily driver, works as expected, but sometimes it consumes so much memory. there's also [LSP](https://github.com/PMunch/nimlsp) in the works 

#### nimble
Everything you expect from the package manager, creating projects, custom tasks, managing dependencies and publishing (too coupled with github, but that's fine with me)


#### Generating documentation
[nim doc](https://nim-lang.org/docs/docgen.html) is the default tool in Nim to generate indexed and searchable documentation for the project

Here's a nimble task to generate documentation
{% highlight nim%}
task genDocs, "Create code documentation for zos":
    exec "nim doc --project src/zos.nim "
{% endhighlight %}


{% highlight bash %}
nim doc src/zos.nim 
Hint: used config file '/home/xmonader/.choosenim/toolchains/nim-0.19.0/config/nim.cfg' [Conf]
Hint: used config file '/home/xmonader/.choosenim/toolchains/nim-0.19.0/config/nimdoc.cfg' [Conf]
Hint: system [Processing]
Hint: zos [Processing]
Hint: strutils [Processing]
Hint: parseutils [Processing]
Hint: math [Processing]
Hint: bitops [Processing]
Hint: algorithm [Processing]
Hint: unicode [Processing]
Hint: strformat [Processing]
Hint: macros [Processing]
Hint: os [Processing]
Hint: times [Processing]
Hint: options [Processing]
Hint: typetraits [Processing]
Hint: posix [Processing]
Hint: ospaths [Processing]
Hint: osproc [Processing]
Hint: strtabs [Processing]
Hint: hashes [Processing]
Hint: streams [Processing]
Hint: cpuinfo [Processing]
Hint: linux [Processing]
Hint: tables [Processing]
Hint: parsecfg [Processing]
Hint: lexbase [Processing]
Hint: json [Processing]
Hint: parsejson [Processing]
Hint: marshal [Processing]
Hint: typeinfo [Processing]
Hint: intsets [Processing]
Hint: logging [Processing]
Hint: net [Processing]
Hint: nativesockets [Processing]
Hint: winlean [Processing]
Hint: dynlib [Processing]
Hint: sets [Processing]
Hint: openssl [Processing]
Hint: asyncdispatch [Processing]
Hint: heapqueue [Processing]
Hint: lists [Processing]
Hint: asyncstreams [Processing]
Hint: asyncfutures [Processing]
Hint: deques [Processing]
Hint: cstrutils [Processing]
Hint: asyncnet [Processing]
Hint: threadpool [Processing]
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(13, 10) Error: Threadpool requires --threads:on option.
Hint: cpuload [Processing]
Hint: locks [Processing]
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(67, 26) Error: undeclared identifier: 'fence'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(67, 31) Error: attempting to call undeclared routine: 'fence'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(67, 31) Error: attempting to call undeclared routine: 'fence'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(67, 31) Error: expression 'fence' cannot be called
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(78, 8) Error: attempting to call undeclared routine: 'fence'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(78, 8) Error: attempting to call undeclared routine: 'fence'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(78, 8) Error: expression 'fence' cannot be called
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(81, 10) Error: attempting to call undeclared routine: 'fence'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(81, 10) Error: attempting to call undeclared routine: 'fence'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(81, 10) Error: expression 'fence' cannot be called
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(83, 10) Error: attempting to call undeclared routine: 'fence'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(83, 10) Error: attempting to call undeclared routine: 'fence'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(83, 10) Error: expression 'fence' cannot be called
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(360, 37) Error: undeclared identifier: 'Thread'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(360, 43) Error: no generic parameters allowed for Thread
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(363, 54) Error: no generic parameters allowed for Thread
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(391, 3) Error: undeclared identifier: 'createThread'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(391, 15) Error: attempting to call undeclared routine: 'createThread'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(391, 15) Error: attempting to call undeclared routine: 'createThread'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(391, 15) Error: expression 'createThread' cannot be called
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(404, 15) Error: attempting to call undeclared routine: 'createThread'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(404, 15) Error: attempting to call undeclared routine: 'createThread'
../../../.choosenim/toolchains/nim-0.19.0/lib/pure/concurrency/threadpool.nim(404, 15) Error: expression 'createThread' cannot be called
Hint: uri [Processing]
Hint: base64 [Processing]
Hint: redisclient [Processing]
Hint: redisparser [Processing]
Hint: sequtils [Processing]
Hint: docopt [Processing]
Hint: nre [Processing]
Hint: pcre [Processing]
Hint: util [Processing]
Hint: util [Processing]
Hint: logger [Processing]
Hint: settings [Processing]
Hint: apphelp [Processing]
Hint: errorcodes [Processing]
Hint: sshexec [Processing]
Hint: hostnamegenerator [Processing]
Hint: random [Processing]
Hint: app [Processing]
Hint: asciitables [Processing]
Hint: zosclient [Processing]
Hint: uuids [Processing]
Hint: isaac [Processing]
Hint: urandom [Processing]
Hint: vbox [Processing]

{% endhighlight %} 
No idea why `generating docs` gives these errors (most likely because I'm using threadpool in my code?) so I went with my gut feeling and `--threads:on` 
{% highlight nim%}
task genDocs, "Create code documentation for zos":
    exec "nim doc  --threads:on --project src/zos.nim "
{% endhighlight %}
and now it works just fine, and earned its place in the Good parts.


## the OK
These are the OK parts that can be improved in my opinion

### Documentation
There's a great community effort to provide [documentation](https://nim-lang.org/documentation.html). I hope we get more and more soft documentation and better quality on the official docs too.


### Weird symbols / json
Nim chooses unreadable symbols `%*` and `$$` over clear names like dumps or loads :(

### Error Messages
Sometimes the error messages aren't good enough. For instance, I got [i is not accessible](https://github.com/nim-lang/Nim/blob/27b081d1f77604ee47c886e69dbc52f53ea3741f/lib/system/chcks.nim#L21) and even with using `writeStackTrace` I couldn't get anything useful. So I grepped the codebase where `accessible` comes from and continued from there.

Another example was this 
{% highlight bash %}
timeddoutable.nim(44, 16) template/generic instantiation from here
timeddoutable.nim(34, 6) Error: type mismatch: got <Thread[ptr Channel[system.bool]], proc (cancelChan: ptr Channel[system.bool]):bool{.gcsafe, locks: 0.}, ptr Channel[system.bool]>
but expected one of:
proc createThread[TArg](t: var Thread[TArg];
                       tp: proc (arg: TArg) {.thread, nimcall.}; param: TArg)
  first type mismatch at position: 2
  required type: proc (arg: TArg){.gcsafe.}
  but expression 'p' is of type: proc (cancelChan: ptr Channel[system.bool]): bool{.gcsafe, locks: 0.}
proc createThread(t: var Thread[void]; tp: proc () {.thread, nimcall.})
  first type mismatch at position: 1
  required type: var Thread[system.void]
  but expression 't' is of type: Thread[ptr Channel[system.bool]]

expression: createThread(t, p, addr(cancelChan))
{% endhighlight %}
While the error is clear I just had a hard time reading it

## The Hard 

I really considered switching to language with a more mature ecosystem for these points (multiple times) 

### Static linking
Nim promises `Produces dependency-free binaries` as stated on its website, but getting a static linked binary is hard, and undocumented process while it was one of the cases I hoped to use Nim for. 

I managed to statically link with PCRE and SSL [with lots of help from the community](https://github.com/kaushalmodi/hello_musl). 

### Dynamic linking
Building on Mac OSX with SSL is no fun, specially when your SSL isn't 1.1 [I managed to do with lots of help from the community] 

{% highlight bash %}
brew install openssl@1.1

nim c -d:ssl  --dynlibOverride:ssl --dynlibOverride:crypto --threads:on --passC:'-I/usr/local/opt/openssl\@1.1/include/' --passL:'-lssl -lcrypto -lpcre' --passL:'-L/usr/local/opt/openssl\@1.1/lib/' src/zos.nim
{% endhighlight %}


### Developing a redisclient
We have a redis protocol keyvalue store [0-db](https://github.com/threefoldtech/0-db) that I needed to work against a while ago, and I found a major [problem](https://github.com/nim-lang/redis/issues/11) with the implementation of the parser and the client in the official nim redis library. So I had to roll my own [parser](https://github.com/xmonader/nim-redisparser)/[client](https://github.com/xmonader/nim-redisclient) 

### Developing asciitable library 
To show a table listing all of the containers (id, name, open ports and image it's running from) I needed an ascii table library in Nim (I found 0 libraries). I had to write my own [nim-asciitables](https://github.com/xmonader/nim-asciitables)


### Nim-JWT 
In the transport layer, we send a JWT token to request extra privileges on zero-os and for that, I needed jwt support. Again, jwt libraries are far from complete in Nim and had to try to fix it [ES384 support](https://github.com/yglukhov/nim-jwt/pull/1) with that fix I was able to get the claims, but I couldn't really verify it with the public key :( So I decided not to do client side validation and leave the validation to zero-os (the backend)

### Concurrency and communication
In some parts of the application we want to add the ability to timeout after some period of time, and
Nim supports multithreading using `threadpool` and async/await combo and has [HTTPBeast](https://github.com/dom96/httpbeast), So that shouldn't be a problem.

When I saw `Channels` and `spawn` I thought it'd be as easy as goroutines in Go or fibers in Crystal
 
So that was my first try with `spawn`

{% highlight nim %}
import os, threadpool

var cancelChan: Channel[bool]

cancelChan.open()

proc p1():bool=
    result = true
    for i in countup(0,50):
        echo "p1 Doing action"
        sleep(1000)
        let (hasData, msg) = cancelChan.tryRecv()
        if msg == true:
            echo "Cancelling p1"
            return 
    echo "Done p1..."

proc p2(): bool =
    result = true
    for i in countup(0,5):
        echo "p2 Doing action"
        sleep(1000)
        let (hasData, msg) = cancelChan.tryRecv()
        if msg == true:
            echo "Cancelling p1"
            return
    echo "Done p2"


proc timeoutable(p:proc, timeout=10)= 
    var t = (spawn p())
    for i in countup(0, timeout):
        if t.isReady():
            return
        sleep(1000)

    cancelChan.send(true)

when isMainModule:
    timeoutable(p1)
    timeoutable(p2)
{% endhighlight %}


However, The Nim creator `Andreas Rumpf` said using Spawn/Channels is a bad idea and channels are meant to be used with Threads, So I tried to move it to threads

{% highlight nim %}

import os, threadpool

type Args = tuple[cancelChan:ptr Channel[bool], respChan: ptr Channel[bool]]

proc p1(a: Args): void {.thread.}=
    var cancelChan = a.cancelChan[]
    var respChan = a.respChan[]
    for i in countup(0,50):
        let (hasData, msg) = cancelChan.tryRecv()
        echo "p1 HASDATA: " & $hasData
        echo "p1 MSG: " & $msg
        if hasData == true:
            echo "Cancelling p1"
            respChan.send(false)
            return 
        echo "p1 Doing action"
        sleep(1000)

    echo "Done p1..."
    respChan.send(true)

proc p2(a: Args): void {.thread.}=
    var cancelChan = a.cancelChan[]
    var respChan = a.respChan[]
    for i in countup(0,5):
        let (hasData, msg) = cancelChan.tryRecv()
        echo "p2 HASDATA: " & $hasData
        echo "p2 MSG: " & $msg
        if hasData:
            echo "proc cancelled successfully" 
            respChan.send(false)
            return 
        echo "p2 Doing action"
        sleep(1000)

    echo "Done p2..."
    respChan.send(true)


proc timeoutable(p:proc, timeout=10): bool= 

    var cancelChan: Channel[bool]
    var respChan: Channel[bool]
    var t:  Thread[Args]
    cancelChan.open()
    respChan.open()
    var args = (cancelChan.addr, respChan.addr) 
    createThread[Args](t, p, (args))

    for i in countup(0, timeout):
        let (hasData, msg) = respChan.tryRecv()
        if hasData:
            return msg 
        sleep(1000)

    echo "Cancelling proc.."
    cancelChan.send(true)
    close(cancelChan)
    close(respChan)

    return false

when isMainModule:
    echo "P1: " & $timeoutable(p1)
    echo "P2: " & $timeoutable(p2)
{% endhighlight %}


I'm not a fan of this `passing pointers`, `casting`, `.addr`


### Macros
Macros allow you to apply transformations on AST on compile time which is really amazing, but It can be very challenging to follow or even work with specially if it's not well documented and I feel they're kinda abused in the language resulting in half-baked libraries and macros playground.


## Conclusion 
Overall, Nim is a language with a great potential, and its small team is doing an excellent job. Just be prepared to write lots of missing libraries if you want to use it in production. It's a great chance to reinvent the wheel with no one blaming you :) 