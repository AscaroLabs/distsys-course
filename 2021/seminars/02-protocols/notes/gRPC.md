# Introduction

## Run a gRPC application

Run the server:

```sh
$ python greeter_server.py
```
From another terminal, run the client:

```sh
$ python greeter_client.py
```
## Update the gRPC service

For now all you need to know is that both the server and the client “stub” have a SayHello RPC method that takes a HelloRequest parameter from the client and returns a HelloReply from the server, and that this method is defined like this:

```protobuf
// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```
Let’s update this so that the Greeter service has two methods. Edit `examples/protos/helloworld.proto` and update it with a new SayHelloAgain method, with the same request and response types:

```protobuf
// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
  // Sends another greeting
  rpc SayHelloAgain (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}

```

## Generate gRPC code
Next we need to update the gRPC code used by our application to use the new service definition.

From the `examples/python/helloworld` directory, run:
```sh
$ python -m grpc_tools.protoc -I../../protos --python_out=. --grpc_python_out=. ../../protos/helloworld.proto
```

This regenerates `helloworld_pb2.py` which contains our generated request and response classes and `helloworld_pb2_grpc.py` which contains our generated client and server classes.

## Update and run the application 

### Update the server 

In the same directory, open `greeter_server.py`. Implement the new method like this:


```python
class Greeter(helloworld_pb2_grpc.GreeterServicer):

  def SayHello(self, request, context):
    return helloworld_pb2.HelloReply(message='Hello, %s!' % request.name)

  def SayHelloAgain(self, request, context):
    return helloworld_pb2.HelloReply(message='Hello again, %s!' % request.name)
...
```
### Update the client

In the same directory, open `greeter_client.py`. Call the new method like this:
```python
def run():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = helloworld_pb2_grpc.GreeterStub(channel)
        response = stub.SayHello(helloworld_pb2.HelloRequest(name='you'))
        print("Greeter client received: " + response.message)
        response = stub.SayHelloAgain(helloworld_pb2.HelloRequest(name='you'))
        print("Greeter client received: " + response.message)
```

# Core concepts, architecture and lifecycle

## Service definition

Like many RPC systems, gRPC is based around the idea of defining a service, specifying the methods that can be called remotely with their parameters and return types.

```protobuf
service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```
gRPC lets you define four kinds of service method:
- `Unary RPCs` where the client sends a single request to the server and gets a single response back, just like a normal function call.
  ```protobuf
  rpc SayHello(HelloRequest) returns (HelloResponse);
  ```
- `Server streaming RPCs` where the client sends a request to the server and gets a stream to read a sequence of messages back. The client reads from the returned stream until there are no more messages. gRPC guarantees message ordering within an individual RPC call.
  ```protobuf
  rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse);
  ```
- `Client streaming RPCs` where the client writes a sequence of messages and sends them to the server, again using a provided stream. Once the client has finished writing the messages, it waits for the server to read them and return its response. Again gRPC guarantees message ordering within an individual RPC call.
  ```protobuf
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
  ```
- `Bidirectional streaming RPCs` where both sides send a sequence of messages using a read-write stream. The two streams operate independently, so clients and servers can read and write in whatever order they like: for example, the server could wait to receive all the client messages before writing its responses, or it could alternately read a message then write a message, or some other combination of reads and writes. The order of messages in each stream is preserved.
  ```protobuf
  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
  ```

## Using the API 
Starting from a service definition in a `.proto` file, gRPC provides protocol buffer compiler plugins that generate client- and server-side code. gRPC users typically call these APIs on the client side and implement the corresponding API on the server side.
- On the `server side`, the server implements the methods declared by the service and runs a gRPC server to handle client calls. The gRPC infrastructure decodes incoming requests, executes service methods, and encodes service responses.
- On the `client side`, the client has a local object known as stub (for some languages, the preferred term is client) that implements the same methods as the service. The client can then just call those methods on the local object, wrapping the parameters for the call in the appropriate protocol buffer message type - gRPC looks after sending the request(s) to the server and returning the server’s protocol buffer response(s).

