---
layout: post
title: "Unified GraphQL API for gRPC microservices using Rejoiner and Spring Boot"
author: "Anar Sultanov"
image: /images/posts/2019-10-19/image.jpg
---

[Rejoiner](https://rejoiner.io/) is a fairly young framework with the goal of bringing together two powerful and increasingly popular technologies, GraphQL and gRPC.
It creates a unified GraphQL schema for gRPC microservices and provides DSL to modify it.
<br>In this tutorial, we are going to create several gRPC microservices and see how we can use Rejoiner to expose them via a single GraphQL API.

## 1. gRPC microservices
We will start by creating gRPC services. 
If you are new to using the gRPC, check out the [quick start guide](https://grpc.io/docs/quickstart/java.html) or the more detailed [gRPC basics](https://grpc.io/docs/tutorials/basic/java.html).

### 1.1. Author Service
We define our first service in a proto file:
{% highlight proto %}
syntax = "proto3";

option java_multiple_files = true;
option java_package = "dev.sultanov.grpc.author";

import "google/protobuf/empty.proto";

message Author {
    int32 id = 1;
    string firstName = 2;
    string lastName = 3;
    repeated int32 bookIds = 4;
}

service AuthorService {
    rpc createAuthor (CreateAuthorRequest) returns (Author);
    rpc getAuthor (GetAuthorRequest) returns (Author);
    rpc addBook(AddBookRequest) returns (Author);
}

message CreateAuthorRequest {
    string firstName = 2;
    string lastName = 3;
}

message GetAuthorRequest {
    int32 id = 1;
}

message AddBookRequest {
    int32 authorId = 1;
    int32 bookId = 2;
}
{% endhighlight %}

Then we implement the service by extending the class generated from the proto file:
{% highlight java %}
public class AuthorService extends AuthorServiceGrpc.AuthorServiceImplBase {

    private final AtomicInteger authorIdCounter = new AtomicInteger(1);
    private final Map<Integer, Author> authorsById = new HashMap<>();

    public void createAuthor(CreateAuthorRequest request, StreamObserver<Author> responseObserver) {
        int id = authorIdCounter.getAndIncrement();
        Author author = Author.newBuilder()
                .setId(id)
                .setFirstName(request.getFirstName())
                .setLastName(request.getLastName())
                .build();
        authorsById.put(id, author);
        responseObserver.onNext(author);
        responseObserver.onCompleted();
    }

    public void getAuthor(GetAuthorRequest request, StreamObserver<Author> responseObserver) {
        responseObserver.onNext(authorsById.get(request.getId()));
        responseObserver.onCompleted();
    }

    public void addBook(AddBookRequest request, StreamObserver<Author> responseObserver) {
        int authorId = request.getAuthorId();
        Author author = authorsById.get(authorId).toBuilder()
                .addBookIds(request.getBookId())
                .build();
        authorsById.put(authorId, author);
        responseObserver.onNext(author);
        responseObserver.onCompleted();
    }
}
{% endhighlight %}

Finally, we create a class that starts the server serving our service:
{% highlight java %}
public class AuthorServer {
    public static void main(String[] args) throws IOException, InterruptedException {
        Server server = ServerBuilder
                .forPort(50001)
                .addService(new AuthorService())
                .build();

        server.start();
        System.out.println("gRPC Server started, listening on port:" + server.getPort());
        server.awaitTermination();
    }
}
{% endhighlight %}

### 1.2. Book Service
We do the same for the second service. Create a proto file:
{% highlight proto %}
syntax = "proto3";

option java_multiple_files = true;
option java_package = "dev.sultanov.grpc.book";

import "google/protobuf/empty.proto";

message Book {
  int32 id = 1;
  string title = 2;
  int32 authorId = 3;
}

service BookService {
  rpc createBook(CreateBookRequest) returns (Book);
  rpc getBook(GetBookRequest) returns (Book);
  rpc listBooks(ListBooksRequest) returns (ListBooksResponse);
}

message CreateBookRequest {
  string title = 2;
  int32 authorId = 3;
}

message GetBookRequest {
  int32 id = 1;
}

message ListBooksRequest {
  repeated int32 ids = 1;
}

message ListBooksResponse {
  repeated Book books = 1;
}
{% endhighlight %}

Implement the service:
{% highlight java %}
public class BookService extends BookServiceGrpc.BookServiceImplBase {

    private final AtomicInteger bookIdCounter = new AtomicInteger(1);
    private final Map<Integer, Book> booksById = new HashMap<>();

    public void createBook(CreateBookRequest request, StreamObserver<Book> responseObserver) {
        int id = bookIdCounter.getAndIncrement();
        Book book = Book.newBuilder()
                .setId(id)
                .setTitle(request.getTitle())
                .setAuthorId(request.getAuthorId())
                .build();
        booksById.put(id, book);
        responseObserver.onNext(book);
        responseObserver.onCompleted();
    }

    public void getBook(GetBookRequest request, StreamObserver<Book> responseObserver) {
        responseObserver.onNext(booksById.get(request.getId()));
        responseObserver.onCompleted();
    }

    public void listBooks(ListBooksRequest request, StreamObserver<ListBooksResponse> responseObserver) {
        ListBooksResponse.Builder builder = ListBooksResponse.newBuilder();
        request.getIdsList().forEach(id -> {
            builder.addBooks(booksById.get(id));
        });
        responseObserver.onNext(builder.build());
        responseObserver.onCompleted();
    }
}
{% endhighlight %}

Create a class starting the server:
{% highlight java %}
public class BookServer {
    public static void main(String[] args) throws IOException, InterruptedException {
        Server server = ServerBuilder
                .forPort(50002)
                .addService(new BookService())
                .build();

        server.start();
        System.out.println("gRPC Server started, listening on port:" + server.getPort());
        server.awaitTermination();
    }
}
{% endhighlight %}

## 2.GraphQL Gateway
Now we move on to creating a gateway that will expose our services via GraphQL.

### 2.1. Rejoiner
We will use [Google Guice](https://github.com/google/guice) to initialize all the components used by Rejoiner, 
as it utilizes Guice internally and requires this.

#### 2.1.1 gRPC Clients
First, we create modules in which we define Guice bindings for stubs used to communicate with gRPC services:
{% highlight java %}
final class AuthorClient extends AbstractModule {

    private static final String HOST = "localhost";
    private static final int PORT = 50001;

    @Override
    protected void configure() {
        ManagedChannel channel = ManagedChannelBuilder.forAddress(HOST, PORT).usePlaintext().build();
        bind(AuthorServiceGrpc.AuthorServiceFutureStub.class).toInstance(AuthorServiceGrpc.newFutureStub(channel));
        bind(AuthorServiceGrpc.AuthorServiceBlockingStub.class).toInstance(AuthorServiceGrpc.newBlockingStub(channel));
    }
}
{% endhighlight %}
{% highlight java %}
final class BookClient extends AbstractModule {

    private static final String HOST = "localhost";
    private static final int PORT = 50002;

    @Override
    protected void configure() {
        ManagedChannel channel = ManagedChannelBuilder.forAddress(HOST, PORT).usePlaintext().build();
        bind(BookServiceGrpc.BookServiceFutureStub.class).toInstance(BookServiceGrpc.newFutureStub(channel));
        bind(BookServiceGrpc.BookServiceBlockingStub.class).toInstance(BookServiceGrpc.newBlockingStub(channel));
    }
}
{% endhighlight %}

And for convenience, we group these modules into one:
{% highlight java %}
public final class GrpcClientModule extends AbstractModule {
    @Override
    protected void configure() {
        install(new BookClient());
        install(new AuthorClient());
    }
}
{% endhighlight %}

#### 2.1.2 Schema
`SchemaModule` is a core component of Rejoiner.
It looks at the parameters and return type of methods that have Rejoiner annotations to generate parts of the schema,
which are then combined into the final schema using `SchemaProviderModule`.

We create a schema module for the first service using Rejoiner's `@Query` and `@Mutation` annotations:
{% highlight java %}
final class AuthorSchemaModule extends SchemaModule {

    @Mutation("createAuthor")
    ListenableFuture<Author> createAuthor(CreateAuthorRequest request, AuthorServiceGrpc.AuthorServiceFutureStub stub) {
        return stub.createAuthor(request);
    }

    @Query("getAuthor")
    ListenableFuture<Author> getAuthor(GetAuthorRequest request, AuthorServiceGrpc.AuthorServiceFutureStub stub) {
        return stub.getAuthor(request);
    }

    @Mutation("addBook")
    ListenableFuture<Author> addBook(AddBookRequest request, AuthorServiceGrpc.AuthorServiceFutureStub stub) {
        return stub.addBook(request);
    }
}
{% endhighlight %}

We also create a schema module for the second service using the same annotations, 
but note that the mutation here actually updates data in both services:
{% highlight java %}
final class BookSchemaModule extends SchemaModule {

    @Mutation("createBook")
    Book createBook(CreateBookRequest request, BookServiceGrpc.BookServiceBlockingStub bookStub,
                    AuthorServiceGrpc.AuthorServiceBlockingStub authorStub) {
        Book book = bookStub.createBook(request);
        Author author = authorStub.addBook(AddBookRequest.newBuilder()
                .setAuthorId(request.getAuthorId())
                .setBookId(book.getId()).build());
        return book;
    }

    @Query("getBook")
    ListenableFuture<Book> getBook(GetBookRequest request, BookServiceGrpc.BookServiceFutureStub stub) {
        return stub.getBook(request);
    }
}
{% endhighlight %}

And since `SchemaModule` is a Guice module, we can combine these modules into one module as well:
{% highlight java %}
public class GraphQlSchemaModule extends AbstractModule {
    @Override
    protected void configure() {
        install(new BookSchemaModule());
        install(new AuthorSchemaModule());
    }
}
{% endhighlight %}
### 2.2. Spring Boot
Rejoiner provides us with a GraphQL schema, but we need to bind it to an HTTP server.
We will use `spring-boot-starter-web` to built a web application and `graphql-spring-boot-starter` to turn it into a GraphQL server.
Using Spring Boot not only simplifies the creation of our application 
but also makes it easy to add features such as authentication to our gateway, if necessary.

First, we create a standard main class to bootstrap our Spring application:
{% highlight java %}
@SpringBootApplication
public class GraphQlApplication {

    public static void main(String[] args) {
        SpringApplication.run(GraphQlApplication.class, args);
    }
}
{% endhighlight %}

We need to add an initializer block to initialize the Guice modules before the Spring context:
{% highlight java %}
    private final Injector injector;
    
    {
        injector = Guice.createInjector(
                new SchemaProviderModule(),
                new GrpcClientModule(),
                new GraphQlSchemaModule()
        );
    }
{% endhighlight %}

Then we create a bean of the `GraphQLSchemaProvider` type which uses the Rejoiner-generated schema obtained from Guice Injector:
{% highlight java %}
    @Bean
    public GraphQLSchemaProvider schemaProvider() {
        GraphQLSchema schema = injector.getInstance(Key.get(GraphQLSchema.class, Schema.class));
        return new DefaultGraphQLSchemaProvider(schema);
    }
{% endhighlight %}

Finally, we need to register a custom `Instrumentation` to support Guava's `ListenableFuture`:
{% highlight java %}
    @Bean
    public Instrumentation instrumentation() {
        return GuavaListenableFutureSupport.listenableFutureInstrumentation();
    }
{% endhighlight %}

Our gateway is ready, and we can run it for testing, but first, we will look at some additional features of Rejoiner.

### 2.3. Data loader
Now we will see how we can use the other annotations provided by Rejoiner to modify the schema, and add Data Loader support for batch loading.

We will use the book service stub to implement batch loading of books. We get it from the injector and declare a bean:
{% highlight java %}
    @Bean
    public BookServiceGrpc.BookServiceFutureStub bookServiceFutureStub() {
        return injector.getInstance(BookServiceGrpc.BookServiceFutureStub.class);
    }
{% endhighlight %}

We need to implement `BatchLoader<K, V>` functional interface whose first type indicates the type of keys used for data loading requests, 
and the second type indicates the type of returned values.
In our case, we will load books by their identifier, so we need to implement `BatchLoader<Integer, Book>`.
We also need to use `FutureConverter` to turn `ListenableFuture` into `CompletableFuture` required by the interface:
{% highlight java %}
@Component
public class BookBatchLoader implements BatchLoader<Integer, Book> {

    private final BookServiceGrpc.BookServiceFutureStub futureStub;

    public BookBatchLoader(BookServiceGrpc.BookServiceFutureStub futureStub) {
        this.futureStub = futureStub;
    }

    @Override
    public CompletionStage<List<Book>> load(List<Integer> keys) {
        ListenableFuture<List<Book>> listenableFuture =
                Futures.transform(futureStub.listBooks(ListBooksRequest.newBuilder().addAllIds(keys).build()),
                        listBooksResponse -> listBooksResponse != null ? listBooksResponse.getBooksList() : null,
                        MoreExecutors.directExecutor());
        return FutureConverter.toCompletableFuture(listenableFuture);
    }
}
{% endhighlight %}

To start using our `BatchLoader`, we have to register it in `DataLoaderRegistry`, which in turn needs to be added to `GraphQLContext`:
{% highlight java %}
    @Bean
    public DataLoaderRegistry buildDataLoaderRegistry(BookBatchLoader bookBatchLoader) {
        DataLoaderRegistry registry = new DataLoaderRegistry();
        registry.register("books", new DataLoader<>(bookBatchLoader));
        return registry;
    }

    @Bean
    public GraphQLContextBuilder contextBuilder(DataLoaderRegistry dataLoaderRegistry) {
        return new GraphQLContextBuilder() {
            @Override
            public GraphQLContext build(HttpServletRequest req, HttpServletResponse response) {
                return DefaultGraphQLServletContext.createServletContext(dataLoaderRegistry, null).with(req).with(response).build();
            }

            @Override
            public GraphQLContext build() {
                return new DefaultGraphQLContext(dataLoaderRegistry, null);
            }

            @Override
            public GraphQLContext build(Session session, HandshakeRequest request) {
                return DefaultGraphQLWebSocketContext.createWebSocketContext(dataLoaderRegistry, null).with(session).with(request).build();
            }
        };
    }
{% endhighlight %}

To use the data loader in our schema modules, we need to obtain it from `DataFetchingEnvironment`.
Let's try to use schema modifications to remove the field with book IDs from the author schema 
and add a field with book objects loaded using the data loader:
{% highlight java %}
final class AuthorSchemaModule extends SchemaModule {

    // ...

    @SchemaModification
    TypeModification removeBookIds = Type.find(Author.getDescriptor()).removeField("bookIds");

    @SchemaModification(addField = "books", onType = Author.class)
    ListenableFuture<List<Book>> authorToBooks(Author author, DataFetchingEnvironment environment) {
        return FutureConverter.toListenableFuture(
                environment.<Integer, Book>getDataLoader("books").loadMany(author.getBookIdsList())
        );
    }
}
{% endhighlight %}

## 3. Test
Now we can run our application for testing (do not forget to also start the gRPC services).
We can use any third-party GraphQL client or add `graphiql-spring-boot-starter` as a dependency to embed GraphiQL tool
which becomes accessible at the root `/graphiql` 

First we create an author:
<span class="image fit"><img src="{{ "/images/posts/2019-10-19/screenshot_1.jpg" | absolute_url }}" alt="" /></span>

Then we create a book of this author using the author ID received in the first request:
<span class="image fit"><img src="{{ "/images/posts/2019-10-19/screenshot_2.jpg" | absolute_url }}" alt="" /></span>

And create another book using the same author ID:
<span class="image fit"><img src="{{ "/images/posts/2019-10-19/screenshot_3.jpg" | absolute_url }}" alt="" /></span>

Finally, we query the author along with the titles of the nested books by his ID:
<span class="image fit"><img src="{{ "/images/posts/2019-10-19/screenshot_4.jpg" | absolute_url }}" alt="" /></span>

## 4. Conclusion
GRPC and GraphQL are great technologies that complement each other perfectly when creating an application with a microservice architecture.
GraphQL gives clients the ability to request exactly the data they need, while gRPC makes internal communication between services more efficient,
and Rejoiner makes it easier to use them together. 

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/grpc-graphql-api)._
