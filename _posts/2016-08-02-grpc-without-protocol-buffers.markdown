---
published: true
title: gRPC without Protocol Buffers
layout: post
---
At Lantern, we use a homegrown time-series Go-based database engine called [tibsdb](https://github.com/getlantern/tibsdb).  Recently, I decided that I'd like to have a CLI for it, similar to the CLIs provided with other databases like Redis or HBase.  For a CLI to work, it needs to have some way to communicate with the server.  Enter RPC!

```
  CLI         Server
   ^             ^
   |             |
   v             v
  ------ RPC -------
```   

Redis uses a custom text-based protocol for its line-level RPC, whereas HBase uses [thrift](https://thrift.apache.org/) to generate its RPC layer.

Adam introduced me to [gRPC](http://www.grpc.io/) and it seemed like it could be a good fit here.  It's got great support in [Go](https://godoc.org/google.golang.org/grpc), supports [streaming](http://www.grpc.io/docs/guides/concepts.html#server-streaming-rpc) in either or both directions and looks generally well designed.  There was just one hangup...

## Protocol Buffers Create Problems for My Style of Development
Basically, I don't like using IDL, for a variety of reasons.

### Code is No Longer Go Gettable
You need to install and run an external tool (`protoc`) in order to keep code up-to-date with your `.proto` files.  Alternately, you could just not do this and risk your code and your IDL getting out of sync.  I'm not sure which is worse.

### Domain Model vs Data Transfer Objects (DTOs)
Defining and creating a set of message types in IDL, outside of the main application code, reopens the old debate about whether to share parts of the application's domain model over RPC or to enforce a strict data transfer layer with its own types.  With Protocol Buffers, I have two options:

1. Use Protocol Buffers to define and generate the datatypes that I use in the application.  This is a non-starter because I may want to add my own non-exported fields, embed other types, etc.

2. Use Protocol Buffers to define DTOs.  This by definition means that I will need to populate those DTOs from my internal data types, which by definition means copying.  In the context of a database API that may return large quantities of data to its client, this seems non-optimal.

### Annoying Handling of time.Time
...


## MessagePack to the Rescue!
[MessagePack](http://msgpack.org/index.html) is a schemaless, fast and efficient binary serialization format.  Using it, I can serialize whatever data I want, and I don't need to mess with IDL and code generation.

gRPC says that it supports encodings other than Protocol Buffers, but I didn't find any good examples of this in practice.  Thankfully, I was able to look at the generated code for protocol buffers and go from there.

### Custom gRPC Codec
This was the easy part.

```golang
package rpc

import (
	"gopkg.in/vmihailenco/msgpack.v2"
)

type MsgPackCodec struct {
}

func (c *MsgPackCodec) Marshal(v interface{}) ([]byte, error) {
	return msgpack.Marshal(v)
}

func (c *MsgPackCodec) Unmarshal(data []byte, v interface{}) error {
	return msgpack.Unmarshal(data, v)
}

func (c *MsgPackCodec) String() string {
	return "MsgPackCodec"
}
```

To use the custom code from a server, you do something like this:

```golang
import ”google.golang.org/grpc"

grpc.NewServer(grpc.CustomCodec(&MsgPackCodec{}))
```

To use the custom code from a client, you do this:

```golang
import ”google.golang.org/grpc"

grpc.Dial(addr, grpc.WithCodec(&MsgPackCodec{}))
```

### Building a Server
Without the use of Protocol Buffers, it’s necessary to hand code the service interface and implementation.  After breaking down the auto-generated code from Protocol Buffers, it turns out that this isn’t hard to do by hand.

In this example, we’re implementing a [server-streaming](http://www.grpc.io/docs/guides/concepts.html#server-streaming-rpc) service that takes a query and returns a stream of rows.

#### The interface

```golang
import "google.golang.org/grpc"

type Server interface {
	Query(*Query, grpc.ServerStream) error
}
```

Notice that the Query function takes two parameters, the `Query` object (passed from the client) and a `grpc.ServerStream`, which gets passed to the service by gRPC.  The `grpc.ServerStream` is what allows us to respond with a stream of results.

#### The implementation

```golang
import "google.golang.org/grpc"

type server struct {
	db *tibsdb.DB
}

func (s *server) Query(query *Query, stream grpc.ServerStream) error {
	q, err := s.db.SQLQuery(query.SQL)
	if err != nil {
		return err
	}
	result, err := q.Run()
	if err != nil {
		return err
	}

	fields, entries := result.Fields, result.Entries
	result.Fields = nil
	result.Entries = nil

	// Send header
	err = stream.SendMsg(result)
	if err != nil {
		return err
	}

	// Write each entry
	for _, entry := range entries {
		row := &Row{
			Dims:   make([]interface{}, 0, len(result.GroupBy)),
			Fields: make([][]float64, 0, len(fields)),
		}
		for _, dim := range result.GroupBy {
			row.Dims = append(row.Dims, entry.Dims[dim])
		}
		for i, field := range fields {
			vals := entry.Fields[i]
			values := make([]float64, 0, result.NumPeriods)
			for j := 0; j < result.NumPeriods; j++ {
				val, _ := vals.ValueAt(j, field)
				values = append(values, val)
			}
			row.Fields = append(row.Fields, values)
		}
		err = stream.SendMsg(row)
		if err != nil {
			return err
		}
	}

	return nil
}
```

Notice a few things about this:

1. We send two different types of datum to the stream, first the result from `db.SQLQuery` and then individual `Row`s.  This is possible because streams are untyped.  This could have been accomplished using an `Any` type in Protocol Buffers.

2. The result from `db.SQLQuery` is part of the application domain, but we can happily send it along since MessagePack will serialize whatever we throw at it.  Note however that we do remove a couple of things that we don’t want to send to the client.

3. `Row` is actually a data-transfer object created specifically for this API.  That’s because it works better for the client than sending the Entries from which the Rows are made.

#### Service Registration
To register the service with gRPC, we need a service description:

```golang
import "google.golang.org/grpc"

var serviceDesc = grpc.ServiceDesc{
	ServiceName: "TibsDB",
	HandlerType: (*Server)(nil),
	Methods:     []grpc.MethodDesc{},
	Streams: []grpc.StreamDesc{
		{
			StreamName:    "Query",
			Handler:       queryHandler,
			ServerStreams: true,
		},
	},
}
```

We can then use it like this:

```golang
import “net”
import "google.golang.org/grpc"

func Serve(db *tibsdb.DB, l net.Listener) error {
	gs := grpc.NewServer(grpc.CustomCodec(msgpackCodec))
	gs.RegisterService(&serviceDesc, &server{db})
	return gs.Serve(l)
}
```

### Building a Client

#### The interface

```golang
import "google.golang.org/grpc"
import "golang.org/x/net/context"

type Client interface {
	Query(ctx context.Context, in *Query, opts ...grpc.CallOption) (*tibsdb.QueryResult, func() (*Row, error), error)
```

- Notice that in addition to the actual `Query` input parameter, we also need a `context.Context` and an optional array of `grpc.CallOption`s.

- Also notice that our interface takes advantage of Go’s support for multiple return parameters.  Protocol Buffers IDL can’t do this.


#### The implementation

```golang
import "google.golang.org/grpc"
import "golang.org/x/net/context"

type tibsDBClient struct {
	cc *grpc.ClientConn
}

func (c *tibsDBClient) Query(ctx context.Context, in *Query, opts ...grpc.CallOption) (*tibsdb.QueryResult, func() (*Row, error), error) {
	stream, err := grpc.NewClientStream(ctx, &serviceDesc.Streams[0], c.cc, "/TibsDB/Query", opts...)
	if err != nil {
		return nil, nil, err
	}
	if err := stream.SendMsg(in); err != nil {
		return nil, nil, err
	}
	if err := stream.CloseSend(); err != nil {
		return nil, nil, err
	}

	result := &tibsdb.QueryResult{}
	err = stream.RecvMsg(result)
	if err != nil {
		return nil, nil, err
	}

	nextRow := func() (*Row, error) {
		row := &Row{}
		err := stream.RecvMsg(row)
		return row, err
	}
	return result, nextRow, nil
}
```

1. We use `grpc.NewClientStream` to get a stub for the service.  Note that this refers directly to the stream defined in the service descriptor.

2. Note that we also have to specify the method as `/<ServiceName>/<StreamName>`, i.e. `/TibsDB/Query`.

3. Once we have a stream, we can send and receive messages to/from it.