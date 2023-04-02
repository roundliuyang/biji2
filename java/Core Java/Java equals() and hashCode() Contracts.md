# Java equals() and hashCode() Contracts



## **1. Overview**

In this tutorial, we'll introduce two methods that closely belong together: *equals()* and *hashCode()*. We'll focus on their relationship with each other, how to correctly override them, and why we should override both or neither.



## **2. equals()**

The *Object* class defines both the *equals()* and *hashCode()* methods, which means that these two methods are implicitly defined in every Java class, including the ones we create:

```java
class Money {
    int amount;
    String currencyCode;
}
```

```java
Money income = new Money(55, "USD");
Money expenses = new Money(55, "USD");
boolean balanced = income.equals(expenses)
```

We would expect *income.equals(expenses)* to return *true,* but with the *Money* class in its current form, it won't.

**The default implementation of equals() in the Object class says that equality is the same as object identity, and income and expenses are two distinct instances.**



### **2.1. Overriding equals()**

Let's override the *equals()* method so that it doesn't consider only object identity, but also the value of the two relevant properties:

```java
@Override
public boolean equals(Object o) {
    if (o == this)
        return true;
    if (!(o instanceof Money))
        return false;
    Money other = (Money)o;
    boolean currencyCodeEquals = (this.currencyCode == null && other.currencyCode == null)
      || (this.currencyCode != null && this.currencyCode.equals(other.currencyCode));
    return this.amount == other.amount && currencyCodeEquals;
}
```



### **2.2. equals() Contract**

Java SE defines the contract that our implementation of the *equals()* method must fulfill. **Most of the criteria are** **common sense.** The *equals()* method must be:

- *reflexive*: an object must equal itself
- *symmetric*: **x.equals(y) must return the same result as y.equals(x)**
- *transitive*: if *x.equals(y)* and *y.equals(z),* then also *x.equals(z)*
- *consistent*: the value of *equals()* should change only if a property that is contained in *equals()* changes (no randomness allowed)

We can look up the exact criteria in the [Java SE Docs for the *Object* class](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/Object.html).



### **2.3. Violating equals() Symmetry With Inheritance**

If the criteria for *equals()* is such common sense, then how can we violate it at all? Well, **violations happen most often if we extend a class that has overridden equals().** Let's consider a *Voucher* class that extends our *Money* class:

```java
class WrongVoucher extends Money {

    private String store;

    @Override
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof WrongVoucher))
            return false;
        WrongVoucher other = (WrongVoucher)o;
        boolean currencyCodeEquals = (this.currencyCode == null && other.currencyCode == null)
          || (this.currencyCode != null && this.currencyCode.equals(other.currencyCode));
        boolean storeEquals = (this.store == null && other.store == null)
          || (this.store != null && this.store.equals(other.store));
        return this.amount == other.amount && currencyCodeEquals && storeEquals;
    }

    // other methods
}
```

At first glance, the *Voucher* class and its override for *equals()* seem to be correct. And both *equals()* methods behave correctly as long as we compare *Money* to *Money* or *Voucher* to *Voucher*. **But what happens, if we compare these two objects:**

```java
Money cash = new Money(42, "USD");
WrongVoucher voucher = new WrongVoucher(42, "USD", "Amazon");

voucher.equals(cash) => false // As expected.
cash.equals(voucher) => true // That's wrong.
```

**That violates the symmetry criteria of the equals() contract.**



### **2.4. Fixing equals() Symmetry With Composition**

To avoid this pitfall, we should **favor composition over inheritance.**

Instead of subclassing *Money*, let's create a *Voucher* class with a *Money* property:

```java
class Voucher {

    private Money value;
    private String store;

    Voucher(int amount, String currencyCode, String store) {
        this.value = new Money(amount, currencyCode);
        this.store = store;
    }

    @Override
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof Voucher))
            return false;
        Voucher other = (Voucher) o;
        boolean valueEquals = (this.value == null && other.value == null)
          || (this.value != null && this.value.equals(other.value));
        boolean storeEquals = (this.store == null && other.store == null)
          || (this.store != null && this.store.equals(other.store));
        return valueEquals && storeEquals;
    }

    // other methods
}
```

Now *equals* will work symmetrically as the contract requires.



## **3. hashCode()**

*hashCode()* returns an integer representing the current instance of the class. We should calculate this value consistent with the definition of equality for the class. Thus, **if we override the equals()** method, we also have to override *hashCode()*.

