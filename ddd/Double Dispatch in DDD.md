# Double Dispatch in DDD



## **1. Overview**



Double dispatch is a technical term to describe the **process of choosing the method to invoke based both on receiver and argument types.**

A lot of developers often confuse double dispatch with [Strategy Pattern](https://en.wikipedia.org/wiki/Strategy_pattern).

Java doesn't support double dispatch, but there are techniques we can employ to overcome this limitation.

**In this tutorial, we'll focus on showing examples of double dispatch in the context of Domain-driven Design (DDD) and Strategy Pattern.**



## **2. Double Dispatch** 

Before we discuss double dispatch, let's review some basics and explain what Single Dispatch actually is.

### 2.1. Single Dispatch

**Single dispatch is a way to choose the implementation of a method based on the receiver runtime type.** In Java, this is basically the same thing as polymorphism.

For example, let's take a look at this simple discount policy interface:

```java
public interface DiscountPolicy {
    double discount(Order order);
}
```

The *DiscountPolicy* interface has two implementations. The flat one, which always returns the same discount:

```java
public class FlatDiscountPolicy implements DiscountPolicy {
    @Override
    public double discount(Order order) {
        return 0.01;
    }
}
```

And the second implementation, which returns the discount based on the total cost of the order:

```java
public class AmountBasedDiscountPolicy implements DiscountPolicy {
    @Override
    public double discount(Order order) {
        if (order.totalCost()
            .isGreaterThan(Money.of(CurrencyUnit.USD, 500.00))) {
            return 0.10;
        } else {
            return 0;
        }
    }
}
```

For the needs of this example, let's assume the *Order* class has a *totalCost()* method.

**Now, single dispatch in Java is just a very well-known polymorphic behavior demonstrated in the following test:**

```java
@DisplayName(
    "given two discount policies, " +
    "when use these policies, " +
    "then single dispatch chooses the implementation based on runtime type"
    )
@Test
void test() throws Exception {
    // given
    DiscountPolicy flatPolicy = new FlatDiscountPolicy();
    DiscountPolicy amountPolicy = new AmountBasedDiscountPolicy();
    Order orderWorth501Dollars = orderWorthNDollars(501);

    // when
    double flatDiscount = flatPolicy.discount(orderWorth501Dollars);
    double amountDiscount = amountPolicy.discount(orderWorth501Dollars);

    // then
    assertThat(flatDiscount).isEqualTo(0.01);
    assertThat(amountDiscount).isEqualTo(0.1);
}
```

If this all seems pretty straight-forward, stay tuned. **We'll use the same example later.**

We're now ready to introduce double dispatch.



### 2.2. Double Dispatch vs Method Overloading

**Double dispatch determines the method to invoke at runtime based both on the receiver type and the argument types**.

Java doesn't support double dispatch.

**Note that double dispatch is often confused with method overloading, which is not the same thing**. Method overloading chooses the method to invoke based only on compile-time information, like the declaration type of the variable.

The following example explains this behavior in detail.

Let's introduce a new discount interface called *SpecialDiscountPolicy*:

```java
public interface SpecialDiscountPolicy extends DiscountPolicy {
    double discount(SpecialOrder order);
}
```

*SpecialOrder* simply extends *Order* with no new behavior added.

Now, when we create an instance of *SpecialOrder* but declare it as normal *Order*, then the special discount method is not used:

```java
@DisplayName(
    "given discount policy accepting special orders, " +
    "when apply the policy on special order declared as regular order, " +
    "then regular discount method is used"
    )
@Test
void test() throws Exception {
    // given
    SpecialDiscountPolicy specialPolicy = new SpecialDiscountPolicy() {
        @Override
        public double discount(Order order) {
            return 0.01;
        }

        @Override
        public double discount(SpecialOrder order) {
            return 0.10;
        }
    };
    Order specialOrder = new SpecialOrder(anyOrderLines());

    // when
    double discount = specialPolicy.discount(specialOrder);

    // then
    assertThat(discount).isEqualTo(0.01);
}
```

**Therefore, method overloading is not double dispatch.**

Even if Java doesn't support double dispatch, we can use a pattern to achieve similar behavior: [Visitor](https://en.wikipedia.org/wiki/Visitor_pattern).



### 2.3. Visitor Pattern

**The Visitor pattern allows us to add new behavior to the existing classes without modifying them**. This is possible thanks to the clever technique of emulating double dispatch.

Let’s leave the discount example for a moment so we can introduce the Visitor pattern.

**Imagine we'd like to produce HTML views using different templates for each kind of order**. We could add this behavior directly to the order classes, but it's not the best idea due to being an SRP violation.

Instead, we'll use the Visitor pattern.

First, we need to introduce the *Visitable* interface:

```java
public interface Visitable<V> {
    void accept(V visitor);
}
```

We'll also use a visitor interface, in our cased named *OrderVisitor*:

```java
public interface OrderVisitor {
    void visit(Order order);
    void visit(SpecialOrder order);
}
```

**However, one of the drawbacks of the Visitor pattern is that it requires visitable classes to be aware of the Visitor.**

If classes were not designed to support the Visitor, it might be hard (or even impossible if the source code is not available) to apply this pattern.

Each order type needs to implement the *Visitable* interface and provide its own implementation which is seemingly identical, another drawback.

Notice that the added methods to *Order* and *SpecialOrder* are identical:

```java
public class Order implements Visitable<OrderVisitor> {
    @Override
    public void accept(OrderVisitor visitor) {
        visitor.visit(this);        
    }
}

public class SpecialOrder extends Order {
    @Override
    public void accept(OrderVisitor visitor) {
        visitor.visit(this);
    }
}
```

**It might be tempting to not re-implement accept in the subclass. However, if we didn't, then the OrderVisitor.visit(Order) method would always get used, of course, due to polymorphism.**

Finally, let's see the implementation of *OrderVisitor* responsible for creating HTML views:

```java
public class HtmlOrderViewCreator implements OrderVisitor {
    
    private String html;
    
    public String getHtml() {
        return html;
    }

    @Override
    public void visit(Order order) {
        html = String.format("<p>Regular order total cost: %s</p>", order.totalCost());
    }

    @Override
    public void visit(SpecialOrder order) {
        html = String.format("<h1>Special Order</h1><p>total cost: %s</p>", order.totalCost());
    }

}
```

The following example demonstrates the use of *HtmlOrderViewCreator*:

```java
@DisplayName(
        "given collection of regular and special orders, " +
        "when create HTML view using visitor for each order, " +
        "then the dedicated view is created for each order"   
    )
@Test
void test() throws Exception {
    // given
    List<OrderLine> anyOrderLines = OrderFixtureUtils.anyOrderLines();
    List<Order> orders = Arrays.asList(new Order(anyOrderLines), new SpecialOrder(anyOrderLines));
    HtmlOrderViewCreator htmlOrderViewCreator = new HtmlOrderViewCreator();

    // when
    orders.get(0)
        .accept(htmlOrderViewCreator);
    String regularOrderHtml = htmlOrderViewCreator.getHtml();
    orders.get(1)
        .accept(htmlOrderViewCreator);
    String specialOrderHtml = htmlOrderViewCreator.getHtml();

    // then
    assertThat(regularOrderHtml).containsPattern("<p>Regular order total cost: .*</p>");
    assertThat(specialOrderHtml).containsPattern("<h1>Special Order</h1><p>total cost: .*</p>");
}
```



## 3. Double Dispatch in DDD

In previous sections, we discussed double dispatch and the Visitor pattern.

**We're now finally ready to show how to use these techniques in DDD.**

Let's go back to the example of orders and discount policies.



### 3.1. Discount Policy as a Strategy Pattern

Earlier, we introduced the *Order* class and its *totalCost()* method that calculates the sum of all order line items:

```java
public class Order {
    public Money totalCost() {
        // ...
    }
}
```

There's also the *DiscountPolicy* interface to calculate the discount for the order. This interface was introduced to allow using different discount policies and change them at runtime.

This design is much more supple than simply hardcoding all possible discount policies in *Order* classes:

```java
public interface DiscountPolicy {
    double discount(Order order);
}
```

**We haven't mentioned this explicitly so far, but this example uses the Strategy pattern. DDD often uses this pattern to conform to the Ubiquitous Language principle and achieve low coupling. In the DDD world, the Strategy pattern is often named Policy.**

Let's see how to combine the double dispatch technique and discount policy.



### 3.2. Double Dispatch and Discount Policy

**To properly use the Policy pattern, it's often a good idea to pass it as an argument**. This approach follows the [Tell, Don't Ask](https://martinfowler.com/bliki/TellDontAsk.html) principle which supports better encapsulation.

For example, the *Order* class might implement *totalCost* like so:

```java
public class Order /* ... */ {
    // ...
    public Money totalCost(SpecialDiscountPolicy discountPolicy) {
        return totalCost().multipliedBy(1 - discountPolicy.discount(this), RoundingMode.HALF_UP);
    }
    // ...
}
```

Now, let's assume we'd like to process each type of order differently.

For example, when calculating the discount for special orders, there are some other rules requiring information unique to the *SpecialOrder* class. We want to avoid casting and reflection and at the same time be able to calculate total costs for each *Order* with the correctly applied discount.

We already know that method overloading happens at compile-time. **So, the natural question arises: how can we dynamically dispatch order discount logic to the right method based on the runtime type of the order?**

**The answer? We need to modify order classes slightly.**

The root *Order* class needs to dispatch to the discount policy argument at runtime. The easiest way to achieve this is to add a protected *applyDiscountPolicy* method:

```java
public class Order /* ... */ {
    // ...
    public Money totalCost(SpecialDiscountPolicy discountPolicy) {
        return totalCost().multipliedBy(1 - applyDiscountPolicy(discountPolicy), RoundingMode.HALF_UP);
    }

    protected double applyDiscountPolicy(SpecialDiscountPolicy discountPolicy) {
        return discountPolicy.discount(this);
    }
   // ...
}
```

Thanks to this design, we avoid duplicating business logic in the *totalCost* method in *Order* subclasses.

Let's show a demo of usage:

```java
@DisplayName(
    "given regular order with items worth $100 total, " +
    "when apply 10% discount policy, " +
    "then cost after discount is $90"
    )
@Test
void test() throws Exception {
    // given
    Order order = new Order(OrderFixtureUtils.orderLineItemsWorthNDollars(100));
    SpecialDiscountPolicy discountPolicy = new SpecialDiscountPolicy() {

        @Override
        public double discount(Order order) {
            return 0.10;
        }

        @Override
        public double discount(SpecialOrder order) {
            return 0;
        }
    };

    // when
    Money totalCostAfterDiscount = order.totalCost(discountPolicy);

    // then
    assertThat(totalCostAfterDiscount).isEqualTo(Money.of(CurrencyUnit.USD, 90));
}
```

This example still uses the Visitor pattern but in a slightly modified version. Order classes are aware that *SpecialDiscountPolicy* (the Visitor) has some meaning and calculates the discount.

**As mentioned previously, we want to be able to apply different discount rules based on the runtime type of Order. Therefore, we need to override the protected applyDiscountPolicy method in every child class.**

Let's override this method in *SpecialOrder* class:

```java
public class SpecialOrder extends Order {
    // ...
    @Override
    protected double applyDiscountPolicy(SpecialDiscountPolicy discountPolicy) {
        return discountPolicy.discount(this);
    }
   // ...
}
```

We can now use extra information about *SpecialOrder* in the discount policy to calculate the right discount:

```java
@DisplayName(
    "given special order eligible for extra discount with items worth $100 total, " +
    "when apply 20% discount policy for extra discount orders, " +
    "then cost after discount is $80"
    )
@Test
void test() throws Exception {
    // given
    boolean eligibleForExtraDiscount = true;
    Order order = new SpecialOrder(OrderFixtureUtils.orderLineItemsWorthNDollars(100), 
      eligibleForExtraDiscount);
    SpecialDiscountPolicy discountPolicy = new SpecialDiscountPolicy() {

        @Override
        public double discount(Order order) {
            return 0;
        }

        @Override
        public double discount(SpecialOrder order) {
            if (order.isEligibleForExtraDiscount())
                return 0.20;
            return 0.10;
        }
    };

    // when
    Money totalCostAfterDiscount = order.totalCost(discountPolicy);

    // then
    assertThat(totalCostAfterDiscount).isEqualTo(Money.of(CurrencyUnit.USD, 80.00));
}
```

Additionally, since we are using polymorphic behavior in order classes, we can easily modify the total cost calculation method.

## **4. Conclusion**

In this article, we’ve learned how to use double dispatch technique and *Strategy* (aka *Policy*) pattern in Domain-driven design.

The full source code of all the examples is available [over on GitHub](https://github.com/eugenp/tutorials/tree/master/ddd).



















