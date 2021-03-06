# bcache

[![godoc](https://godoc.org/github.com/iwanbk/bcache?status.svg)](http://godoc.org/github.com/iwanbk/bcache)
[![Build Status](https://travis-ci.org/iwanbk/bcache.svg?branch=master)](https://travis-ci.org/iwanbk/bcache)
[![codecov](https://codecov.io/gh/iwanbk/bcache/branch/master/graph/badge.svg)](https://codecov.io/gh/iwanbk/bcache)
[![Go Report Card](https://goreportcard.com/badge/github.com/iwanbk/bcache)](https://goreportcard.com/report/github.com/iwanbk/bcache)
[![Maintainability](https://api.codeclimate.com/v1/badges/0535095fdd215f2e22ad/maintainability)](https://codeclimate.com/github/iwanbk/bcache/maintainability)

A Go Library to create distributed in-memory cache inside your app.

## Features

- LRU cache with configurable maximum keys
- Eventual Consistency synchronization between peers
- Data are replicated to all nodes
- cache filling mechanism. When the cache of the given key is not exist, bcache coordinates cache fills such that only one call populates the cache to avoid thundering herd or [cache stampede](https://en.wikipedia.org/wiki/Cache_stampede)

## Why using it

- if extra network hops needed by external caches like `redis` or `memcached` is not acceptable for you
- you only need cache with simple `Set`, `Get`, and `Delete` operation
- you have enough RAM to hold the cache data

## How it Works

1. Nodes find each other using [Gossip Protocol](https://en.wikipedia.org/wiki/Gossip_protocol)

Only need to specify one or few nodes as bootstrap nodes, and all nodes will find each other using gossip protocol

2. When there is cache `Set` and `Delete`, the event will be propagated to all of the nodes.

So, all of the nodes will eventually have synced data.



## Cache filling

Cache filling mechanism is provided in [GetWithFiller](https://godoc.org/github.com/iwanbk/bcache#Bcache.GetWithFiller) func.

When the cache for the given key is not exists:
- it will call the provided `Filler`
- set the cache using value returned by the `Filler`

Even there are many goroutines which call `GetWithFiller`, the given `Filler` func
will only called once for each of the key.
Cache stampede could be avoided this way.

## Quick Start

In server 1
```go
bc, err := New(Config{
	// PeerID:     1, // leave it, will be set automatically based on mac addr
	ListenAddr: "192.168.0.1:12345",
	Peers:      nil, // it nil because we will use this node as bootstrap node
	MaxKeys:    1000,
	Logger:     logrus.New(),
})
if err != nil {
    log.Fatalf("failed to create cache: %v", err)
}
bc.Set("my_key", "my_val",86400)
```

In server 2
```go
bc, err := New(Config{
	// PeerID:     2, // leave it, will be set automatically based on mac addr
	ListenAddr: "192.168.0.2:12345",
	Peers:      []string{"192.168.0.1:12345"},
	MaxKeys:    1000,
	Logger:     logrus.New(),
})
if err != nil {
    log.Fatalf("failed to create cache: %v", err)
}
bc.Set("my_key2", "my_val2", 86400)
```

In server 3
```go
bc, err := New(Config{
	// PeerID:     3,// will be set automatically based on mac addr
	ListenAddr: "192.168.0.3:12345",
	Peers:      []string{"192.168.0.1:12345"},
	MaxKeys:    1000,
	Logger:     logrus.New(),
})
if err != nil {
    log.Fatalf("failed to create cache: %v", err)
}
val, exists := bc.Get("my_key2")
```

### GetWithFiller example

```go
c, err := New(Config{
	PeerID:     3,
	ListenAddr: "192.168.0.3:12345",
	Peers:      []string{"192.168.0.1:12345"},
	MaxKeys:    1000,
})
if err != nil {
    log.Fatalf("failed to create cache: %v", err)
}
val, exp,err  := bc.GetWithFiller("my_key2",func(key string) (string, error) {
        // get value from database
         .....
         //
		return value, 0, nil
}, 86400)
```

## Credits

- [weaveworks/mesh](https://github.com/weaveworks/mesh) for the gossip library
- [groupcache](https://github.com/golang/groupcache) for the inspiration