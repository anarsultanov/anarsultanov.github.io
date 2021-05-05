---
layout: post
title: "Orchestration-based Saga using Cadence Workflow"
author: "Anar Sultanov"
image: /images/posts/2021-05-05/image.jpg
---
[The Saga pattern](https://microservices.io/patterns/data/saga.html) is a way to maintain data consistency in distributed systems by sacrificing atomicity and relying on eventual consistency.
A saga is a sequence of transactions that update each service, and if at some stage it fails, it performs compensating transactions that counteract the previous stages.
There are two common saga implementation approaches - choreography and orchestration.
<br>In this tutorial, we are going to look at how to implement orchestration-based saga using Cadence Workflow.

[Cadence](https://cadenceworkflow.io/) is a reliable orchestration engine for executing asynchronous business logic.
It is well suited for implementing orchestration-based sagas because it simplifies the coding of compensation logic, has built-in support for unlimited exponential retry attempts,
and guarantees that the workflow code eventually completes. 
In addition, it gives full visibility into the state of each workflow, unlike queue-based orchestration, where getting the current status of each individual request is almost impossible.

## 1. Prerequisites
We need a Cadence server running locally and the easiest way is to run it in Docker.
Download the docker-compose file from the official repository and start the services:
{% highlight console %}
> curl -O https://raw.githubusercontent.com/uber/cadence/master/docker/docker-compose.yml
> docker-compose up
{% endhighlight %}

Next, we need to register a domain for further use.
We could do this from Java, but since this is a one-time operation, we will do it using the CLI.
Let's register a domain with the name `example`:
{% highlight console %}
> docker run --network=host --rm ubercadence/cli:master --do example domain register -rd 1
{% endhighlight %}

## 2. Project
We will implement all services in one maven project to simplify the example and make it easier to share dependencies as well as some classes.

We need to include Cadence Java Client and some dependencies that it relies on:
```
<dependency>
    <groupId>com.uber.cadence</groupId>
    <artifactId>cadence-client</artifactId>
    <version>3.0.0</version>
</dependency>
<dependency>
    <groupId>commons-configuration</groupId>
    <artifactId>commons-configuration</artifactId>
    <version>1.10</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.3</version>
</dependency>
```

We will also add Spring Boot web starter to create a rest controller and Lombok for convenience:
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>2.4.5</version>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.20</version>
    <scope>provided</scope>
</dependency>
```

Finally, we exclude the debug logs by adding the following logback config file to the classpath:
{% highlight xml %}
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- encoders are assigned the type
             ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    <logger name="io.netty" level="INFO"/>
    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
{% endhighlight %}

## 3. Application
We will implement the sample e-commerce application provided on [the Saga pattern description page](https://microservices.io/patterns/data/saga.html):
1. The `Order Service` receives the `POST /orders` request and starts the `Create Order` saga orchestrator
2. The saga orchestrator creates an `Order` in the `PENDING` state
3. It then sends a `Reserve Credit` command to the `Customer Service`
4. The `Customer Service` attempts to reserve credit
5. It then sends back a reply message indicating the outcome
6. The saga orchestrator either approves or rejects the `Order`

Before moving on to implementing services, let's take a look at a few Cadence concepts, which include [workflows](https://cadenceworkflow.io/docs/concepts/workflows/) and [activities](https://cadenceworkflow.io/docs/concepts/activities/).
A workflow is a fault-oblivious stateful abstraction that is not allowed to call an external API directly due to deterministic execution requirements. 
Instead, it organizes the execution of activities, which can be a function or method in one of the supported languages and can contain any code without restriction.

From this, we can conclude that we need to define and implement activities in each of our services participating in the saga, and then define and implement a workflow that will orchestrate these activities.

#### 3.1 Activities
Activities are defined as methods of an interface, and each method defines a single activity type.
Since these interfaces are also required on the client side to invoke the activity, we will put them in a common package.

The `Customer Service` should attempt to reserve credit and return the result:
{% highlight java %}
public interface CustomerActivities {

    boolean reserveCredit(String customerId, BigDecimal amount);
}
{% endhighlight %}

The `Order Service` should create new pending orders and approve or reject them based on the response from the `Customer Service`:
{% highlight java %}
public interface OrderActivities {

    String createOrder(String customerId, BigDecimal totalMoney);
    void approveOrder(String orderId);
    void rejectOrder(String orderId);
}
{% endhighlight %}

#### 3.2 Workflow

Workflows are also defined in the interface, but unlike activities, in addition to the entry point method,
they can contain additional methods to react to external signals and query requests, so each method must be annotated accordingly.

We are not interested in additional features and will only use the `@WorkflowMethod` annotation:
{% highlight java %}
public interface CreateOrderWorkflow {
    @WorkflowMethod
    String createOrder(String customerId, BigDecimal totalMoney);
}
{% endhighlight %}

#### 3.3 Constants
In addition to the activity interfaces, our services will share some constants, 
such as the domain name that we created on the Cadence Server, as well as the names of [task lists](https://cadenceworkflow.io/docs/concepts/task-lists/), 
which are queues used to deliver tasks to workers:
{% highlight java %}
public class Constants {
    public static final String DOMAIN = "example";
    public static final String CUSTOMER_TASK_LIST = "CustomerTaskList";
    public static final String ORDER_TASK_LIST = "OrderTaskList";
}
{% endhighlight %}

#### 3.4 Customer Service
First, we will add a very simple implementation of the activity interface so as not to overload the example with extraneous logic:
{% highlight java %}
public class CustomerActivitiesImpl implements CustomerActivities {

    private static final Logger logger = LoggerFactory.getLogger(CustomerActivitiesImpl.class);

    @Override
    public boolean reserveCredit(String customerId, BigDecimal amount) {
        if (amount.compareTo(BigDecimal.valueOf(100)) > 0) {
            logger.info("Credit limit is exceeded for customer {}", customerId);
            return false;
        }
        logger.info("Credit reserved for customer {}", customerId);
        return true;
    }
}
{% endhighlight %}

After that, we need to register the implementation with a worker that connects to Cadence. 
Using the default options, it will connect to the locally running server:
{% highlight java %}
public class CustomerApplication {

    public static void main(String[] args) {
        IWorkflowService service = new WorkflowServiceTChannel(ClientOptions.defaultInstance());

        WorkflowClientOptions workflowClientOptions = WorkflowClientOptions.newBuilder()
                .setDomain(Constants.DOMAIN)
                .build();
        WorkflowClient client = WorkflowClient.newInstance(service, workflowClientOptions);

        WorkerFactory factory = WorkerFactory.newInstance(client);
        Worker worker = factory.newWorker(CUSTOMER_TASK_LIST);
        worker.registerActivitiesImplementations(new CustomerActivitiesImpl());
        factory.start();
    }
}
{% endhighlight %}

#### 3.5 Order Service
Next, we will also implement the activities of the `Order Service`:
{% highlight java %}
public class OrderActivitiesImpl implements OrderActivities {

    private static final Logger logger = LoggerFactory.getLogger(OrderActivitiesImpl.class);

    @Override
    public String createOrder(String customerId, BigDecimal amount) {
        var orderId = UUID.randomUUID().toString();
        logger.info("Order {} created in pending state", orderId);
        return orderId;
    }

    @Override
    public void approveOrder(String orderId) {
        logger.info("Order {} approved", orderId);
    }

    @Override
    public void rejectOrder(String orderId) {
        logger.info("Order {} rejected", orderId);
    }
}
{% endhighlight %}

We could implement the workflow in a separate service, but we will add it to the `Order Service` and make it also responsible for orchestrating the saga.
We use `Workflow.newActivityStub`, which takes activity type and activity options as arguments to create client-side stubs for our activities.
In the implemented method, we instantiate the saga object with default parameters, call the order activity to create a new order and register its rejection as compensation, 
then we try to reserve credit and, if successful, approve the order and return its ID, otherwise we throw an exception that, 
like any unexpected exception, gets caught by the `try-catch` block which triggers a rollback of the saga, performing all registered compensations:
{% highlight java %}
public class CreateOrderWorkflowImpl implements CreateOrderWorkflow {

    private final ActivityOptions customerActivityOptions = new ActivityOptions.Builder()
            .setTaskList(CUSTOMER_TASK_LIST)
            .setScheduleToCloseTimeout(Duration.ofSeconds(2))
            .build();
    private final CustomerActivities customerActivities =
            Workflow.newActivityStub(CustomerActivities.class, customerActivityOptions);

    private final ActivityOptions orderActivityOptions = new ActivityOptions.Builder()
            .setTaskList(ORDER_TASK_LIST)
            .setScheduleToCloseTimeout(Duration.ofSeconds(2))
            .build();
    private final OrderActivities orderActivities =
            Workflow.newActivityStub(OrderActivities.class, orderActivityOptions);

    @Override
    public String createOrder(String customerId, BigDecimal amount) {
        Saga.Options sagaOptions = new Saga.Options.Builder().build();
        Saga saga = new Saga(sagaOptions);
        try {
            String orderId = orderActivities.createOrder(customerId, amount);
            saga.addCompensation(orderActivities::rejectOrder, orderId);

            if (customerActivities.reserveCredit(customerId, amount)) {
                orderActivities.approveOrder(orderId);
                return orderId;
            } else {
                saga.compensate();
                throw new IllegalStateException("Failed to reserve credit");
            }
        } catch (ActivityException e) {
            saga.compensate();
            throw e;
        }
    }
}
{% endhighlight %}

We are planning to use Spring Boot in this service, so its main class will be slightly different from the previous one.
We again create a worker and register the activities, but we also need to register the workflow. 
In addition, we do it using the `CommandLineRunner`, not in the main method, and we separate the creation of the `WorkflowClient` into a bean, since we will need it later:
{% highlight java %}
@SpringBootApplication
public class OrderApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }

    @Bean
    WorkflowClient workflowClient() {
        IWorkflowService service = new WorkflowServiceTChannel(ClientOptions.defaultInstance());

        WorkflowClientOptions workflowClientOptions = WorkflowClientOptions.newBuilder()
                .setDomain(DOMAIN)
                .build();
        return WorkflowClient.newInstance(service, workflowClientOptions);
    }

    @Bean
    CommandLineRunner commandLineRunner(WorkflowClient workflowClient) {
        return args -> {
            WorkerFactory factory = WorkerFactory.newInstance(workflowClient);
            Worker worker = factory.newWorker(ORDER_TASK_LIST);
            worker.registerActivitiesImplementations(new OrderActivitiesImpl());
            factory.start();
        };
    }
}
{% endhighlight %}

Finally, we create a controller in which we implement two ways to start workflow execution, synchronously and asynchronously.
We will use a client-side stub to start a workflow that we instantiate by using `WorkflowClient`,
but note that on each request we create a new stub to start a new workflow, and not reconnect to the same one:
{% highlight java %}
@RestController
@RequestMapping("/orders")
@RequiredArgsConstructor
public class OrderController {

    private final WorkflowClient workflowClient;

    @PostMapping("/sync")
    public ResponseEntity<CreateOrderResponse> createOrderAsync(@RequestBody CreateOrderRequest request) {
        var workflow = workflowClient.newWorkflowStub(
                CreateOrderWorkflow.class,
                new WorkflowOptions.Builder()
                        .setExecutionStartToCloseTimeout(Duration.ofSeconds(5))
                        .setTaskList(ORDER_TASK_LIST)
                        .build()
        );
        var orderId = workflow.createOrder(request.customerId, request.amount);
        return ResponseEntity.ok(new CreateOrderResponse(orderId));
    }
    
    @PostMapping("/async")
    public ResponseEntity<Void> createOrder(@RequestBody CreateOrderRequest request) {
        var workflow = workflowClient.newWorkflowStub(
                CreateOrderWorkflow.class,
                new WorkflowOptions.Builder()
                        .setExecutionStartToCloseTimeout(Duration.ofSeconds(5))
                        .setTaskList(ORDER_TASK_LIST)
                        .build()
        );
        WorkflowClient.execute(workflow::createOrder, request.customerId, request.amount);
        return ResponseEntity.accepted().build();
    }

    @Value
    @NoArgsConstructor(force = true)
    private static class CreateOrderRequest {
        String customerId;
        BigDecimal amount;
    }

    @Value
    private static class CreateOrderResponse {
        String orderId;
    }
}
{% endhighlight %}

We can now start both of our services and make sure everything works.

## 4. Test
First, let's make a request to start the workflow asynchronously:
{% highlight console %}
> curl -X POST -H "Content-Type: application/json" -d "{\"customerId\":\"0123\", \"amount\":80}" http://localhost:8080/orders/async
{% endhighlight %}

If we take a look at the logs of our services we will see something like:
```
19:58:28.081 [Activity Executor taskList="OrderTaskList", domain="example": 1] INFO  d.s.c.s.s.order.OrderActivitiesImpl - Order 85a203c5-3147-4078-8a00-eb9eeada5494 created in pending state
19:58:28.288 [Activity Executor taskList="CustomerTaskList", domain="example": 1] INFO  d.s.c.s.s.c.CustomerActivitiesImpl - Credit reserved for customer 0123
19:58:28.579 [Activity Executor taskList="OrderTaskList", domain="example": 1] INFO  d.s.c.s.s.order.OrderActivitiesImpl - Order 85a203c5-3147-4078-8a00-eb9eeada5494 approved
```

Now let's try to run the workflow synchronously:
{% highlight console %}
> curl -X POST -H "Content-Type: application/json" -d "{\"customerId\":\"3456\", \"amount\":72}" http://localhost:8080/orders/sync
{"orderId":"c5fdd661-6618-44ea-9723-966a5e5f7390"}
{% endhighlight %}

We have successfully received an order ID in response and the logs correspond to the previous ones:
```
19:58:46.432 [Activity Executor taskList="OrderTaskList", domain="example": 2] INFO  d.s.c.s.s.order.OrderActivitiesImpl - Order c5fdd661-6618-44ea-9723-966a5e5f7390 created in pending state
19:58:46.502 [Activity Executor taskList="CustomerTaskList", domain="example": 2] INFO  d.s.c.s.s.c.CustomerActivitiesImpl - Credit reserved for customer 3456
19:58:46.559 [Activity Executor taskList="OrderTaskList", domain="example": 2] INFO  d.s.c.s.s.order.OrderActivitiesImpl - Order c5fdd661-6618-44ea-9723-966a5e5f7390 approved
```

Finally, let's try to create an order with an excess of the credit limit:
{% highlight console %}
> curl -X POST -H "Content-Type: application/json" -d "{\"customerId\":\"5678\", \"amount\":170}" http://localhost:8080/orders/sync
{% endhighlight %}

In the case of a request for synchronous execution, an internal server error is expected, because we are not handling exceptions properly, 
and in the logs we should see something like:
```
19:59:00.621 [Activity Executor taskList="OrderTaskList", domain="example": 3] INFO  d.s.c.s.s.order.OrderActivitiesImpl - Order 884c1def-cd52-4efa-8853-35fd19ff60e8 created in pending state
19:59:00.762 [Activity Executor taskList="CustomerTaskList", domain="example": 3] INFO  d.s.c.s.s.c.CustomerActivitiesImpl - Credit limit is exceeded for customer 5678
19:59:00.876 [Activity Executor taskList="OrderTaskList", domain="example": 3] INFO  d.s.c.s.s.order.OrderActivitiesImpl - Order 884c1def-cd52-4efa-8853-35fd19ff60e8 rejected
```

We can also check the status and details of the executed workflows in the Cadence Web UI. 
To do this, in the browser, go to [http://localhost:8088/domains/example/workflows](http://localhost:8088/domains/example/workflows)

## 5. Conclusion
We've implemented an orchestration-based saga using Cadence, but we've covered a small fraction of the capabilities of this
promising platform that can be adapted for a wide variety of use cases and really deserves a look.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/cadence-saga)._
