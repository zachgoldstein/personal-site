---
title: "Simple, minimal, production-ready RPC in golang with twirp"
date: 2018-06-12T13:11:29-04:00
draft: false
type: "post"
---

The more code you write, the greater the surface for potential bugs. In traditional REST APIs, you often get stuck writing lots of routing and serialization code, which can often get out of sync as your application grows. RPC (Remote Procedure Call) libraries come to the rescue here, generating a lot of this client & server code for you so you can focus on your implementations.

Previously, RPC usage in golang was centered around a couple larger libraries, gokit and grpc. As a counter to this, some glorious Twitch engineers have recently open-sourced a much more minimal library called Twirp. In this post, we’re going to go through a quick example of how to use this library in go, along with a brief description of when you might want to use it over the heavier options. For a more in-depth post on the library, check out the release blog post here: https://blog.twitch.tv/twirp-a-sweet-new-rpc-framework-for-go-5f2febbf35f

So! Off we go to the world of simplistic examples.

In twirp, we start our journey with a protocol buffer (protobuf) file. This outlines the service endpoints and request response objects in a language-agnostic way.

Fist start by installing twirp and protobuf:

{{< highlight bash "" >}}
get github.com/twitchtv/twirp/protoc-gen-twirp
go get github.com/golang/protobuf/protoc-gen-go
{{< / highlight >}}

