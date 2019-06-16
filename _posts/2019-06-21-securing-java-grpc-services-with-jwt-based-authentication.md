---
layout: post
title: "Securing Java gRPC services with JWT-based authentication"
author: "Anar Sultanov"
image: /images/posts/2019-06-21/image.jpg
---
[gRPC](https://grpc.io/) is an open source, high-performance RPC framework that has several advantages to be used for communication between services, 
but unfortunately, in addition to SSL/TLS support, the only authentication mechanism built-in to gRPC is token-based authentication for use with Google services.
<br>In this tutorial, we are going to create a gRPC server and secure it with JWT-based authentication.

## 1. Maven Dependencies
Our first step is to create a Maven project and add the necessary dependencies:
```
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-netty-shaded</artifactId>
  <version>1.21.0</version>
</dependency>
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-protobuf</artifactId>
  <version>1.21.0</version>
</dependency>
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-stub</artifactId>
  <version>1.21.0</version>
</dependency>
```

## 2. gPRC service
Next we need to define the gRPC service using [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview) 
and generate interfaces to be used by the gPRC server and client.

#### 2.1. Service definition
Let's create a proto file and define a simple service: 
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

#### 2.2. Code generation
For protobuf-based codegen integrated with the Maven build system, we need to put our proto file in the `src/main/proto` directory 
and use `protobuf-maven-plugin`:
```
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.6.2</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.6.1:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.19.0:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

As soon as everything is set up, we can execute the following goal to generate code:
```
mvn compile
```

## 3. gPRC server
There are two parts to get our server to do its job.

#### 3.1. Service implementation
We need to extend the generated service class and override the method so that it performs the actual work:
{% highlight java %}
public class GreetingServer {
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
We made the service a private inner class only to reduce the number of classes, but we could have it as a separate class.

#### 3.2. Server startup
Then we need to run a gRPC server to listen for requests from clients and return the service responses:
{% highlight java %}
public class GreetingServer {

    // ...
    
    public static void main(String[] args) throws IOException, InterruptedException {
        Server server = ServerBuilder
                .forPort(8080)
                .addService(new GreetingService())
                .build();

        server.start();
        System.out.println("gRPC Server started, listening on port:" + server.getPort());
        server.awaitTermination();
    }
}
{% endhighlight %}

## 4. gPRC client
We can start our server, but the only way to call a remote procedure is to use the generated gRPC client.
Let's create a client that will make a request to our server at startup and display the greeting received in the response in the console:
{% highlight java %}
public class GreetingClient {

    public static void main(String[] args) {
        var channel = ManagedChannelBuilder.forAddress("localhost", 8080)
                .usePlaintext()
                .build();
        var stub = GreetingServiceGrpc.newBlockingStub(channel);
        GreetingRequest request = GreetingRequest.newBuilder().setName("John").build();
        GreetingResponse response = stub.greeting(request);
        System.out.println(response.getGreeting());
    }
}
{% endhighlight %}

Having started the client, we will see in the console a greeting with the specified name received from the server:
{% highlight console %}
Hello, John!
{% endhighlight %}

## 5. Authorization
To create and verify JSON Web Tokens (JWTs), we add the [JJWT](https://github.com/jwtk/jjwt) library to our dependencies:
```
 <dependency>
     <groupId>io.jsonwebtoken</groupId>
     <artifactId>jjwt</artifactId>
     <version>0.9.1</version>
 </dependency>
```

We also create a class for constants, some of which will be used by both the server and the client.
{% highlight java %}
public class Constants {
    public static final String JWT_SIGNING_KEY = "L8hHXsaQOUjk5rg7XPGv4eL36anlCrkMz8CJ0i/8E/0=";
    public static final String BEARER_TYPE = "Bearer";

    public static final Metadata.Key<String> AUTHORIZATION_METADATA_KEY = Metadata.Key.of("Authorization", ASCII_STRING_MARSHALLER);
    public static final Context.Key<String> CLIENT_ID_CONTEXT_KEY = Context.key("clientId");

    private Constants() {
        throw new AssertionError();
    }
}
{% endhighlight %}

#### 5.1. Server security
To protect the server, we need to implement an `ServerInterceptor`, which will get the authorization token from the metadata, 
verify it and set the client identifier obtained from the claims into the context. 
In order not to complicate the code with additional checks (expiration date, issuer and etc.), we will only rely on the signature of the token:
{% highlight java %}
public class AuthorizationServerInterceptor implements ServerInterceptor {

    private JwtParser parser = Jwts.parser().setSigningKey(Constants.JWT_SIGNING_KEY);

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(ServerCall<ReqT, RespT> serverCall, Metadata metadata, ServerCallHandler<ReqT, RespT> serverCallHandler) {
        String value = metadata.get(Constants.AUTHORIZATION_METADATA_KEY);

        Status status;
        if (value == null) {
            status = Status.UNAUTHENTICATED.withDescription("Authorization token is missing");
        } else if (!value.startsWith(Constants.BEARER_TYPE)) {
            status = Status.UNAUTHENTICATED.withDescription("Unknown authorization type");
        } else {
            try {
                String token = value.substring(Constants.BEARER_TYPE.length()).trim();
                Jws<Claims> claims = parser.parseClaimsJws(token);
                Context ctx = Context.current().withValue(Constants.CLIENT_ID_CONTEXT_KEY, claims.getBody().getSubject());
                return Contexts.interceptCall(ctx, serverCall, metadata, serverCallHandler);
            } catch (Exception e) {
                status = Status.UNAUTHENTICATED.withDescription(e.getMessage()).withCause(e);
            }
        }

        serverCall.close(status, metadata);
        return new ServerCall.Listener<>() {
            // noop
        };
    }
}
{% endhighlight %}

We also will not introduce any restrictions on the client's identifier, 
but simply make our service output it to the console when processing the request:
{% highlight java %}
public void greeting(GreetingRequest request, StreamObserver<GreetingResponse> responseObserver) {
    String clientId = Constants.CLIENT_ID_CONTEXT_KEY.get();
    System.out.println("Processing request from " + clientId);
    // ...
}
{% endhighlight %}

Now, if we restart the server and run our client, we will see an exception in the console with a message about the missing token:
{% highlight console %}
Exception in thread "main" io.grpc.StatusRuntimeException: UNAUTHENTICATED: Authorization token is missing
{% endhighlight %}

#### 5.2. Client authorization
To authorize our client, we need to send a valid token along with our request.
All gRPC stubs inherit the `withCallCredentials(CallCredentials credentials)` method which returns a new stub that uses the given call credentials,
but to use this method we need to implement `CallCredentials` that will contain the token in the required format:
{% highlight java %}
public class BearerToken extends CallCredentials {

    private String value;

    public BearerToken(String value) {
        this.value = value;
    }

    @Override
    public void applyRequestMetadata(RequestInfo requestInfo, Executor executor, MetadataApplier metadataApplier) {
        executor.execute(() -> {
            try {
                Metadata headers = new Metadata();
                headers.put(Constants.AUTHORIZATION_METADATA_KEY, String.format("%s %s", Constants.BEARER_TYPE, value));
                metadataApplier.apply(headers);
            } catch (Throwable e) {
                metadataApplier.fail(Status.UNAUTHENTICATED.withCause(e));
            }
        });
    }

    @Override
    public void thisUsesUnstableApi() {
        // noop
    }
}
{% endhighlight %}

Then we add a method to our client to generate a valid token:
{% highlight java %}
private static String getJwt() {
    return Jwts.builder()
            .setSubject("GreetingClient") // client's identifier
            .signWith(SignatureAlgorithm.HS256, Constants.JWT_SIGNING_KEY)
            .compact();
}
{% endhighlight %}

Next, we just need to create an instance of the `BearerToken` from the generated JWT
and make a stub in the main method of our client to use it:
{% highlight java %}    
public static void main(String[] args) {
    // ...
    String jwt = getJwt();
    BearerToken token = new BearerToken(getJwt());
    var stub = GreetingServiceGrpc.newBlockingStub(channel)
            .withCallCredentials(token);
    // ...
}
{% endhighlight %}

Now, if we start the client, we should get the expected greeting in the client console 
and the message with the client's identifier in the service console:
{% highlight console %}
Processing request from GreetingClient
{% endhighlight %}

## 6. Conclusion
We created a simple gRPC server and protected it with JWT based authentication. 
We also created a client for this server and implemented its authorization.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/grpc-java-authentication)._
