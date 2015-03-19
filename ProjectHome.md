Protobuf for Node adds idiomatic [protocol buffer](http://code.google.com/p/protobuf) handling to [node.JS](http://nodejs.org).

It is actually two things in one: firstly, you can marshal protocol
messages to and from Node byte buffers and send them over the
wire in pure JS.

Secondly, you can use protocol messages as a native interface
from JS to C++ add-ons in the same process. It takes away the details of the
V8 and the eio threadpool APIs and makes native extensions to Node
much easier to implement.

## How to use ##

To read/write protocol buffers, protobuf for node needs the parsed
representation of your protocol buffer schema (which is in protobuf
format itself). The protobuf compiler spits out this format:

```
$ vi feeds.proto
package feeds;
message Feed {
  optional string title = 1;
  message Entry {
    optional string title = 1;
  }
  repeated Entry entry = 2;
}
:wq
$ $PROTO/bin/protoc --descriptor_set_out=feeds.desc --include_imports feeds.proto
```

Using this file (read into a Buffer) you can create a 'Schema' object:

```
 var fs = require('fs');
 var Schema = require('protobuf_for_node').Schema;
 var schema = new Schema(fs.readFileSync('feeds.desc'));
```

A Schema object maps fully qualified protocol buffer names to type
objects that know how to marshal JS objects to and from Buffers:

```
 var Feed = schema['feeds.Feed'];
 var aFeed = Feed.parse(aBuffer);
 var serialized = Feed.serialize(aFeed);
```

When marshalling, the serializer accepts any JavaScript object (but
only picks the properties defined in the schema):

```
 var serialized = Feed.serialize({ title: 'Title', ignored: 42 });
 var aFeed = Feed.parse(serialized);  => { title: 'Title' }
```

Note that property names are the camel-cased version of the field names in the protocol buffer description, like for the Java version (http://code.google.com/apis/protocolbuffers/docs/reference/java-generated.html#fields). E.g. "optional string title\_string = 1" will translate into a "titleString" JS property.

Note how we're using uppercase for the type objects like for
constructor functions. That's because they are: when parsing, objects
are constructed using the respective constructor for the message
type. This means that you are able to attach methods:

```
  Feed.prototype.numEntries = function() {
    return this.entry.length;
  };
  var aFeed = Feed.parse(Feed.serialize({ entry: [{}, {}] }));
  aFeed.numEntries()  =>  2
```

### Native Interface ###

Protocol buffers aren't only great for data interchange _between_ processes - you can also use them to send data between code written in JS and C++ _within the same process_. Protobuf for node makes it simple to implement a native add-on without having to touch the V8 api at all: you implement a [protobuf service](http://code.google.com/apis/protocolbuffers/docs/proto.html#services) instead. It's three easy steps:

1. Define the add-on service interface:

```
// An example service to query pwd(3).
package pwd;

// Empty (=no arg) request message.
message EntriesRequest {
}

message Entry {
  optional string name = 1;
  optional int32 uid = 2;
  optional int32 gid = 3;
  optional string home = 4;
  optional string shell = 5;
}
message EntriesResponse {
  repeated Entry entry = 1;
}

service Pwd {
  rpc GetEntries(EntriesRequest) returns (EntriesResponse);
}
```

... and generate the C++ code for it:
```
proto/bin/protoc --cpp_out=. service.proto
```

2. Implement and export the service in an add-on:

```
extern "C" void init(v8::Handle<v8::Object> target) {
  // Look Ma - no V8 api required!

  // Simple synchronous implementation.
  protobuf_for_node::ExportService(
      target, "pwd",
      new (class : public pwd::Pwd {
        virtual void GetEntries(google::protobuf::RpcController*,
                                const pwd::EntriesRequest* request,
                                pwd::EntriesResponse* response,
                                google::protobuf::Closure* done) {
	  struct passwd* pwd;
	  while ((pwd = getpwent())) {
            pwd::Entry* e = response->add_entry();
            e->set_name(pwd->pw_name);
            e->set_uid(pwd->pw_uid);
            e->set_gid(pwd->pw_gid);
            e->set_home(pwd->pw_dir);
            e->set_shell(pwd->pw_shell);
          }
	  setpwent();
	  done->Run();
        }
      }));
```

3. Use it from JS:

```
// prints the user database
puts(JSON.stringify(require('pwd').pwd.GetEntries({}), null, 2));
```

If your service is CPU intensive, you should call it with an extra callback argument: the invocation is automatically placed on the eio thread pool and does not block node:

```
// prints the user database
require('pwd').pwd.GetEntries({}, function(response) {
  puts(JSON.stringify(response), null, 2);
});
```

Your service is free to finish and call the "done" closure asynchronously.

Note that marshalling between JS and the request and response protos
necessarily happens on the JS thread. Passing tons of data will block
just as much using asynchronous invocation.

## Speed ##

The Protobuf for Node add-on relies on the protobuf C++ runtime but it
does not require any generation, compilation or linkage of generated
C++ code. It works reflectively and is thus able to deal with
arbitrary schemas. This means however, that it is not as fast as with
generated code. Simple measurements show that [un](un.md)marshalling is about
between 20% and 50% faster than V8's native JSON support.

Calling C++ services is faster since it avoids marshalling to bytes
but transfers data directly between the JS objects and the protocol
messages. Also, in this case you compile the generated service
interface into your code, so reflection is faster.

## Building and running ##

You need to have node.js 0.4 and protobuf 2.4.0a installed. protobuf\_for\_node depends on the respective versions of v8 and eio shipped by node. **Use branch node2 for node0.2/protobuf 2.3.0**. If you want to use a later protobuf version, you may need to re-generate the example protoservice code**: `cd example; protoc --cpp_out=. protoservice.proto`.**

To build protobuf for node, do:

```
PROTOBUF=<protobuf prefix> /path/to/node/bin/node-waf configure clean build
```

This will build
a) the protobuf\_for\_node **library** (build/default/libprotobuf\_for\_node\_lib.so).
It is linked to by your service-exporting add-ons and by the
b) protobuf\_for\_node **add-on** (build/default/protobuf\_for\_node.node).

You need a) and the protobuf library in your [DY](DY.md)LD\_LIBRARY\_PATH and
b) in your NODE\_PATH if you require('protobuf\_for\_node').


## Running the examples ##

The examples should be run from within the protobuf-for-node directory, e.g.
"node example/feeds.js".

## Limitations ##

There are mismatches between the protocol buffer and javascript type systems. For example, the JS Number datatype does not offer enough precision to represent proto's [u](u.md)int64 datatypes.

There is no specification for the type conversions performed during JS->proto conversion. There may be surprises when you marshal, say, a JS object into an int64 proto field. That being said, the behavior is more or less in line with the ECMA conversions like ToNumber et al.

The marshalling code does not preserve unknown fields nor extensions.