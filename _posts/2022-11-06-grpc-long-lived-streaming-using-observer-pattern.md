---
layout: post
title: "gRPC Long-lived Streaming using Observer Pattern"
author: "Anar Sultanov"
image: /images/posts/2022-11-06/image.png
---

[gRPC](https://grpc.io/) builds on HTTP/2’s long-lived connections which provides a foundation for long-lived, real-time communication streams
and allows gRPC to support [multiple communication patterns](https://grpc.io/docs/what-is-grpc/core-concepts/#rpc-life-cycle), one of which is server streaming RPC.
<br>One use case for server streaming RPC could be to implement long-lived streaming of events or notifications from the server to interested clients, 
and in this tutorial, we are going to look at how to implement this with gRPC Java using [observer pattern](https://en.wikipedia.org/wiki/Observer_pattern).

## 1. Sample Domain
As an example, we will implement a service that simulates stock price updates and publishes events about it. 
We will also create a listener for these events and use it to notify gRPC clients using streaming.
And we will not use any extra dependencies other than [gRPC Java](https://github.com/grpc/grpc-java/tree/v1.50.x#download) and the libraries included in it, 
like [Guava](https://github.com/google/guava), which provides us with `EventBus` that we will use to implement a simple publish-subscribe communication.

First of all, let's implement a utility class that will allow us to publish events and subscribe to them:
{% highlight java %}
public class DomainEvents {

    private static final EventBus eventBus = new EventBus();

    private DomainEvents() {
        throw new AssertionError();
    }

    public static void publish(Object event) {
        eventBus.post(event);
    }

    public static void subscribe(Object eventListener) {
        eventBus.register(eventListener);
    }
}
{% endhighlight %}

Next, we define a record for the event that will be published:
{% highlight java %}
public record StockPriceChangedEvent(String symbol, double price) {}
{% endhighlight %}

As well as a domain model that will publish an event on update:
{% highlight java %}
public class Stock {

    private final String symbol;
    private double price;

    public Stock(String symbol, double price) {
        this.symbol = symbol;
        this.price = price;
    }

    public String getSymbol() {
        return symbol;
    }

    public double getPrice() {
        return price;
    }

    public void updatePrice(double price) {
        this.price = price;
        DomainEvents.publish(new StockPriceChangedEvent(this.symbol, this.price));
    }
}
{% endhighlight %}

We also implement an in-memory repository that we initialize with several stocks with random prices:
{% highlight java %}
public enum StockRepository {
    INSTANCE;

    private Map<String, Stock> stocks = new HashMap<>();

    StockRepository() {
        var random = ThreadLocalRandom.current();
        addStock(new Stock("GOOGL", random.nextInt(50, 200)));
        addStock(new Stock("AMZN", random.nextInt(50, 200)));
        addStock(new Stock("MSFT", random.nextInt(50, 200)));
        addStock(new Stock("AAPL", random.nextInt(50, 200)));
        addStock(new Stock("NFLX", random.nextInt(50, 200)));
    }

    public void addStock(Stock stock) {
        stocks.put(stock.getSymbol(), stock);
    }

    public Collection<Stock> getStocks() {
        return Collections.unmodifiableCollection(stocks.values());
    }

    public Optional<Stock> getStock(String symbol) {
        return Optional.ofNullable(stocks.get(symbol));
    }
}
{% endhighlight %}

After that, we implement a listener for events, which for now will simply print them:
{% highlight java %}
public class StockPriceChangedEventListener {

    @Subscribe
    public void handleEvent(StockPriceChangedEvent event) {
        System.out.println(event.toString());
    }
}
{% endhighlight %}

And we implement a task that will randomly update the price of one of the stocks every second:
{% highlight java %}
public class RandomStockPriceUpdatingTask implements Runnable {

    private final StockRepository repository = StockRepository.INSTANCE;

    @Override
    public void run() {
        while (true) {
            updateRandomStock();
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    private void updateRandomStock() {
        var random = ThreadLocalRandom.current();
        var stocks = repository.getStocks();
        var randomStock = stocks.stream().skip(random.nextInt(stocks.size())).findFirst().orElseThrow();
        var newPrice = randomStock.getPrice() + random.nextDouble(-5.0, 5.0);
        randomStock.updatePrice(newPrice);
    }
}
{% endhighlight %}

Finally, we create the main method where we register the listener and run the task on a separate thread:
{% highlight java %}
public class Main {

    public static void main(String[] args) {
        DomainEvents.subscribe(new StockPriceChangedEventListener());
        Executors.newFixedThreadPool(1).submit(new RandomStockPriceUpdatingTask());
    }
}
{% endhighlight %}

Now if we run the application, in the output we will see something like this:
{% highlight console %}
StockPriceChangedEvent[symbol=GOOGL, price=68.66692014709673]
StockPriceChangedEvent[symbol=AMZN, price=173.50718828139696]
StockPriceChangedEvent[symbol=AAPL, price=99.4590415304896]
StockPriceChangedEvent[symbol=MSFT, price=147.62425305550153]
StockPriceChangedEvent[symbol=NFLX, price=173.8481746249368]
StockPriceChangedEvent[symbol=GOOGL, price=69.86325610049497]
...
{% endhighlight %}


## 2. RPC Implementation
gRPC, like many RPC systems, is based on the idea of defining a service and its methods that can be called remotely, which is what we need to do first.

### 2.1. Service definition
The only method in our service will be server streaming RPC that returns prices of a requested stock:
{% highlight java %}
syntax = "proto3";
option java_multiple_files = true;
package dev.sultanov.grpc.streaming;

message StockPriceRequest {
    string symbol = 1;
}

message StockPriceResponse {
    string symbol = 1;
    double price = 2;
}

service StockService {
    rpc GetPrice(StockPriceRequest) returns (stream StockPriceResponse);
}
{% endhighlight %}


### 2.2. Server
Let's move on to implementing this service in our application.
<br/>Using server streaming RPC allows us to send one or more messages to the client before completing the call, 
but as an initial implementation, we will simply return the current price of the requested stock or an error if it is not found:
{% highlight java %}
public class StockService extends StockServiceGrpc.StockServiceImplBase {

    private final StockRepository repository = StockRepository.INSTANCE;

    @Override
    public void getPrice(StockPriceRequest request, StreamObserver<StockPriceResponse> responseObserver) {
        repository.getStock(request.getSymbol())
                .ifPresentOrElse(
                        stock -> {
                            responseObserver.onNext(StockPriceResponse.newBuilder().setSymbol(stock.getSymbol()).setPrice(stock.getPrice()).build());
                            responseObserver.onCompleted();
                        },
                        () -> responseObserver.onError(new StatusException(Status.NOT_FOUND))
                );
    }
}
{% endhighlight %}

Once we’ve implemented our service, we also need to start up a gRPC server, which we will do in the main method:
{% highlight java %}
public class Main {

    public static void main(String[] args) throws IOException, InterruptedException {
        // ...

        Server server = ServerBuilder
                .forPort(8080)
                .addService(new StockService())
                .build();

        server.start();
        System.out.println("gRPC Server started, listening on port:" + server.getPort());
        server.awaitTermination();
    }
}
{% endhighlight %}

### 2.3. Client
Now we can move on to creating a simple client.
<br/>We use a blocking stub to request the price of a certain stock and print responses:
{% highlight java %}
public class StockServiceClient {

    public static void main(String[] args) {
        var channel = ManagedChannelBuilder.forAddress("localhost", 8080)
                .usePlaintext()
                .build();
        var stub = StockServiceGrpc.newBlockingStub(channel);

        var stockPriceResponseIterator = stub.getPrice(StockPriceRequest.newBuilder().setSymbol("AMZN").build());
        stockPriceResponseIterator.forEachRemaining(StockServiceClient::printStockPrice);
    }

    private static void printStockPrice(StockPriceResponse stockPriceResponse) {
        System.out.println("Stock: " + stockPriceResponse.getSymbol() + ", price: " + stockPriceResponse.getPrice());
    }
}
{% endhighlight %}

By running the server and the client, we can verify that everything works, but there will be only one response in the output of the client, 
after which the connection is closed from the server side and the client process is finished.

## 3. Long-lived Streaming
But after all, our goal was to get updates from the server, otherwise, we could just use unary RPC.
Therefore, we will improve the processing of the request on the server by implementing the observer pattern.

### 3.1. Observer Pattern
The observer pattern encompasses two types of objects — a subject and an observer.
The subject maintains a list of observers and notifies them of state changes, usually by calling one of their methods.
In our implementation, `StreamObserver` from gRPC will act as an observer, and we only need to implement a subject.

#### 3.1.1. Registration
Our subject will allow observers to be registered by the stock symbol and will notify the right ones upon request.
We synchronize updates to the collection of observers, and to ensure that we notify only observers registered before the message is published to the subject but avoid holding a lock while notifying all observers, we implement copy-on-read and only synchronize this operation:
{% highlight java %}
public enum StockPriceChangedSubject {
    INSTANCE;

    private final Logger logger = Logger.getLogger(StockPriceChangedSubject.class.getName());
    private final Map<String, List<StreamObserver<StockPriceResponse>>> observers = new HashMap<>();
    private final Object MUTEX = new Object();

    public void register(String symbol, StreamObserver<StockPriceResponse> observer) {
        synchronized (MUTEX) {
            observers.computeIfAbsent(symbol, k -> new ArrayList<>()).add(observer);
        }
    }

    public void notify(StockPriceChangedEvent event) {
        List<StreamObserver<StockPriceResponse>> observerList;
        synchronized (MUTEX) {
            observerList = List.copyOf(this.observers.getOrDefault(event.symbol(), List.of()));
        }

        logger.log(Level.INFO, "Notifying {0} observers", observerList.size());
        var stockPrice = StockPriceResponse.newBuilder().setSymbol(event.symbol()).setPrice(event.price()).build();
        observerList.forEach(observer -> {
            try {
                observer.onNext(stockPrice);
            } catch (Exception e) {
                logger.log(Level.WARNING, "Error notifying observer", e);
            }
        });
    }
}
{% endhighlight %}
<i class="fa fa-exclamation-triangle"></i> **Note:** `StreamObservers` are not thread safe, if multiple threads will be writing to a `StreamObserver` concurrently, you must synchronize those calls as well.

Now let's update the gRPC service so that instead of completing the call, it registers an observer in the subject:
{% highlight java %}
public class StockService extends StockServiceGrpc.StockServiceImplBase {

    private final StockRepository repository = StockRepository.INSTANCE;
    private final StockPriceChangedSubject subject = StockPriceChangedSubject.INSTANCE;

    @Override
    public void getPrice(StockPriceRequest request, StreamObserver<StockPriceResponse> responseObserver) {
        repository.getStock(request.getSymbol())
                .ifPresentOrElse(
                        stock -> {
                            responseObserver.onNext(StockPriceResponse.newBuilder().setSymbol(stock.getSymbol()).setPrice(stock.getPrice()).build());
                            subject.register(request.getSymbol(), responseObserver);
                        },
                        () -> responseObserver.onError(new StatusException(Status.NOT_FOUND))
                );
    }
}
{% endhighlight %}

And also update the listener so that it requests notification of observers from the subject on events:
{% highlight java %}
public class StockPriceChangedEventListener {

    @Subscribe
    public void handleEvent(StockPriceChangedEvent event) {
        System.out.println(event.toString());
        StockPriceChangedSubject.INSTANCE.notify(event);
    }
}
{% endhighlight %}

By restarting the server and the client, we can see that now the client process does not terminate, and it receives all price updates, but we can notice a small problem if we terminate the client manually.
The subject will try to notify the observer, but since the call is canceled, it will endlessly display an error about this:
{% highlight console %}
INFO: Notifying 1 observers
WARNING: Error notifying observer
io.grpc.StatusRuntimeException: CANCELLED: call already cancelled. Use ServerCallStreamObserver.setOnCancelHandler() to disable this exception
{% endhighlight %}
To fix this, we need to implement the unregistration of observers when canceling a call.

#### 3.1.2. Unregistration
First, we need to add a method for unregistering to our subject:
{% highlight java %}
public enum StockPriceChangedSubject {
    // ...

    public void unregister(String symbol, StreamObserver<StockPriceResponse> observer) {
        synchronized (MUTEX) {
            Optional.ofNullable(observers.get(symbol)).ifPresent(observerList -> observerList.remove(observer));
        }
    }
}
{% endhighlight %}

After that, we need to add a callback to `StreamObserver` which will unregister it when the call is canceled.
We can do this in the subject itself when registering the observer, or in the service where we register it:
{% highlight java %}
public class StockService extends StockServiceGrpc.StockServiceImplBase {

    private final StockRepository repository = StockRepository.INSTANCE;
    private final StockPriceChangedSubject subject = StockPriceChangedSubject.INSTANCE;

    @Override
    public void getPrice(StockPriceRequest request, StreamObserver<StockPriceResponse> responseObserver) {
        repository.getStock(request.getSymbol())
                .ifPresentOrElse(
                        stock -> {
                            responseObserver.onNext(StockPriceResponse.newBuilder().setSymbol(stock.getSymbol()).setPrice(stock.getPrice()).build());
                            subject.register(request.getSymbol(), responseObserver);
                            var serverCallStreamObserver = ((ServerCallStreamObserver<StockPriceResponse>) responseObserver);
                            serverCallStreamObserver.setOnCancelHandler(() -> subject.unregister(request.getSymbol(), responseObserver));
                        },
                        () -> responseObserver.onError(new StatusException(Status.NOT_FOUND))
                );
    }
}
{% endhighlight %}
Now if we terminate the client process, we see that the observer has been successfully unregistered and there are no more errors in the server output.

## 3.2. Client-side cancellation
Currently, we have to completely terminate the client process or at least the thread to cancel a call, otherwise we have to wait until the server completes the call, which may not suit us.
Fortunately, there are other ways to cancel the call from the client side, one of which is using `CancellableContext`, which will work for all kinds of stubs, both synchronous and asynchronous.

To take advantage of this feature, we need to obtain a context using `Context.current().withCancellation()` and then use it to start a call with `CancellableContext#run(Runnable)`.
Now having a reference to the context, we can cancel the call at any stage using `CancellableContext#cancel(Throwable)`.

Let's look at its use in practice, for example, imagine that we want to receive only 5 responses from the server, and then cancel the call.
To do this, we start a call as described above, and also implement a counter that we will increment as each response is processed, then simply end the call when the counter reaches 5:
{% highlight java %}
public class StockServiceClient {
    // ...

    public static void main(String[] args) {
        var channel = ManagedChannelBuilder.forAddress("localhost", 8080)
                .usePlaintext()
                .build();
        var stub = StockServiceGrpc.newBlockingStub(channel);

        try (var cancellableContext = Context.current().withCancellation()) {
            var counter = new AtomicInteger();
            cancellableContext.run(() -> {
                var response = stub.getPrice(StockPriceRequest.newBuilder().setSymbol("AMZN").build());
                response.forEachRemaining(stockPriceResponse -> {
                    StockServiceClient.printStockPrice(stockPriceResponse);
                    if (counter.incrementAndGet() == 5) {
                        cancellableContext.cancel(null);
                    }
                });
            });
        } catch (StatusRuntimeException e) {
            var status = Status.fromThrowable(e);
            logger.log(Level.WARNING, "Cancelled with status: " + status);
        }

        System.out.println("Stock price retrieval completed!");
    }
}
{% endhighlight %}
Running the updated client, we see that it will cancel the call after receiving 5 responses and continue the main method by printing the string we specified.

## 4. Conclusion
We looked at how to implement long-lived streaming in gRPC Java, reaffirming just how powerful and versatile this framework is.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/grpc-java-long-lived-stream)._
