---
layout: post
title: "Client-Side Load Balancing in gRPC Java"
author: "Anar Sultanov"
image: /images/posts/2019-11-24/image.jpg
---
Load balancing is the process of distributing network traffic between multiple servers, 
used to improve the performance and reliability of websites, applications, databases and other services.
Using a centralized load balancer is the most traditional approach for this, but client-side load balancing still has some advantages and is also quite common.
<br>In this tutorial, we are going to look at how to implement client-side load balancing in gRPC Java client.

## 1. Multiple server instances
We will start by creating multiple server instances serving the same gRPC service, which we will use to demonstrate load balancing.

#### 1.1. Service
First we define simple service in a proto file:
{% highlight proto %}
syntax = "proto3";
option java_multiple_files = true;
package dev.sultanov.grpc.loadbalancing;

message EchoRequest {
    string message = 1;
}

message EchoResponse {
    string message = 1;
}

service EchoService {
    rpc echo (EchoRequest) returns (EchoResponse);
}
{% endhighlight %}

Then we implement this service so that it returns the server name passed to it in the constructor and the message received:
{% highlight java %}
class EchoService extends EchoServiceGrpc.EchoServiceImplBase {

    private final String name;

    EchoService(String name) {
        this.name = name;
    }

    @Override
    public void echo(EchoRequest request, StreamObserver<EchoResponse> responseObserver) {
        String message = request.getMessage();
        String reply = this.name + " echo: " + message;
        EchoResponse response = EchoResponse.newBuilder()
                .setMessage(reply)
                .build();
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
{% endhighlight %}

#### 1.2. Servers
To simplify the example, we will start all servers from one main method in separate threads:
{% highlight java %}
public static void main(String[] args) {
    final int nServers = 3;
    ExecutorService executorService = Executors.newFixedThreadPool(nServers);
    for (int i = 0; i < nServers; i++) {
        String name = "Server_" + i;
        int port = 50000 + i;
        executorService.submit(() -> {
            try {
                startServer(name, port);
            } catch (IOException | InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
}

private static void startServer(String name, int port) throws IOException, InterruptedException {
    Server server = ServerBuilder
            .forPort(port)
            .addService(new EchoService(name))
            .build();

    server.start();
    System.out.println(name + " server started, listening on port: " + server.getPort());
    server.awaitTermination();
}
{% endhighlight %}

Running the main method, we will see that the servers are up and ready to receive requests:
{% highlight console %}
Server_0 server started, listening on port: 50000
Server_1 server started, listening on port: 50001
Server_2 server started, listening on port: 50002
{% endhighlight %}

## 2. Client
The two main components responsible for client-side request distribution are the name resolver and the load balancing policy.

#### 2.1. Name Resolver
Before starting interaction with the server, the client needs to obtain all its IP addresses using a name resolver.
By default, gRPC uses DNS as its name-system, i.e. each A record of the DNS entry will be used as a possible endpoint.
But in case we want to use a service registry, such as Eureka or Consul, or specify several addresses ourselves, we will need to implement a custom name resolver.

In this example, we create a simple name resolver that will accept multiple addresses as an argument:
{% highlight java %}
class MultiAddressNameResolverFactory extends NameResolver.Factory {

    final List<EquivalentAddressGroup> addresses;

    MultiAddressNameResolverFactory(SocketAddress... addresses) {
        this.addresses = Arrays.stream(addresses)
                .map(EquivalentAddressGroup::new)
                .collect(Collectors.toList());
    }

    public NameResolver newNameResolver(URI notUsedUri, NameResolver.Args args) {
        return new NameResolver() {
            @Override
            public String getServiceAuthority() {
                return "fakeAuthority";
            }
            public void start(Listener2 listener) {
                listener.onResult(ResolutionResult.newBuilder().setAddresses(addresses).setAttributes(Attributes.EMPTY).build());
            }
            public void shutdown() {
            }
        };
    }

    @Override
    public String getDefaultScheme() {
        return "multiaddress";
    }
}
{% endhighlight %}

#### 2.2. Load Balancing
In the case of the load balancing policy, we do not need to implement anything ourselves, 
we can choose one of the policies presented in the library, e.g. `grpclb`, `pick_first` or `round_robin`.
Although the first two policies do not suit us, a Round Robin will work just fine.

Let's create an instance of the name resolver with the addresses of previously started servers 
and use it along with the chosen load balancing policy to create a client connection channel:
{% highlight java %}
public static void main(String[] args) {
    NameResolver.Factory nameResolverFactory = new MultiAddressNameResolverFactory(
            new InetSocketAddress("localhost", 50000),
            new InetSocketAddress("localhost", 50001),
            new InetSocketAddress("localhost", 50002)
    );

    ManagedChannel channel = ManagedChannelBuilder.forTarget("service")
            .nameResolverFactory(nameResolverFactory)
            .defaultLoadBalancingPolicy("round_robin")
            .usePlaintext()
            .build();
}
{% endhighlight %}

Finally, we use the channel to create a stub and make several calls, the result of which we output to the console:
{% highlight java %}
public static void main(String[] args) {

    // ...
    
    EchoServiceGrpc.EchoServiceBlockingStub stub = EchoServiceGrpc.newBlockingStub(channel);
    for (int i = 0; i < 10; i++) {
        EchoResponse response = stub.echo(EchoRequest.newBuilder()
                .setMessage("Hello!")
                .build());
        System.out.println(response.getMessage());
    }
}
{% endhighlight %}

We can see that responses are coming back from different servers:
{% highlight console %}
Server_0 echo: Hello!
Server_1 echo: Hello!
Server_2 echo: Hello!
Server_0 echo: Hello!
...
{% endhighlight %}

## 3. Conclusion
We looked at how to enable client-side load balancing using our custom name resolver and round-robin algorithm.
Although this does not require writing a lot of code, it may take some time to figure out how the different parts fit together.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/grpc-client-side-load-balancing)._
