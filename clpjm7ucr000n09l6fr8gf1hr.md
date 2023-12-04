---
title: "Capture gRPC Traffic going out from a Server"
datePublished: Wed Nov 29 2023 10:19:39 GMT+0000 (Coordinated Universal Time)
cuid: clpjm7ucr000n09l6fr8gf1hr
slug: capture-grpc-traffic-going-out-from-a-server
canonical: https://keploy.io/blog/technology/capture-grpc-traffic-going-out-from-a-server
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1701251589983/0ec04be9-d7f6-47ab-b730-3f84dc266833.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1701252799714/d24571fa-9919-4059-a780-ef9e403d85fc.png
tags: golang, testing, grpc

---

How does **gRPC** work? A quick Google search would tell you that it uses **HTTP/2.0** under the hood, but that's about it. Most available guides talk about gRPC internals by assuming that you are already deeply familiar with *HTTP/2.0*, and the only proper documentation for HTTP/2 is the official RFC document, which doesn't contain implementation details, and talks in abstract terms. All of this might be a bit difficult to grasp, unless you've inspected the gRPC traffic manually, at least once.

In this blog, we try to capture (log) the request and response from a gRPC client and server. This version assumes that you have access to the source code of the server.

**NOTE**: It is also possible to log the traffic without having access to the source code of the client or server. We will discuss this in future blogs.

## Start a gRPC Server