### Synchronous vs. asynchronous 
Synchronous RPC calls that block until a response arrives from the server are the closest approximation to the abstraction of a procedure call that RPC aspires to. On the other hand, networks are inherently asynchronous and in many scenarios it’s useful to be able to start RPCs without blocking the current thread.

The gRPC programming API in most languages comes in both synchronous and asynchronous flavors. You can find out more in each language’s tutorial and reference documentation.

## RPC life cycle 

### Unary RPC

First consider the simplest type of RPC where the client sends a single request and gets back a single response.

1. Once the client calls a stub method, the server is notified that the RPC has been invoked with the client’s metadata for this call, the method name, and the specified deadline if applicable.
2. The server can then either send back its own initial metadata (which must be sent before any response) straight away, or wait for the client’s request message. Which happens first, is application-specific.
3. Once the server has the client’s request message, it does whatever work is necessary to create and populate a response. The response is then returned (if successful) to the client together with status details (status code and optional status message) and optional trailing metadata.
4. If the response status is OK, then the client gets the response, which completes the call on the client side.

### Server streaming RPC

A server-streaming RPC is similar to a unary RPC, except that the server returns a stream of messages in response to a client’s request. After sending all its messages, the server’s status details (status code and optional status message) and optional trailing metadata are sent to the client. This completes processing on the server side. The client completes once it has all the server’s messages.

### Client streaming RPC

A client-streaming RPC is similar to a unary RPC, except that the client sends a stream of messages to the server instead of a single message. The server responds with a single message (along with its status details and optional trailing metadata), typically but not necessarily after it has received all the client’s messages.

### Bidirectional streaming RPC 

In a bidirectional streaming RPC, the call is initiated by the client invoking the method and the server receiving the client metadata, method name, and deadline. The server can choose to send back its initial metadata or wait for the client to start streaming messages.

Client- and server-side stream processing is application specific. Since the two streams are independent, the client and server can read and write messages in any order. For example, a server can wait until it has received all of a client’s messages before writing its messages, or the server and client can play “ping-pong” – the server gets a request, then sends back a response, then the client sends another request based on the response, and so on.

## Deadlines/Timeouts 

gRPC allows clients to specify how long they are willing to wait for an RPC to complete before the RPC is terminated with a DEADLINE_EXCEEDED error. On the server side, the server can query to see if a particular RPC has timed out, or how much time is left to complete the RPC.

Specifying a deadline or timeout is language specific: some language APIs work in terms of timeouts (durations of time), and some language APIs work in terms of a deadline (a fixed point in time) and may or may not have a default deadline.

## RPC termination

In gRPC, both the client and server make independent and local determinations of the success of the call, and their conclusions may not match. This means that, for example, you could have an RPC that finishes successfully on the server side (“I have sent all my responses!”) but fails on the client side (“The responses arrived after my deadline!”). It’s also possible for a server to decide to complete before a client has sent all its requests.

## Metadata

Metadata is information about a particular RPC call (such as authentication details) in the form of a list of key-value pairs, where the keys are strings and the values are typically strings, but can be binary data. Metadata is opaque to gRPC itself - it lets the client provide information associated with the call to the server and vice versa.

Access to metadata is language dependent.

## Channels

A gRPC channel provides a connection to a gRPC server on a specified host and port. It is used when creating a client stub. Clients can specify channel arguments to modify gRPC’s default behavior, such as switching message compression on or off. A channel has state, including connected and idle.

How gRPC deals with closing a channel is language dependent. Some languages also permit querying channel state.

# Basics tutorial

## Defining the service

Your first step is to define the gRPC service and the method request and response types using protocol buffers. You can see the complete `.proto` file in `examples/protos/route_guide.proto`

To define a service, you specify a named service in your `.proto` file:

