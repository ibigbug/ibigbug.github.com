---
layout: post
title: Read Golang MemsStats the Hard Way
date: 2017-09-02
category: work
meta: Several approaches to read the Golang program runtime memory stats.
---

There is an interesting struct definition named `runtime.MemStats` which keeps the program runtime memory metris and stats. The definition somehow explains how Golang runtime manage runtime heap/stack/os memory usage. Normally we can use `runtime.ReadMemStats(m *MemStats)` to read it. In this post, plus the normal way, I'll introduce two other approaches to get a MemStats.

## Normal way

First, the normal way, super straighforward is to use the `runtime.ReadMemStats(m *MemStats)`:

{% highlight go %}
package main

import (
	"encoding/json"
	"fmt"
	"runtime"
)

func main() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	s, _ := json.Marshal(m)
	fmt.Printf("%s\n", s)
}

{% endhighlight %}

save it to `main.go` and run `go run main.go`, you'll see the output like(go1.7):

```
{
    "Alloc": 63776,
    "BuckHashSys": 2046,
    "BySize": [
        {
            "Frees": 0,
            "Mallocs": 0,
            "Size": 0
        },
        ...
        {
            "Frees": 0,
            "Mallocs": 4,
            "Size": 8
        },
    ],
    "DebugGC": false,
    "EnableGC": true,
    "Frees": 7,
    "GCCPUFraction": 0,
    "GCSys": 98304,
    "HeapAlloc": 63776,
    "HeapIdle": 499712,
    "HeapInuse": 253952,
    "HeapObjects": 174,
    "HeapReleased": 0,
    "HeapSys": 753664,
    "LastGC": 0,
    "Lookups": 3,
    "MCacheInuse": 4800,
    "MCacheSys": 16384,
    "MSpanInuse": 5280,
    "MSpanSys": 16384,
    "Mallocs": 181,
    "NextGC": 4194304,
    "NumGC": 0,
    "OtherSys": 559106,
    "PauseEnd": [
        0,
        ...
        0
    ],
    "PauseNs": [
        0,
        ...
        0
    ],
    "PauseTotalNs": 0,
    "StackInuse": 294912,
    "StackSys": 294912,
    "Sys": 1740800,
    "TotalAlloc": 63776
}

```

You can refer to the [documentation](https://golang.org/pkg/runtime/#MemStats) for the meaning of each fields.

## Using pprof

Supposing you didn't install any signal mechanism or other similar approach to tell your application print the above output, but luckily, you've installed the `pprof` endpoint to your web application, you can still get the MemStats information super easily:

{% highlight go %}
package main

import "net/http"
import _ "net/http/pprof"

func main() {
	http.ListenAndServe(":8000", nil)
}
{% endhighlight %}

Just accessing the `/debug/pprof/heap` endpoint via for example curl:

```
$ curl -s dev:8000/debug/pprof/heap | tail -n21
```

You can also get the similar output:

```
# runtime.MemStats
# Alloc = 672728
# TotalAlloc = 672728
# Sys = 3084288
# Lookups = 11
# Mallocs = 5228
# Frees = 151
# HeapAlloc = 672728
# HeapSys = 1736704
# HeapIdle = 622592
# HeapInuse = 1114112
# HeapReleased = 0
# HeapObjects = 5077
# Stack = 360448 / 360448
# MSpan = 17600 / 32768
# MCache = 4800 / 16384
# BuckHashSys = 2598
# NextGC = 4194304
# PauseNs = [0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0]
# NumGC = 0
# DebugGC = false
```

## Using gdb + python

We can still use *gdb* plus some gdb python extension scripts to do it.

### The basic idea

0. Attach to the process using gdb
0. Use gdb python extension scripts to read the MemStats directly from memory

### Requirements

0. gdb (You may need [codesign](https://gist.github.com/hlissner/898b7dfc0a3b63824a70e15cd0180154) your gdb if you are on a Mac OS)
0. the extension: <https://raw.githubusercontent.com/ibigbug/runtime-gdb.py/master/runtime-gdb.py>


### Let's go

#### First, start the demo application

```
$ cat a.go
package main

import (
	"fmt"
	"net/http"
	_ "net/http/pprof"
	"os"
)

func main() {
	fmt.Println(os.Getpid())
	http.ListenAndServe(":8000", nil)
}

$ go build -o test a.go
$ ./test

# let the runtime updatememstats
$ curl dev:8000/debug/pprof/heap -o /dev/null
```

#### Second, attach to the process

```
$ gdb -pid $PID
```

#### Third, use the script

```
(gdb) source runtime-gdb.py
```

Then you have a new command named `mstat` now, just type it:

```
(gdb) mstat
runtime.MemStats
Alloc = 583200
TotalAlloc = 583200
Sys = 2822144
Lookups = 5
Mallocs = 4951
Frees = 99
HeapAlloc = 583200
HeapSys = 1736704
HeapIdle = 843776
HeapInuse = 892928
HeapReleased = 0
HeapObjects = 4852
Stack = 360448 / 360448
MSpan = 13440 / 16384
MCache = 4800 / 16384
BuckHashSys = 2598
GCSys = 131072
OtherSys = 558554
NextGC = 4194304
LastGC = 0
NumGC = 0
GCCPUFraction = 0
DebugGC = false
```