The [official documentation](https://grpc.io/docs/languages/go/quickstart/) guides you on how to create a gRPC server. The [core component](https://github.com/grpc/grpc-go/blob/d7f45cdf9ae720256d328fcdb6356ae58122a7f6/examples/helloworld/greeter_server/main.go#L50) is:

```go
func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

First, we create a listener that listens on the given TCP address using [net.Listen](https://pkg.go.dev/net#Listen). We then initialize a gRPC server using [grpc.NewServer](https://pkg.go.dev/google.golang.org/grpc@v1.56.1#Server), register the generated protos and RPC methods and finally, we start serving the requests using [server.Serve](https://pkg.go.dev/google.golang.org/grpc@v1.56.1#Server.Serve). If you look at the function arguments of the `Serve` method, you'll notice that it requires a `net.Listener`. Lucky for us, `net.Listen` already outputs a `net.Listener` which we can reuse here to run the gRPC server. But what if we also want to inspect the traffic that goes through `net.Listener` before it is consumed by the `Serve` method?

## Interfaces

One of the most important constructs in Go is an **interface**. It focuses on abstracting out the implementation details from the end user and treating certain functionalities as a black box. In short, we only focus on *what* is being done, instead of *how*.

The documentation of [net.Listener](https://pkg.go.dev/net#Listener) says that it is an interface with the following methods:

```go
type Listener interface {
	Accept() (Conn, error)
	Close() error
	Addr() Addr
}
```

Digging a bit deeper, it also says that the `Accept` method should yield a [net.Conn](https://pkg.go.dev/net#Conn), which is again an interface.

```go
type Conn interface {
	Read(b []byte) (n int, err error)
	Write(b []byte) (n int, err error)
	Close() error
	LocalAddr() Addr
	RemoteAddr() Addr
	SetDeadline(t time.Time) error
	SetReadDeadline(t time.Time) error
	SetWriteDeadline(t time.Time) error
}
```

## Fake Connection

Note that the `Serve` method accepts an interface, it doesn't care about how it is implemented. What if we could pass in a custom struct with a dummy implementation of all the methods corresponding to the interface? That works during the compilation phase but would fail to serve the actual gRPC requests as the dummy implementation no longer talks to the gRPC client using a TCP connection. How do we get the best of both worlds, i.e., we don't want to spend the effort writing a TCP implementation from scratch, but at the same time, we want to act as a middleman before sending the actual traffic to the client and server.

We can achieve this by utilizing the current implementation of `net.Conn` from the `net` package. We wrap the struct implemented in the original interface, and we shadow all the methods, which internally calls the inner struct's methods, before printing some log lines. A sample implementation might look like:

```go
type fakeConn struct {
	actualConn net.Conn
}

func (f fakeConn) Read(b []byte) (n int, err error) {
	n, err = f.actualConn.Read(b)
	fmt.Printf("Read initiated from the client with data \n%v\n\n", string(b))
	return n, err
}

func (f fakeConn) Write(b []byte) (n int, err error) {
	fmt.Printf("Writing data to client \n%v\n\n", b)
	n, err = f.actualConn.Write(b)
	return n, err
}

// Mirror other methods here...
```

## Fake Listener

Similarly, we can pass in a fake listener that wrappers our fake connection for each *Accept* call. It mirrors all other function calls of the original listener.

```go
type fakeListener struct {
	actualListener net.Listener
}

func (f fakeListener) Accept() (net.Conn, error) {
	netConn, err := f.actualListener.Accept()
	return fakeConn{actualConn: netConn}, err
}

// Mirror other methods here...
```

## Tying it All Together

Once we have the fake connection and listener setup, we can just invoke the gRPC server's `Serve` method, but with our custom listener.

```go
	if err := s.Serve(fakeListener{actualListener: lis}); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
```

## Sample Traffic

If you run the server with the custom implementation of the interfaces, you'll see these log lines. Note that the client traffic is omitted for brevity.

```go
Writing data to client 
[0 0 6 4 0 0 0 0 0 0 5 0 0 64 0]

Read initiated from the client with data ...
Read initiated from the client with data ...

Writing data to client 
[0 0 0 4 1 0 0 0 0]

Read initiated from the client with data ...
Read initiated from the client with data ...


Writing data to client 
[0 0 4 8 0 0 0 0 0 0 0 0 16 0 0 8 6 0 0 0 0 0 2 4 16 16 9 14 7 7]

Read initiated from the client with data ...

Writing data to client 
[0 0 14 1 4 0 0 0 1 136 95 139 29 117 208 98 13 38 61 76 77 101 100 0 0 22 0 0 0 0 0 1 0 0 0 0 17 10 15 72 101 108 108 111 32 103 82 80 67 45 99 97 108 108 0 0 24 1 5 0 0 0 1 64 136 154 202 200 178 18 52 218 143 1 48 64 137 154 202 200 181 37 66 7 49 127 0]

Read initiated from the client with data ...

Writing data to client 
[0 0 8 6 1 0 0 0 0 2 4 16 16 9 14 7 7]
```

## Application: Capture Redis Traffic from Client

Let's apply whatever we've learned to capture Redis traffic from a client connection. If you look at the [documentation](https://pkg.go.dev/github.com/go-redis/redis) of the Redis package, you'll notice the [NewClient](https://pkg.go.dev/github.com/go-redis/redis#NewClient) function which takes in a list of dial options and returns a client, which is a struct with hidden fields. Inspecting [redis.Options](https://pkg.go.dev/github.com/go-redis/redis#Options) reveals that it directly accepts the IP and port of the Redis server. But if you do a deep dive into all the fields in the struct, you'd notice this field

```go
	// Dialer creates new network connection and has priority over
	// Network and Addr options.
	Dialer func() (net.Conn, error)
```

This means that manual IP and port are low in priority as compared to this `Dialer`. Hence, to capture traffic, all we need to do is create a fake `net.Conn` wrapping of the original `net.Conn` from the Redis server, and create a dialer function that would act as a wrapper over this fake connection. After that, we should be able to view the traffic exchanged in the logs.

## Why Capture Traffic?

[Keploy](https://github.com/keploy/keploy) is a toolkit for generating E2E tests for APIs, along with the mocks for the dependency calls (database, Redis, gRPC, etc). However, traditionally, it followed the SDK approach, wherein each developer had to add a hook in their handlers that would let Keploy capture the traffic. This approach, although working, is not feasible for scaling across multiple languages, and different routers for the underlying APIs. It also assumes that the caller has access to the source code of the application.

To overcome this problem, we at Keploy, are working on V2 of this project, which would be language agnostic, as it captures the traffic on the TCP connection level. As part of my GSoC project, I'm working on ensuring that we correctly capture and replay gRPC traffic.

Not a lot of people (including me) have deep networking knowledge, hence, after struggling for a couple of days trying to trace the order of operations in which a client talks to the server, it finally struck me that I could replace the listeners with fake ones, it was an amazing experience. I could finally see what was going on in the system, instead of just guessing or reading stuff on the internet. As they say, you learn by doing!

## Next Steps

Now that we have the traffic from the client and server side, we still need to make sense of it. Right now, it's just a series of 8-bit numbers (or a *byte slice*). How do we make sense of these numbers, for example, how do we determine if this is a gRPC traffic? Who sends the first message, client or server? Is there a predefined order in which the clients and server communicate with each other? How are messages multiplexed? How are headers compressed? How do you interpret the wire format of the body and make it human-readable and editable?

We'll talk about some of these in the upcoming blogs. Stay tuned!