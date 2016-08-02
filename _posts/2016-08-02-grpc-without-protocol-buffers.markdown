---
published: true
title: gRPC without Protocol Buffers
layout: post
---
At Lantern, we use a homegrown time-series database engine called [tibsdb](https://github.com/getlantern/tibsdb).  Recently, I decided that I'd like to have a CLI for it, similar to the CLIs provided with other databases like Redis or HBase.  For a CLI to work, it needs to have some way to communicate with the server.  Enter RPC!

```
    CLI         Server
      ^                ^
      |                 |
      v               v
--------- RPC -------------
```   

Redis uses a custom text-based protocol for its line-level RPC, whereas HBase uses [thrift](https://thrift.apache.org/) to generate its RPC layer.