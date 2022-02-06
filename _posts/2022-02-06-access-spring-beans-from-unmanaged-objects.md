---
layout: post
title: "Access Spring Beans from Unmanaged Objects"
author: "Anar Sultanov"
image: /images/posts/2022-02-06/image.jpg
---

We may run into the need to inject Spring Beans into JPA entities or some other unmanaged objects,
which could be an indication that we need to rethink our architecture, but sometimes this cannot be avoided.
It is possible to do this using [@Configurable](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-atconfigurable) annotation,
but for this to work, the annotated types must be woven with the AspectJ weaver.
<br>In this tutorial, we are going to look at an alternative way to access Spring-managed components from unmanaged objects that is arguably better than injection.

## 1. Spring Bean
As an example, we will use a simple Spring application in which we will create a `TaxCalculator` component, 
which in turn could have some dependencies, but in our case, it is quite simple:
{% highlight java %}
@Component
public class TaxCalculator {

    public double calculate(double price) {
        return price * 0.25;
    }
}
{% endhighlight %}

## 2. Unmanaged Object
Now we will create a record for unmanaged objects:
{% highlight java %}
public record Invoice(double price) {}
{% endhighlight %}

Suppose we want them to provide their tax. To calculate it, we need to use `TaxCalculator`, 
but since instances of this class are not managed by Spring, we cannot simply inject this dependency into them.

## 3. Option 1: Expose an instance
Since Spring bean is [a singleton by default](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-scopes), 
we can inject it into its own static field by implementing the `InitializingBean` interface, or by using the `@PostConstruct` annotation, and then expose it with a static method.
In this example, we will use the annotation:
{% highlight java %}
public class TaxCalculator {

    private static TaxCalculator instance;

    public static TaxCalculator getInstance() {
        return instance;
    }

    @PostConstruct
    private void registerInstance() {
        instance = this;
    }

    // ...
}
{% endhighlight %}

Now we can use this method in our unmanaged objects:
{% highlight java %}
public record Invoice(double price) {

    public double calculateTaxUsingComponent() {
        return TaxCalculator.getInstance().calculate(this.price);
    }
}
{% endhighlight %}

Let's write a test by running which we can verify that everything works:
{% highlight java %}
@SpringBootTest
class InvoiceTest {

    @Test
    void calculateTaxUsingComponent() {
        // given
        var invoice = new Invoice(20);

        // when
        var result = invoice.calculateTaxUsingComponent();

        // then
        assertThat(result).isEqualTo(5.0);
    }
}
{% endhighlight %}


## 4. Option 2: Implement provider
Perhaps we don't want to change the class of the component or don't want to depend on a particular implementation, in which case we can implement a provider 
that will provide us with the component, or if we use an interface then an instance of its implementation found in the IoC container.
For example, let's update our component and make it implement the following interface:
{% highlight java %}
public interface ITaxCalculator {

    double calculate(double price);
}
{% endhighlight %}

Then we implement a provider, in the static field of which we inject our component and expose it using a static method:
{% highlight java %}
@Component
public class TaxCalculatorProvider {

    private static ITaxCalculator calculator;

    public TaxCalculatorProvider(ITaxCalculator calculator) {
        TaxCalculatorProvider.calculator = calculator;
    }

    public static ITaxCalculator getCalculator() {
        return calculator;
    }
}
{% endhighlight %}

Now we can use the provider in our unmanaged object:
{% highlight java %}
public record Invoice(double price) {

    public double calculateTaxUsingProvider() {
        return TaxCalculatorProvider.getCalculator().calculate(this.price);
    }
}
{% endhighlight %}

Finally, we can verify that everything works with a simple test:
{% highlight java %}
@SpringBootTest
class InvoiceTest {

    @Test
    void calculateTaxUsingProvider() {
        // given
        var invoice = new Invoice(50);

        // when
        var result = invoice.calculateTaxUsingProvider();

        // then
        assertThat(result).isEqualTo(12.5);
    }
}
{% endhighlight %}

## 5. Conclusion
While accessing managed components in this way is fairly easy, we shouldn't overuse it, 
as it can degrade the maintainability of the application by tying components and unmanaged objects together.
Often a better option would be to simply look up for dependency before a method is invoked and pass it in as an argument.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/spring-boot-unmanaged-objects)._
