---
layout: post
title: "Streaming Integration: gRPC and Reactive"
author: "Anar Sultanov"
image: /images/posts/2023-07-16/image.png
---
In the world of service communication, gRPC serves as a common choice for internal interactions, while REST and GraphQL act as the go-to options for exposing external APIs.
When these technologies are combined, there arises a need to seamlessly map streaming RPC to reactive stream publishers, which are integral components of reactive REST endpoints and GraphQL subscriptions.
<br>In this tutorial, we will explore the implementation of this mapping, bridging the gap between gRPC streaming and reactive streams.

In [a previous publication](https://sultanov.dev/blog/grpc-long-lived-streaming-using-observer-pattern/), we implemented a gRPC service capable of streaming data. 
In this article, we will reuse this example as a foundation and focus on creating a client for the service, that will expose the streamed data through a reactive endpoint using Spring WebFlux.

## 1. Creating Application
We start by creating a Spring Boot application in which we expose one endpoint for retrieving stock prices. 
At this stage, it will return an empty `Flux`. However, we will soon integrate it with gRPC streaming to populate the stock prices dynamically:
{% highlight java %}
@SpringBootApplication
public class StockApplication {

    public static void main(String[] args) {
        SpringApplication.run(StockApplication.class, args);
    }

    @Bean
    RouterFunction<ServerResponse> getStockPrice() {
        return route(GET("/stocks/{symbol}"),
                req -> ServerResponse.ok().body(getStockPrice(req.pathVariable("symbol")), StockPrice.class));
    }

    private Flux<StockPrice> getStockPrice(String symbol) {
        return Flux.empty();
    }

    private record StockPrice(String symbol, double price) {

    }
}
{% endhighlight %}

Now we need to set up a connection with our server. While we could utilize the convenience of the gRPC Spring Boot starter, for simplicity, we will configure it manually:
{% highlight java %}
public class StockApplication {

    private final ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 8080)
            .usePlaintext()
            .build();
    private final StockServiceGrpc.StockServiceStub stub = StockServiceGrpc.newStub(channel);

    // ...
}
{% endhighlight %}
With this setup, we are ready to proceed with integrating the gRPC streaming functionality into our application.

## 2. Integration of gRPC Streaming with Flux
To facilitate the mapping of the gRPC stream to a reactive stream publisher, we will implement a custom gRPC `StreamObserver`.

This observer will leverage a construct called `Sink`, which allows us to programmatically push Reactive Streams signals.
By using `Sinks.many().unicast().onBackpressureBuffer()`, we create a `Sink` object that will broadcast multiple signals to a single subscriber, and buffer emitted values when subscriber is not ready to receive them, preventing data loss.

While we could also extend `Flux` or implement `Publisher` to use the same object in the reactive endpoint, we will instead expose the `Sink` as a `Flux` through a method.

Additionally, we need to implement the mapping from the proto message used in gRPC to a DTO class used in our API, 
but since we may want to reuse this observer in multiple places, we will keep it abstract, with an abstract method responsible for the mapping:
{% highlight java %}
public abstract class ReactiveStreamObserver<T, S> implements StreamObserver<S> {

    private final Many<T> sink;

    protected ReactiveStreamObserver() {
        this.sink = Sinks.many().unicast().onBackpressureBuffer();
    }

    @Override
    public void onNext(S value) {
        sink.tryEmitNext(process(value));
    }

    @Override
    public void onError(Throwable t) {
        sink.tryEmitError(t);
    }

    @Override
    public void onCompleted() {
        sink.tryEmitComplete();
    }

    public Flux<T> getFlux() {
        return sink.asFlux();
    }

    public abstract T process(S value);

}
{% endhighlight %}

With the observer in place, we can now proceed to utilize it in the mapping of the gRPC stream to the reactive stream publisher. 
We will declare and initialize an instance of the observer specific to our use case using an anonymous class. 
Then, we will use this observer in a call to the gRPC service and return the `Flux` obtained from it:
{% highlight java %}
public class StockApplication {