```protobuf
service RouteGuide {
   // (Method definitions not shown)
}
```
Then you define `rpc` methods inside your service definition, specifying their request and response types. gRPC lets you define four kinds of service method, all of which are used in the `RouteGuide service`:

- A `simple RPC` where the client sends a request to the server using the stub and waits for a response to come back, just like a normal function call.
  ```protobuf
  // Obtains the feature at a given position.
  rpc GetFeature(Point) returns (Feature) {}
  ```
- A `response-streaming RPC` where the client sends a request to the server and gets a stream to read a sequence of messages back. The client reads from the returned stream until there are no more messages. As you can see in the example, you specify a response-streaming method by placing the `stream` keyword before the response type.
  ```protobuf 
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
  ```
- A `request-streaming RPC` where the client writes a sequence of messages and sends them to the server, again using a provided stream. Once the client has finished writing the messages, it waits for the server to read them all and return its response. You specify a request-streaming method by placing the `stream` keyword before the request type.
  ```protobuf
  // Accepts a stream of Points on a route being traversed, returning a
  // RouteSummary when traversal is completed.
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
  ```
- A `bidirectionally-streaming RPC` where both sides send a sequence of messages using a read-write stream. The two streams operate independently, so clients and servers can read and write in whatever order they like: for example, the server could wait to receive all the client messages before writing its responses, or it could alternately read a message then write a message, or some other combination of reads and writes. The order of messages in each stream is preserved. You specify this type of method by placing the stream keyword before both the request and the response.
  ```protobuf
  // Accepts a stream of RouteNotes sent while a route is being traversed,
  // while receiving other RouteNotes (e.g. from other users).
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
  ```
Your `.proto` file also contains protocol buffer message type definitions for all the request and response types used in our service methods - for example, here’s the `Point` message type:
```protobuf
// Points are represented as latitude-longitude pairs in the E7 representation
// (degrees multiplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

## Generating client and server code

Use the following command to generate the Python code:
```sh
python -m grpc_tools.protoc -I../../protos --python_out=. --grpc_python_out=. ../../protos/route_guide.proto
```

## Creating the server

First let’s look at how you create a `RouteGuide` server.

Creating and running a `RouteGuide` server breaks down into two work items:

- Implementing the servicer interface generated from our service definition with functions that perform the actual “work” of the service.

- Running a gRPC server to listen for requests from clients and transmit responses.

## Implementing RouteGuide

`route_guide_server.py` has a `RouteGuideServicer` class that subclasses the generated class `route_guide_pb2_grpc.RouteGuideServicer`:
```python
# RouteGuideServicer provides an implementation of the methods of the RouteGuide service.
class RouteGuideServicer(route_guide_pb2_grpc.RouteGuideServicer):
```
`RouteGuideServicer` implements all the `RouteGuide` service methods.

### Simple RPC 

Let’s look at the simplest type first, `GetFeature`, which just gets a `Point` from the client and returns the corresponding feature information from its database in a `Feature`.

```python
def GetFeature(self, request, context):
  feature = get_feature(self.db, request)
  if feature is None:
    return route_guide_pb2.Feature(name="", location=request)
  else:
    return feature
```
The method is passed a `route_guide_pb2.Point` request for the RPC, and a `grpc.ServicerContext` object that provides RPC-specific information such as timeout limits. It returns a `route_guide_pb2.Feature` response.

### Response-streaming RPC 

Now let’s look at the next method. `ListFeatures` is a response-streaming RPC that sends multiple `Features` to the client.

```python
def ListFeatures(self, request, context):
  left = min(request.lo.longitude, request.hi.longitude)
  right = max(request.lo.longitude, request.hi.longitude)
  top = max(request.lo.latitude, request.hi.latitude)
  bottom = min(request.lo.latitude, request.hi.latitude)
  for feature in self.db:
    if (feature.location.longitude >= left and
        feature.location.longitude <= right and
        feature.location.latitude >= bottom and
        feature.location.latitude <= top):
      yield feature