Inspired by the existential dread of universal paperclips (http://www.decisionproblem.com/paperclips/index2.html), we’ll make a simple protobuf definition for a service that does three silly things:

- Get the current number of paperclips
- Increment the number of paperclips
- Calculate how close we are to the death of the universe.
- Here’s what that looks like in protobuf:


We’ll save this to rpc/service.proto. And now we can generate our client/server code:

{{< highlight protobuf "" >}}
syntax = "proto3";

package twirp.paperclips;
option go_package = "rpc";

// UniversalPaperclips service creates paperclips and calculates existential dread.
service UniversalPaperclips {
  rpc GetPaperclips(Empty) returns (Paperclips);
  rpc IncrementPaperclips(Size) returns (Empty);
  rpc CalculateUniverseLifespan(Empty) returns (Dread);
}

message Size {
  int32 paperclips = 1; // must be > 0
}

message Paperclips {
  int32 paperclips = 1;
}

message Dread {
  int32 paperclips = 1;
  string universeLifespan = 2;
}

message Empty {}
{{< / highlight >}}

This will create two files:


- rpc/service.pb.go (Containing our message definitions)
- rpc/service.twirp.go (Containing our service functions and interfaces)

We’ll now create our server implementation at /paperclipserver/server.go:

{{< highlight golang "" >}}
paperclipserver

import (
    "context"

    pb "github.com/zachgoldstein/twirpPaperclips/rpc"
)

// Server implements the UniversalPaperclips service
type Server struct {
    PaperClips int32
}

// NewServer creates an instance of our server
func NewServer() *Server {
    return &Server{
        PaperClips: 1,
    }
}

// GetPaperclips returns the current paperclip count
func (s *Server) GetPaperclips(ctx context.Context, empty *pb.Empty) (*pb.Paperclips, error) {
    return &pb.Paperclips{
        Paperclips: s.PaperClips,
    }, nil
}

// IncrementPaperclips increments the paperclip count
func (s *Server) IncrementPaperclips(ctx context.Context, size *pb.Size) (*pb.Empty, error) {
    s.PaperClips += size.Paperclips
    return &pb.Empty{}, nil
}

// CalculateUniverseLifespan calulcates the universe's lifespan and returns some marginally relaxing value
func (s *Server) CalculateUniverseLifespan(ctx context.Context, empty *pb.Empty) (*pb.Dread, error) {
    return &pb.Dread{
        Paperclips:       s.PaperClips,
        UniverseLifespan: "42",
    }, nil
{{< / highlight >}}

So the benefits here are pretty awesome. We didn’t write a single line of serialization code and had a nice, clear interface that was generated for us to code against. Our model types (called messages) were also generated, and we didn’t need to bother pulling the object bodies out of response objects. If we miss a field or forget to implement some function, we’ll see failures at compile time that will prevent incorrect code from getting into a binary.

So pretty cool right? Usage of the server looks quite clean as well. Our main.go file is:

{{< highlight golang "" >}}
main

import (
    "fmt"
    "log"
    "net/http"

    "github.com/zachgoldstein/twirpPaperclips/paperclipserver"
    "github.com/zachgoldstein/twirpPaperclips/rpc"
)

func main() {
    fmt.Printf("Starting Universal Paperclip Service on :6666")

    server := paperclipserver.NewServer()
    twirpHandler := rpc.NewUniversalPaperclipsServer(server, nil)

    log.Fatal(http.ListenAndServe(":6666", twirpHandler))
}
{{< / highlight >}}

We can now run our twirp RPC service with: 
{{< highlight bash "" >}}
go run main.go
{{< / highlight >}}

And we can hit it with curl:

{{< highlight bash "" >}}
--request "POST" \
     --location "http://localhost:6666/twirp/twirp.paperclips.UniversalPaperclips/GetPaperclips" \
     --header "Content-Type:application/json" \
     --data '{}' \
     --verbose
{{< / highlight >}}

Which returns:
{{< highlight bash "" >}}
{"paperclips":6}
{{< / highlight >}}

Instead of REST GET/POST/etc. semantics, every twirp request is POST. You also have the opportunity to take advantage of protobuf’s binary serialization instead of JSON.

If you want to issue a request to this service, you also have generated client code you can take advantage of:

{{< highlight golang "" >}}
main

import (
    "context"
    "fmt"
    "net/http"

    "github.com/zachgoldstein/twirpPaperclips/rpc"
)

func main() {
    fmt.Println("Client Example for Universal Paperclip Service")

    client := rpc.NewUniversalPaperclipsProtobufClient("http://localhost:6666", &http.Client{})
    _, err := client.IncrementPaperclips(context.Background(), &rpc.Size{Paperclips: 5})
    if err != nil {
        fmt.Println(err.Error())
    }
    paperclips, err := client.GetPaperclips(context.Background(), &rpc.Empty{})
    if err != nil {
        fmt.Println(err.Error())
    }
    fmt.Println("Paperclips: ", paperclips.String())

    dread, err := client.CalculateUniverseLifespan(context.Background(), &rpc.Empty{})
    if err != nil {
        fmt.Println(err.Error())
    }
    fmt.Println("Dread: ", dread.String())
}
{{< / highlight >}}

Running this outputs:

{{< highlight bash "" >}}
Example for Universal Paperclip Service
Paperclips:  paperclips:6
Dread:  paperclips:6 universeLifespan:"42"
{{< / highlight >}}


You can see this full example online at https://github.com/zachgoldstein/twirpPaperclips.

The question still remains — why use this library when we have pretty fantastic, Google-supported projects like grpc or gokit?

To sum it up pretty succinctly, gRPC is a much heavier library, and goKit doesn’t really include enough to do the client/server code generation we see here. Twirp is ideal for use cases where you want to minimize new dependencies and sync possibly out-of-date client/server code.

gRPC requires you use http/2, which might not be feasible for you depending on your use case. It also has a heavy runtime that implements its own version of the built-in http library. This allows it to do some pretty cool stuff, but it also means you need to keep that library updated and in sync with the generated client/server code. Twirp has chosen to not do this, and instead generates heavier client/server code that’s less likely to encounter breaking changes when it’s much older than the latest version of the library itself. If your use case allows you to control many of the services implementing RPC in your system, then syncing issues might not be that big of a deal, and gRPC is worth a good look.

goKit, on the other hand, is not all that opinionated about how you do your RPC. It outlines an approach to building RPC services and includes a bunch of great tools for logging and instrumentation, but it stops short of enforcing protobuf usage and generated code. It’s pretty much as it sounds — more of a “kit” than an RPC-specific library. If you’re wary of generated code and want to roll many of the RPC mechanisms yourself, goKit gives you some great patterns to help structure those efforts.

I hope this post helps you in your journey towards service-oriented nirvana. Thanks for reading and thanks to the team at Twitch for releasing this excellent library!

---

Originally published at x-team.com on February 5, 2018