For more details, check out our [guide to *hashCode()*](https://www.baeldung.com/java-hashcode).





### **3.1. hashCode() Contract**

Java SE also defines a contract for the *hashCode()* method. A thorough look at it shows how closely related *hashCode()* and *equals()* are.

**All three criteria in the hashCode() contract mention the equals() method in some way:**

- *internal consistency*: the value of *hashCode()* may only change if a property that is in *equals()* changes
- *equals consistency*: **objects that are equal to each other must return the same hashCode**
- *collisions*: **unequal objects may have the same hashCode**



### **3.2. Violating the Consistency of hashCode() and equals()**

The second criteria of the hashCode methods contract has an important consequence: **If we override equals(), we must also override hashCode().** This is by far the most widespread violation regarding the *equals()* and *hashCode()* methods contracts.

Let's see such an example:

```java
class Team {

    String city;
    String department;

    @Override
    public final boolean equals(Object o) {
        // implementation
    }
}
```

The *Team* class overrides only *equals()*, but it still implicitly uses the default implementation of *hashCode()* as defined in the *Object* class. And this returns a different *hashCode()* for every instance of the class. **This violates the second rule.**

Now, if we create two *Team* objects, both with city “New York” and department “marketing,” they will be equal, **but they'll return different hashCodes.**



### **3.3. HashMap Key With an Inconsistent hashCode()**

But why is the contract violation in our *Team* class a problem? Well, the trouble starts when some hash-based collections are involved. Let's try to use our *Team* class as a key of a *HashMap*:

```java
Map<Team,String> leaders = new HashMap<>();
leaders.put(new Team("New York", "development"), "Anne");
leaders.put(new Team("Boston", "development"), "Brian");
leaders.put(new Team("Boston", "marketing"), "Charlie");

Team myTeam = new Team("New York", "development");
String myTeamLeader = leaders.get(myTeam);
```

We would expect *myTeamLeader* to return “Anne,” **but with the current code, it doesn't.**

If we want to use instances of the *Team* class as *HashMap* keys, we have to override the *hashCode()* method so that it adheres to the contract; **equal objects return the same hashCode.**

Let's see an example implementation:

```java
@Override
public final int hashCode() {
    int result = 17;
    if (city != null) {
        result = 31 * result + city.hashCode();
    }
    if (department != null) {
        result = 31 * result + department.hashCode();
    }
    return result;
}
```

After this change, *leaders.get(myTeam)* returns “Anne” as expected.





## **4. When Do We Override equals() and hashCode()?**

**Generally, we want to override either both of them or neither of them.** We just saw in Section 3 the undesired consequences if we ignore this rule.

Domain-Driven Design can help us decide circumstances when we should leave them be. For entity classes, for objects having an intrinsic identity, the default implementation often makes sense.

However, **for value objects, we usually prefer equality based on their properties**. Thus, we want to override *equals()* and *hashCode()*. Remember our *Money* class from Section 2: 55 USD equals 55 USD, even if they're two separate instances.



## **5. Implementation Helpers**

We typically don't write the implementation of these methods by hand. As we've seen, there are quite a few pitfalls.

One common option is to [let our IDE](https://www.baeldung.com/java-eclipse-equals-and-hashcode) generate the *equals()* and *hashCode()* methods.

[Apache Commons Lang](https://www.baeldung.com/java-commons-lang-3) and [Google Guava](https://www.baeldung.com/whats-new-in-guava-19) have helper classes in order to simplify writing both methods.

[Project Lombok](https://www.baeldung.com/intro-to-project-lombok) also provides an *@EqualsAndHashCode* annotation. **Note again how equals() and hashCode() “go together” and even have a common annotation.**



## **6. Verifying the Contracts**

If we want to check whether our implementations adhere to the Java SE contracts, and also to best practices, **we can use the EqualsVerifier library.**

Let's add the [EqualsVerifier](https://mvnrepository.com/artifact/nl.jqno.equalsverifier/equalsverifier) Maven test dependency:

```java
<dependency>
    <groupId>nl.jqno.equalsverifier</groupId>
    <artifactId>equalsverifier</artifactId>
    <version>3.0.3</version>
    <scope>test</scope>
</dependency>
```

Now let's verify that our *Team* class follows the *equals()* and *hashCode()* contracts:

```java
@Test
public void equalsHashCodeContracts() {
    EqualsVerifier.forClass(Team.class).verify();
}
```

It's worth noting that *EqualsVerifier* tests both the *equals()* and *hashCode()* methods.

**EqualsVerifier is much stricter than the Java SE contract.** For example, it makes sure that our methods can't throw a *NullPointerException.* Also, it enforces that both methods, or the class itself, are final.

It's important to realize that **the default configuration of EqualsVerifier allows only immutable fields**. This is a stricter check than what the Java SE contract allows. It adheres to a recommendation of Domain-Driven Design to make value objects immutable.



If we find some of the built-in constraints unnecessary, we can add a *suppress(Warning.SPECIFIC_WARNING)* to our *EqualsVerifier* call.



## **7. Conclusion** 

In this article, we discussed the *equals()* and *hashCode()* contracts. We should remember to:

- Always override *hashCode()* if we override *equals()*
- Override *equals()* and *hashCode()* for value objects
- Be aware of the traps of extending classes that have overridden *equals()* and *hashCode()*
- Consider using an IDE or a third-party library for generating the *equals()* and *hashCode()* methods
- Consider using EqualsVerifier to test our implementation

Finally, all code examples can be found [over on GitHub](https://github.com/eugenp/tutorials/tree/master/core-java-modules/core-java-lang-oop-methods).















