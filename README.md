# Siberite
[![Build Status](https://travis-ci.org/bogdanovich/siberite.svg?branch=master)](https://travis-ci.org/bogdanovich/siberite)
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/bogdanovich/siberite?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Go Walker](http://gowalker.org/api/v1/badge)](https://gowalker.org/github.com/bogdanovich/siberite)

Siberite is a simple leveldb backed message queue server<br>
([twitter/kestrel](https://github.com/twitter/kestrel), [wavii/darner](https://github.com/wavii/darner) rewritten in Go).

Siberite is a very simple message queue server.  Unlike in-memory servers such as [redis](http://redis.io/), Siberite is
designed to handle queues much larger than what can be held in RAM.  And unlike enterprise queue servers such as
[RabbitMQ](http://www.rabbitmq.com/), Siberite keeps all messages **out of process**,
using [goleveldb](https://github.com/syndtr/goleveldb) as a persistent storage.

The result is a durable queue server that uses a small amount of in-resident memory regardless of queue size.

Siberite is based on Robey Pointer's [Kestrel](https://github.com/robey/kestrel) - simple, distributed message queue.
Like Kestrel, Siberite follows the "No talking! Shhh!" approach to distributed queues:
A single Siberite server has a set of queues identified by name.  Each queue is a strictly-ordered FIFO,
and querying from a fleet of Siberite servers provides a loosely-ordered queue.
Siberite also supports Kestrel's two-phase reliable fetch: if a client disconnects before confirming it handled
a message, the message will be handed to the next client.

Compared to Kestrel and Darner, Siberite is easier to build, maintain and distribute.
It uses an order of magnitude less memory compared to Kestrel, but has less configuration far fewer features.

Siberite is used at [Spyonweb.com](http://spyonweb.com).<br>
We used to use Darner before, but got 2 large production queues corrupted at some point and decided to rewrite it in Go.

## Features

1. Multiple consumer groups per queue using `get <queue>:<cursor>` syntax.

  - When you read an item in a usual way: `get <queue>`, item gets expired and deleted.
  - When you read an item using cursor syntax `get <queue>:<cursor>`, a durable
    cursor gets initialized. It shifts forward with every read without deleting
    any messages in the source queue. Number of cursors per queue is not limited.
  - If you continue reads from the source queue directly, siberite will continue
    deleting messages from the head of that queue. Any existing cursor that is
    internally points to an already deleted message will catch up during next read
    and will start serving messages from the current source queue head.
  - Durable cursors are also support two-phase reliable reads. All failed reliable
    reads for each cursor are stored in cursor's own small persistent queue.

2. Fanout queues

  - Siberite allows you to insert new message into multiple queues at once
    by using the following syntax `set <queue>+<another_queue>+<third_queue> ...`



##Benchmarks

[Siberite performance benchmarks](docs/benchmarks.md)


## Build

Make sure your `GOPATH` is correct

```
go get github.com/bogdanovich/siberite
cd $GOPATH/src/github.com/bogdanovich/siberite
go get ./...
cd siberite
go build siberite.go
mkdir ./data
./siberite -listen localhost:22133 -data ./data
2015/09/22 06:29:38 listening on 127.0.0.1:22133
2015/09/22 06:29:38 initializing...
2015/09/22 06:29:38 data directory:  ./data
```

or download [darwin-x86_64 or linux-x86_64 builds](https://github.com/bogdanovich/siberite/releases)

## Protocol

Siberite follows the same protocol as [Kestrel](http://github.com/robey/kestrel/blob/master/docs/guide.md#memcache),
which is the memcache TCP text protocol.

[List of compatible clients](docs/clients.md)

## Telnet example

```
telnet localhost 22133
Connected to localhost.
Escape character is '^]'.

set work 0 0 10
1234567890
STORED

set work 0 0 2
12
STORED

get work
VALUE work 0 10
1234567890
END

get work/open
VALUE work 0 2
12
END

get work/close
END

stats
STAT uptime 47
STAT time 1443308758
STAT version siberite-0.4.1
STAT curr_connections 1
STAT total_connections 1
STAT cmd_get 2
STAT cmd_set 2
STAT queue_work_items 0
STAT queue_work_open_transactions 0
END

# other commands:
# get work/peek
# get work/open
# get work/close/open
# get work/abort
# flush work
# delete work
# flush_all
```


## Not supported

  - Waiting a given time limit for a new item to arrive /t=<milliseconds> (allowed by protocol but does nothing)