```

Here the request message is a `route_guide_pb2.Rectangle` within which the client wants to find `Features`. Instead of returning a single response the method yields zero or more responses.

### Request-streaming RPC 

The request-streaming method `RecordRoute` uses an iterator of request values and returns a single response value.

```python
def RecordRoute(self, request_iterator, context):
  point_count = 0
  feature_count = 0
  distance = 0.0
  prev_point = None

  start_time = time.time()
  for point in request_iterator:
    point_count += 1
    if get_feature(self.db, point):
      feature_count += 1
    if prev_point:
      distance += get_distance(prev_point, point)
    prev_point = point

  elapsed_time = time.time() - start_time
  return route_guide_pb2.RouteSummary(point_count=point_count,
                                      feature_count=feature_count,
                                      distance=int(distance),
                                      elapsed_time=int(elapsed_time))
                            
```

### Bidirectional streaming RPC 

Lastly let’s look at the bidirectionally-streaming method `RouteChat`.

```python
def RouteChat(self, request_iterator, context):
  prev_notes = []
  for new_note in request_iterator:
    for prev_note in prev_notes:
      if prev_note.location == new_note.location:
        yield prev_note
    prev_notes.append(new_note)
```
This method’s semantics are a combination of those of the request-streaming method and the response-streaming method. It is passed an iterator of request values and is itself an iterator of response values.

## Starting the server

Once you have implemented all the `RouteGuide` methods, the next step is to start up a gRPC server so that clients can actually use your service:
```python
def serve():
  server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
  route_guide_pb2_grpc.add_RouteGuideServicer_to_server(
      RouteGuideServicer(), server)
  server.add_insecure_port('[::]:50051')
  server.start()
  server.wait_for_termination()
```

The server `start()` method is non-blocking. A new thread will be instantiated to handle requests. The thread calling `server.start()` will often not have any other work to do in the meantime. In this case, you can call server.`wait_for_termination()` to cleanly block the calling thread until the server terminates.

## Creating the client
You can see the complete example client code in `examples/python/route_guide/route_guide_client.py`

### Creating a stub

To call service methods, we first need to create a `stub`.

We instantiate the `RouteGuideStub` class of the `route_guide_pb2_grpc` module, generated from our `.proto`.

```python
channel = grpc.insecure_channel('localhost:50051')
stub = route_guide_pb2_grpc.RouteGuideStub(channel)
```

### Calling service methods 
 For RPC methods that return a single response (“response-unary” methods), gRPC Python supports both synchronous (blocking) and asynchronous (non-blocking) control flow semantics. For response-streaming RPC methods, calls immediately return an iterator of response values. Calls to that iterator’s next() method block until the response to be yielded from the iterator becomes available.

### Simple RPC 

A synchronous call to the simple RPC `GetFeature` is nearly as straightforward as calling a local method. The RPC call waits for the server to respond, and will either return a response or raise an exception:

```python
feature = stub.GetFeature(point)
```

An asynchronous call to GetFeature is similar, but like calling a local method asynchronously in a thread pool:

```python
feature_future = stub.GetFeature.future(point)
feature = feature_future.result()
```

### Response-streaming RPC

Calling the response-streaming `ListFeatures` is similar to working with sequence types:

```python
for feature in stub.ListFeatures(rectangle):
```

### Request-streaming RPC 

Calling the request-streaming `RecordRoute` is similar to passing an iterator to a local method. Like the simple RPC above that also returns a single response, it can be called synchronously or asynchronously:

```python
route_summary = stub.RecordRoute(point_iterator)
```

```python
route_summary_future = stub.RecordRoute.future(point_iterator)
route_summary = route_summary_future.result()
```

### Bidirectional streaming RPC 

Calling the bidirectionally-streaming `RouteChat` has (as is the case on the service-side) a combination of the request-streaming and response-streaming semantics:

```python
for received_route_note in stub.RouteChat(sent_route_note_iterator):
```
