# Create a gRPC enabled app, and invoke Dapr over gRPC

Dapr implements both an HTTP and a gRPC API for local calls.gRPC is useful for low-latency, high performance scenarios and has language integration using the proto clients.

You can find a list of auto-generated clients [here](https://github.com/dapr/docs#sdks).

The Dapr runtime implements a [proto service](https://github.com/dapr/dapr/blob/master/dapr/proto/runtime/v1/dapr.proto) that apps can communicate with via gRPC.

In addition to calling Dapr via gRPC, Dapr can communicate with an application via gRPC. To do that, the app needs to host a gRPC server and implements the [Dapr appcallback service](https://github.com/dapr/dapr/blob/master/dapr/proto/runtime/v1/appcallback.proto)

## Configuring Dapr to communicate with an app via gRPC

### Self hosted

When running in self hosted mode, use the `--app-protocol` flag to tell Dapr to use gRPC to talk to the app:

```bash
dapr run --app-protocol grpc --app-port 5005 node app.js
```
This tells Dapr to communicate with your app via gRPC over port `5005`.


### Kubernetes

On Kubernetes, set the following annotations in your deployment YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "myapp"
        dapr.io/app-protocol: "grpc"
        dapr.io/app-port: "5005"
...
```

## Invoking Dapr with gRPC - Go example

The following steps show you how to create a Dapr client and call the `SaveStateData` operation on it:

1. Import the package

```go
package main

import (
	"context"
	"log"
	"os"

	dapr "github.com/dapr/go-sdk/client"
)
```

2. Create the client

```go
// just for this demo
ctx := context.Background()
data := []byte("ping")
  
// create the client
client, err := dapr.NewClient()
if err != nil {
  logger.Panic(err)
}
defer client.Close()
```

3. Invoke the Save State method

```go
// save state with the key key1
err = client.SaveStateData(ctx, "statestore", "key1", "1", data)
if err != nil {
  logger.Panic(err)
}
logger.Println("data saved")
```

Hooray!

Now you can explore all the different methods on the Dapr client.

## Creating a gRPC app with Dapr

The following steps will show you how to create an app that exposes a server for Dapr to communicate with.

1. Import the package

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net"

	"github.com/golang/protobuf/ptypes/any"
	"github.com/golang/protobuf/ptypes/empty"

	commonv1pb "github.com/dapr/go-sdk/dapr/proto/common/v1"
	pb "github.com/dapr/go-sdk/dapr/proto/runtime/v1"
	"google.golang.org/grpc"
)
```

2. Implement the interface

```go
// server is our user app
type server struct {
}

// EchoMethod is a simple demo method to invoke
func (s *server) EchoMethod() string {
	return "pong"
}

// This method gets invoked when a remote service has called the app through Dapr
// The payload carries a Method to identify the method, a set of metadata properties and an optional payload
func (s *server) OnInvoke(ctx context.Context, in *commonv1pb.InvokeRequest) (*commonv1pb.InvokeResponse, error) {
	var response string

	switch in.Method {
	case "EchoMethod":
		response = s.EchoMethod()
	}

	return &commonv1pb.InvokeResponse{
		ContentType: "text/plain; charset=UTF-8",
		Data:        &any.Any{Value: []byte(response)},
	}, nil
}

// Dapr will call this method to get the list of topics the app wants to subscribe to. In this example, we are telling Dapr
// To subscribe to a topic named TopicA
func (s *server) ListTopicSubscriptions(ctx context.Context, in *empty.Empty) (*pb.ListTopicSubscriptionsResponse, error) {
	return &pb.ListTopicSubscriptionsResponse{
		Subscriptions: []*pb.TopicSubscription{
			{Topic: "TopicA"},
		},
	}, nil
}

// Dapr will call this method to get the list of bindings the app will get invoked by. In this example, we are telling Dapr
// To invoke our app with a binding named storage
func (s *server) ListInputBindings(ctx context.Context, in *empty.Empty) (*pb.ListInputBindingsResponse, error) {
	return &pb.ListInputBindingsResponse{
		Bindings: []string{"storage"},
	}, nil
}

// This method gets invoked every time a new event is fired from a registerd binding. The message carries the binding name, a payload and optional metadata
func (s *server) OnBindingEvent(ctx context.Context, in *pb.BindingEventRequest) (*pb.BindingEventResponse, error) {
	fmt.Println("Invoked from binding")
	return &pb.BindingEventResponse{}, nil
}

// This method is fired whenever a message has been published to a topic that has been subscribed. Dapr sends published messages in a CloudEvents 0.3 envelope.
func (s *server) OnTopicEvent(ctx context.Context, in *pb.TopicEventRequest) (*empty.Empty, error) {
	fmt.Println("Topic message arrived")
	return &empty.Empty{}, nil
}

```

3. Create the server

```go
func main() {
	// create listener
	lis, err := net.Listen("tcp", ":50001")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	// create grpc server
	s := grpc.NewServer()
	pb.RegisterAppCallbackServer(s, &server{})

	fmt.Println("Client starting...")

	// and start...
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

This creates a gRPC server for your app on port 4000.

4. Run your app

To run locally, use the Dapr CLI:

```
dapr run --app-id goapp --app-port 4000 --app-protocol grpc go run main.go
```

On Kubernetes, set the required `dapr.io/app-protocol: "grpc"` and `dapr.io/app-port: "4000` annotations in your pod spec template as mentioned above.

## Other languages

You can use Dapr with any language supported by Protobuf, and not just with the currently available generated SDKs.
Using the [protoc](https://developers.google.com/protocol-buffers/docs/downloads) tool you can generate the Dapr clients for other languages like Ruby, C++, Rust and others.

 ## Related Topics
*  [Service invocation concepts](../../concepts/service-invocation/README.md)
* [Service invocation API specification](../../reference/api/service_invocation_api.md)