---
layout: post
title:  "Nim: The good, the OK, and the hard"
date:   2018-12-06 23:34:30 +0200
categories: nim
---

## Background

I'm a software engineer at [threefoldtech](https://threefold.io) and I'm the author of [Nim Days](https://xmonader.github.io/nimdays)

One of the projects we develop at threefoldtech is [Zero-OS](https://github.com/threefoldtech/0-core) a stateless Linux operating system designed for clustered deployments to host virtual machines and containerized applications. We wanted to have a CLI (like docker) to manage the containers and communicate with zero-os instead of using Python client.


## Application requirements
- single binary
- zos should be like docker for dockerd
- commands to interact with zero-os (via redis)
- subcommands to interact with containers on zero-os
- documentation (soft documentation, hard documentation)
- tabular output for humans (listing containers and such) 
- support json output when needed too (for further manipulation by tools like jq)

So it all seems simple enough any language would be just fine


## Choosing Nim

From [Nim](https://nim-lang.org) website

> Nim is a systems and applications programming language. Statically typed and compiled, it provides unparalleled performance in an elegant package.
- High-performance garbage-collected language
- Compiles to C, C++ or JavaScript
- Produces dependency-free binaries
- Runs on Windows, macOS, Linux, and more

This sounds promising so we gave it a try


## The good

### Nim is expressive and statically typed
Nim is like python but eliminates a whole class of errors by being statically typed (while maintaing expressiveness)
 (whitespace sensitive language) and there's even a guide on the official repo [Nim for Python programmers](https://github.com/nim-lang/Nim/wiki/Nim-for-Python-Programmers). Seeing some of pascal concepts in Nim gets me very nostalgic too. 

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


I consider UFCS (Uniform Function Call Syntax) expressive too [excellent nim basics](https://narimiran.github.io/nim-basics/)

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

Also case insensitivity toUpper toupper to_upper is pretty neat
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

I like they way of defining types, enums and access control `*` means public.

## Developing sync, async in the same interface

> Pragmas are Nim's method to give the compiler additional information / commands without introducing a massive number of new keywords. Pragmas are processed on the fly during semantic checking. Pragmas are enclosed in the special {. and .} curly brackets. Pragmas are also often used as a first implementation to play with a language feature before a nicer syntax to access the feature becomes available.

I'm a fan on `multisync` pragma because it allows you to define procs for async, sync code easily

{% highlight nim %}


proc readMany(this:Redis|AsyncRedis, count:int=1): Future[string] {.multisync.} =
  if count == 0:
    return ""
  let data = await this.receiveManaged(count)
  return data

{% endhighlight %}
Basically in sync execution multisync with remove Future, and await from the code definition and will leave them in case of async execution

## the OK
These are the OK parts that can be better

### Documentation
There's a great community effort to provide [documentation](https://nim-lang.org/documentation.html) I hope we get more and more soft documentation and better quality on the official docs too.

### The tooling
I've been only relying on [vscode-nim](https://github.com/pragmagic/vscode-nim) only it works fine as expected but sometimes it consumes so much memory on my 8GB machine and

### weird symbols / json
Nim uses unreadable symbols `%*` and `$$` as procs in json i don't mind typing dump or load even [serializing tables](https://github.com/nim-lang/Nim/pull/7203) needs a bit of work 

### Error messages
Sometimes the error messages aren't good enough for instance I got [i is not accessible](https://github.com/nim-lang/Nim/blob/27b081d1f77604ee47c886e69dbc52f53ea3741f/lib/system/chcks.nim#L21) and even with using `writeStackTrace` I couldn't get anything useful so I grepped the codebase where `accessible` comes from.

another example was this 
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
While the error is clear I just had hard time reading it

## The Hard
These are the pain points I faced 

### Static linking
Nim promises `Produces dependency-free binaries` as stated on its website, but getting a static linked binary is hard, and undocumented process while it was one of the cases I hoped to use Nim for. 

I managed to statically build with PCRE and SSL [with lots of help from the community](https://github.com/kaushalmodi/hello_musl). 

### Dynamic linking
Building on Mac OSX with SSL is no fun, specially when your SSL isn't 1.1 [I managed to do with lots of help from the community] 

{% highlight bash %}
brew install openssl@1.1

nim c -d:ssl  --dynlibOverride:ssl --dynlibOverride:crypto --threads:on --passC:'-I/usr/local/opt/openssl\@1.1/include/' --passL:'-lssl -lcrypto -lpcre' --passL:'-L/usr/local/opt/openssl\@1.1/lib/' src/zos.nim
{% endhighlight %}


### Developing a redisclient
We have a redis protocol keyvalue store [0-db](https://github.com/threefoldtech/0-db) that I needed to work with a while a go and I found a major [problem](https://github.com/nim-lang/redis/issues/11) with the implementation of the parser and the client in the official nim redis library, So I had to roll my own [parser](https://github.com/xmonader/nim-redisparser)/[client](https://github.com/xmonader/nim-redisclient) 

### Developing asciitable library 
To show human readable listing of the containers (id, name, open ports and image it's running from) It made sense to search for ascii table library in Nim (I found 0 libraries). I had to write my own [nim-asciitables](https://github.com/xmonader/nim-asciitables)


### Nim-JWT 
In the transport layer, we can sent JWT token for extra privileges on zero-os and for that I needed jwt support. I found out that jwt libraries are far from complete in Nim and had to try to fix it [ES384 support](https://github.com/yglukhov/nim-jwt/pull/1) with that fix I was able to get the claims, but I couldn't really verify it with the public key :( So I decided not to do client side validation and leave the validation to zero-os

### Concurrency and communication
In some parts of the application we want to add the ability to timeout after some period of time, and
Nim supports multithreading using `threadpool` and async/await combo and has [HTTPBeast](https://github.com/dom96/httpbeast), So that shouldn't be a problem.

When I saw Channels and spawn i thought it'd be as easy as goroutines in Go or fibers in Crystal
 

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


However, Nim creator `Andreas Rumpf` said using Spawn/Channels is a bad idea and channels are meant to be used with Threads, So I tried to move it to threads

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


I'm not fan of this  `passing ptrs`, `casting`, `.addr`


### Macros
Macros allows you to apply transformations on AST on compile time which is really amazing, but It can be very challenging to follow or even work with specially if it's not well documented and I feel they're kinda abused in the language resulting in half-baked libraries and macros playground.


Overall it's language with a great potential and its small team are doing an excellent job, give them a hand if you're interested in Nim, It's a great chance to help a language to grow and no one can blame you for reinventing the wheel :) 