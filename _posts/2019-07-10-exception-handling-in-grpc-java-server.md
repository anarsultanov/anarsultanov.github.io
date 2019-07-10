---
layout: post
title: "Exception Handling and Error Propagation in gRPC Java"
author: "Anar Sultanov"
image: /images/posts/2019-07-10/image.jpg
---
It is quite important to propagate detailed error information from the server to the client in case something goes wrong, but the gRPC documentation lacks details on this topic.
<br>In this tutorial, we are going to look at how to handle exceptions in the gRPC Java server and provide information about them to clients.

## 1. Server and Client
To begin, we will create a simple server and client.

#### 1.1. Proto
{% highlight proto %}
message GreetingRequest {
    string name = 1;
}

message GreetingResponse {
    string greeting = 1;
}

service GreetingService {
    rpc greeting (GreetingRequest) returns (GreetingResponse);
}
{% endhighlight %}

#### 1.2. Server
{% highlight java %}
public class GreetingServer {

    public static void main(String[] args) throws IOException, InterruptedException {
        Server server = ServerBuilder
                .forPort(8080)
                .addService(new GreetingService())
                .build();

        server.start();
        System.out.println("gRPC Server started, listening on port:" + server.getPort());
        server.awaitTermination();
    }

    private static class GreetingService extends GreetingServiceGrpc.GreetingServiceImplBase {
        @Override
        public void greeting(GreetingRequest request, StreamObserver<GreetingResponse> responseObserver) {
            String name = request.getName();
            String greeting = String.format("Hello, %s!", name.isBlank() ? "World" : name);
            GreetingResponse response = GreetingResponse.newBuilder()
                    .setGreeting(greeting)
                    .build();
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        }
    }

}
{% endhighlight %}

#### 1.3. Client
{% highlight java %}
public class GreetingClient {

    public static void main(String[] args) {
        var channel = ManagedChannelBuilder.forAddress("localhost", 8080)
                .usePlaintext()
                .build();
        var stub = GreetingServiceGrpc.newBlockingStub(channel);

        GreetingRequest request = GreetingRequest.newBuilder().build();
        GreetingResponse response = stub.greeting(request);
        System.out.println(response.getGreeting());
    }
}
{% endhighlight %}

#### 1.4. Result
We created a simple server and client to it, which sends the request and outputs the response to the console. 
<br>**Note:** on the server, we process requests with a missing name and use `World` in this case, 
and since our client does not include any name in the request, we will see `Hello, World!` in the output.

## 2. Exception Handling
Let's move on to the main issue, exception handling.

#### 2.1 Throw Exception
Now we do not want to accept requests with a missing name, so we update our server to throw exception in this case.
We replace the line:
{% highlight java %}
String greeting = String.format("Hello, %s!", name.isBlank() ? "World" : name);
{% endhighlight %}
With:
{% highlight java %}
if (name.isBlank()) {
    throw new IllegalArgumentException("Missing name");
}
String greeting = String.format("Hello, %s!", name);
{% endhighlight %}

If we start our server and client, we will see an exception in the server console:
<br>`java.lang.IllegalArgumentException: Missing name`

We will also see an exception in the client console, but there is no information about what went wrong:
<br>`Exception in thread "main" io.grpc.StatusRuntimeException: UNKNOWN`

#### 2.2 Handle Exception
Our goal is to provide more detailed information about exceptions on the server to the client. 
To do this, we create an interceptor that will catch exceptions and handle them:
{% highlight java %}
public class ExceptionHandler implements ServerInterceptor {

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(ServerCall<ReqT, RespT> serverCall, Metadata metadata,
                                                                 ServerCallHandler<ReqT, RespT> serverCallHandler) {
        ServerCall.Listener<ReqT> listener = serverCallHandler.startCall(serverCall, metadata);
        return new ExceptionHandlingServerCallListener<>(listener, serverCall, metadata);
    }

    private class ExceptionHandlingServerCallListener<ReqT, RespT>
            extends ForwardingServerCallListener.SimpleForwardingServerCallListener<ReqT> {
        private ServerCall<ReqT, RespT> serverCall;
        private Metadata metadata;

        ExceptionHandlingServerCallListener(ServerCall.Listener<ReqT> listener, ServerCall<ReqT, RespT> serverCall,
                                            Metadata metadata) {
            super(listener);
            this.serverCall = serverCall;
            this.metadata = metadata;
        }

        @Override
        public void onHalfClose() {
            try {
                super.onHalfClose();
            } catch (RuntimeException ex) {
                handleException(ex, serverCall, metadata);
                throw ex;
            }
        }

        @Override
        public void onReady() {
            try {
                super.onReady();
            } catch (RuntimeException ex) {
                handleException(ex, serverCall, metadata);
                throw ex;
            }
        }

        private void handleException(RuntimeException exception, ServerCall<ReqT, RespT> serverCall, Metadata metadata) {
            if (exception instanceof IllegalArgumentException) {
                serverCall.close(Status.INVALID_ARGUMENT.withDescription(exception.getMessage()), metadata);
            } else {
                serverCall.close(Status.UNKNOWN, metadata);
            }
        }
    }
}
{% endhighlight %}
**Note:** in the interceptor, we use a private class that inherits `SimpleForwardingServerCallListener`, 
and overrides `onHalfClose` and `onReady` methods to handle exceptions.
There is no point to override `onCancel` and `onComplete`, since it is already too late.

Next we need to add an interceptor to the server:
{% highlight java %}
 Server server = ServerBuilder
                // ...
                .intercept(new ExceptionHandler())
                .build();
{% endhighlight %}

Now, when we start the server and client, we will see a more informative exception in the client console:
<br>`Exception in thread "main" io.grpc.StatusRuntimeException: INVALID_ARGUMENT: Missing name`

## 3. Conclusion
In our example, we handle only `IllegalArgumentException`, but you can handle any other exceptions in the same way 
and also include much more information in the response besides the exception message.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/grpc-java-exception-handling)._