    // ...

    private Flux<StockPrice> getStockPrice(String symbol) {
        var observer = new ReactiveStreamObserver<StockPrice, StockPriceResponse>() {
            @Override
            public StockPrice process(StockPriceResponse value) {
                return new StockPrice(value.getSymbol(), value.getPrice());
            }
        };
        StockPriceRequest request = StockPriceRequest.newBuilder().setSymbol(symbol).build();
        stub.getPrice(request, observer);
        return observer.getFlux();
    }
}
{% endhighlight %}

If we start both the gRPC server and our Spring application and navigate to [http://localhost:8081/stocks/AMZN](http://localhost:8081/stocks/AMZN) in the browser, we will start receiving updated prices for the specified stock.

However, there is an issue with the current implementation. When we close the browser, the gRPC call is not canceled, and the server continues publishing data to it, which can be observed in the server logs. 
To address this issue, we can use `CancellableContext` to cancel the call from the client-side, as discussed in the previous article about long-lived streaming. 
We can implement this cancellation in a callback on `Flux` termination:
{% highlight java %}
public class StockApplication {

    // ...

    private Flux<StockPrice> getStockPrice(String symbol) {
        var observer = new ReactiveStreamObserver<StockPrice, StockPriceResponse>() {
            @Override
            public StockPrice process(StockPriceResponse value) {
                return new StockPrice(value.getSymbol(), value.getPrice());
            }
        };
        StockPriceRequest request = StockPriceRequest.newBuilder().setSymbol(symbol).build();

        var context = Context.current().fork().withCancellation();
        context.run(() -> stub.getPrice(request, observer));
        return observer.getFlux().doFinally(signalType -> context.cancel(new RuntimeException("Context closed by " + signalType.name())));
    }
}
{% endhighlight %}

Now everything works fine, and the gRPC call is canceled when needed. 

## 3. Encapsulating Cancellation Logic
The only issue is that if we have multiple streaming calls, we would need to repeat this cancellation logic for each one. 
To simplify our code and avoid duplication, we can encapsulate this cancellation logic into our custom observer, by introducing an `observe` method.
This method will accept the request and a `BiConsumer` representing the RPC we want to execute.
The `observe` method will be responsible for creating a cancellable context, executing the call within that context, and returning a `Flux` with a registered callback for canceling the call:
{% highlight java %}
public abstract class ReactiveStreamObserver<T, S> implements StreamObserver<S> {

    // ...

    public <R> Flux<T> observe(R request, BiConsumer<R, StreamObserver<S>> consumer) {
        var context = Context.current().fork().withCancellation();
        context.run(() -> consumer.accept(request, this));
        return sink.asFlux().doFinally(signalType -> context.cancel(new RuntimeException("Context closed by " + signalType.name())));
    }
}
{% endhighlight %}

We can now utilize the `observe` in all streaming calls, making our code more concise and reusable:
{% highlight java %}
public class StockApplication {

    // ...

    private Flux<StockPrice> getStockPrice(String symbol) {
        var observer = new ReactiveStreamObserver<StockPrice, StockPriceResponse>() {
            @Override
            public StockPrice process(StockPriceResponse value) {
                return new StockPrice(value.getSymbol(), value.getPrice());
            }
        };
        StockPriceRequest request = StockPriceRequest.newBuilder().setSymbol(symbol).build();

        return observer.observe(request, stub::getPrice);
    }
}
{% endhighlight %}

## 4. Conclusion
In this article, we explored the mapping of gRPC streams to an external API that uses reactive streams. 
By leveraging a custom observer and incorporating cancellation logic, we were able to seamlessly integrate gRPC streaming with Spring WebFlux, enabling the reactive handling of data streams. 
This approach provides a concise and reusable solution for bridging the gap between gRPC and reactive systems.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/grpc-java-webflux)._
