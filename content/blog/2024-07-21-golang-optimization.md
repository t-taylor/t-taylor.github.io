+++
title = "Golang Tricks for faster api throughput"

description = "Don't rely on the GC"

tags = ["computers"]
+++

Recently at work, I've been doing some greenfield work creating an endpoint in golang that can serve a large static dataset. While doing this, because of the large amount of data throughput I've been running into OOM errors. I was pretty surprised about this considering all the praise that golangs GC gets, but after tinkering with the program I was able to reign in the memory usage.

## Diagnosing

Initially, I found it quite hard to figure out what was happening. In my code, I was loading aprox 15Gib of data into memory, and so I expected to see a large increase while that was loading, and then it would remain relatively constant while running a benchmark. What I was seeing was that sometimes it would OOM while loading the dataset, and while running, it would gradually increase in memory usage until it OOM crashed.

Typically, it's recommended to use `pprof` to determine what behaviour is happening in runtime. I was expecting the memory usage endpoints to be particularly helpful to me, but it would just show the 15Gib chunks I was expecting and nothing else. The biggest help in diagnosing this problem was the CPU profile endpoint. In particular, I saw a 32 bit hash combined with a malloc call node taking up a load of CPU time. From this, I gathered that a large issue was the hashmaps I was using to extract the data and serialise it into JSON. This accounted for the behaviour I was seeing while running benchmarks, but I was still encountering issues while loading the data.

After reading how golang allocates its arrays, I discovered it was due to the fact I wasn't preallocating the array size. As an array gets appended to, golang has to keep on creating larger arrays to move the expanding data. Because it'd be extremely inefficient to keep increasing the array size by 1 element each time we append, it instead does this exponentially. As a result, if the program got unlucky with the array size allocation, sometimes it was trying to allocate 32Gib of mem for my 24Gib system.

As a note, golang allows you to allocate an array without actually filling it with 0'd elements, only seeing the 'capacity'.

```go
package main

import "log"

func main() {
  foo := make([]byte, 10)
  bar := make([]byte, 0, 10)

  log.Println("foo size", len(foo)) // 10
  log.Println("foo cap", cap(foo)) // 10

  log.Println("bar size", len(bar)) // 0
  log.Println("bar cap", cap(bar)) // 10

  bar = append(bar, 2) // this doesn't require a malloc call
  foo = append(foo, 2) // but this does
}

```
## `sync.Pool`

One way I succeeded in removing the memory usage for allocating hashmaps, was to allocate less! `sync.Pool` gives you a useful way of reusing already allocated structures. By init'ing it first with a structure constructer, it provides 2 thread safe methods to use: `func (p *Pool) Get() any` & `func (p *Pool) Put(x any)`.

```go
package main

import "sync"

func main() {

  p := &sync.Pool{
    New: func() any {
      return make(map[string]string)
    }
  }

  // retrieve an item from the pool. sync.Pool will call New if none exists
  data := p.Get().(map[string]string)

  data["foo"] = "bar"

  // some json encoding/http response
  // ...

  // put it back when you're done with it
  p.Put(data)
}
```

You might notice one possible issue with this code, namely data sanitization. If we're reusing maps, and we don't overwrite data we might get some previously assigned key value pairs. By iterating through each key and `delete()`ing it, we solve this issue. This has the added benefit of removing extra mallocs, as we will reduce the number of total hashmap bins to store the map elements. Below I show an example wrapping the `sync.Pool` which returns a close function which should be called after finishing with the map.

```go
package data

type Pool struct {
  pl *sync.Pool
}

func NewPool() (*Pool) {
  return &Pool{
    pl: &sync.Pool{
      New: func() any {
        return make(map[string]string)
      }
    }
  }
}

func (p *Pool) Get() (data any, close func()) {
  data = p.pl.Get()
  close = func() {
    for k := range data {
      delete(data, k)
    }

    p.pl.Put(data)
  }
}
```

Note with this method, we don't have to bother providing a `Put` method. Instead, we can just call `defer close()` after every call to our `Get` method.

## Preventative methods with a soft memory limit

One of the features of `pprof` allows you to see what any cpu is running at any time. This normally identifies code by showing the goroutine ID, or more importantly for me in this case, when it's running a GC cycle. I noticed that golang does not care if you're about to run out of memory and will happily keep on running with a load of reserved memory until it asks for more and then kills itself with OOM. To resolve this, by setting `GOMEMLIMIT` to a target memory limit, the GC will scale the number of cycles it runs as it gets closer to the set limit. By setting this to 95% of the system's memory, it totally eliminated any OOM crashes. For me, this will become a nessaccary setting for any critical program.

## Quick wins
### Don't use `encoding/json`

One easy optimisation was to switch out the standard library encoder for [goccy/go-json](https://github.com/goccy/go-json). It's fully api compatable so it's as easy as changing the import at the top of the source file. It's more efficient by reusing byte buffers & avoiding reflection by instead looking directly at the type pointer in the underlying golang implementation of interfaces/types.

### Encoding the json body buffer directly

Using `json.Marshal` is easy to use, but one downside is the entire response is loaded into memory in a `[]byte` var. To avoid this, and instead write the encoding straight to the http reponse, a `json.Encoder` can be used. Passing in the http writer to the constructer allows you to encoder the response directly to the socket without having to store the full encoded bytes in memory.

## TODO

golang hashmaps have a lot of overhead. By the very nature of a hashmap, it requires a lot of hashing, memory allocation, and cpu cycles. In my case for basically just using the map for data serialisation, is a lot of wasted work. One alernative that I couldn't get working was instead using a struct with 2 arrays. One for headers and one for values. Obviously worse lookup, but we're iterating through each element anyway so it doesn't matter for our use case.

```go
// This also has the added benefit of not having to reallocate the header map.
// Just pass the same array through each time.
type VecMap struct {
  headers []string
  values []string
}
```

For the life of me, I couldn't get the json serialisation to work faster. By using the `benchmem` option in testing, I always had way better stats for allocating the data in the inital step, but with serialisation I was always behind. Looking at the `goccy/go-json` code didn't help much either. To get that boost in performance they use the underlying implementation of golang data types, and so reading through it was very opaque. By this point I was getting good performance anyway, so this is on the backburner until something calls for it.
