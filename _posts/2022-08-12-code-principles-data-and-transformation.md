---
layout: post
title: Code Principle&#58; Data & Transformation
excerpt: "We discuss how thinking software design in terms of data and transformation helps improve over all software design."
comments: false
tags:
  - code
  - software design
  - code principles
---

Hello there. It’s been a while since my last post. There goes my goal of writing at least one article every month. This time it’s a lot more theoretical, as opposed to my previous pile of technical turd. So let us begin.

## Introduction

Most often we tend to only think about code in terms of logic or structure. When designing software, we often worry about languages and frameworks, data structure and algorithms, and classes and modules. We tend to forget about one of the most important aspects of software – `Data` and `Transformation`. Let us explore how taking `Data` and `Transformation` into account ultimately leads to a better software design.

We shall begin by forming a concise yet flexible criteria for 'good design'.

## What is Good Software Design ?

Software – as its name suggests – is meant to change and extend. It is 'soft', unlike hardware, which is generally meant to be fixed (hard). Software is expected to be susceptible to changes. When the requirements change, it falls upon us the developers to make sure that our software effectively adapts to the proposed changes. This often boils down to diving into the code base and making required changes. Based on this, we can argue that good software design is design that is susceptible to change - i.e. easier to change.

There are many patterns, principles and guidelines that help in designing software that is easier to change. Taking `Data Transformation` into account while designing is one such guideline that we’ll be exploring today.

## Data & Transformation

`Data` is a vital part of any program. A program, in a basic sense, is a set of instructions that operates, communicates, processes, stores and/or presents data. When data flows through a program, it passes through operations that might introduce changes to the data in different ways, which can be defined as a `Transformation`. When a program operates on or changes a piece of data, we can basically say that the program transforms the data.

<p class="message">
<i>Programming is About Code, But Programs Are About Data.</i>
<br />
- The Pragmatic Programmer
</p>

```typescript
// A basic example of data & transformation

const dataA = [1203, 2123, 2134, 2323];

const dataB = dataA.map((x) => x.toString());
```

Here, the array of numbers `dataA` is transformed into the array of strings `dataB` via `map`, which is a function that defines the transformation.

## Thinking in Terms of Transformation

We can start thinking of a program as a series of transformations. The data passes through one or more transformations, each one of which operates on the data in order to produce the desired output. Let’s make this concept clear with a quick example.

```typescript
// A program that takes a user.csv file, parses it.
// And sends notification to the user while logging failures if any.

const users = CSV.parse("user.csv");
// filter users with valid email
const usersWithValidEmail = users.filter((u) => u.email);
// extract email array
const emails = usersWithValidEmail.map((u) => u.email);
// send notification
const results = sendNotification(emails);
// filter failure
const failures = result.filter((r) => !r.success);
```

The `user.csv` file here contains our data. Our program basically does 4 operations on this data:

1. Acquire through parse,
2. Filter valid emails,
3. Send notification to those emails,
4. Filter failures.

To carry out these operations, we move our data through a series of transformations until we reach our end goal.

This approach in programming – where we chain multiple transformations by passing one output as input to another is known as pipelining. Pipelining is mostly provided as a feature in functional languages, where a pipeline operator `|>` automatically pipes output of one transformation as input to another.

```elixir
CSV.parse('user.csv')
  |> Enum.filter(& !is_nil(&1.email))
  |> Enum.map(& &1.email)
  |> send_notification()
  |> Enum.filter(& !&1.success)
```

This is same example but with the pipe operator. Another way to do this kind of chaining is with composition. Composition is somewhat similar to pipelining but true to its mathematical meaning.

The goal is not to hoard state, but to pass them around and have our code conduct required operations, which in turn produce new state that’s essentially derived from the old state. Note that we try not to mutate state, but instead we derive new state from the previous one.

It’s always a good idea to avoid mutation whenever possible. Avoiding mutation deserves its own separate article, but I digress.

Since 'Thinking in terms of transformation' is a mouthful, let’s just call this `Transformative Programming`.

## Transformative Programing a Good Design Approach

As we have learned, good software design is design that’s easier to change, but we haven’t yet explored what makes bad software design.
Or to rephrase, **what makes a software difficult to change?**

> `Coupling`

<p class="message">
<i>Coupling ties things together, so that it's harder to change just one thing.</i>
<br />
- The pragmatic programming
</p>

`Transformative Programming` helps reduce coupling. Transformations are by design independent of each other. In fact, a transformation doesn’t even need to know about the existence of any other transformation. A transformation is only concerned with a specific operation that needs to be performed on its input.

Besides reduced coupling, `Transformative Programming`  provides other goodies, such as increased code readability and better DRY (don’t repeat yourself) (don’t repeat yourself). As the program is divided into similar well-defined transformations, readability increases. The fact that the code is being broken into smaller transformations means that they can be effectively reused as needed.

Let’s look at an example of a method `sendNotification`, which is responsible for sending notifications to a given user.

```typescript
function sendNotification(notification, user) {
  const results = [];
  const devices = user.getDevices();
  // loop through user devices
  for (device of devices) {
    const registrationId = device.registrationId;
    // pre-process notification content
    const body = processTemplate(notification.body, user);
    const title = processTemplate(notification.title, user);
    // queue notification
    const result = NotificationClientApi.queueNotification(registrationId, {
      body: body,
      title: title,
    });
    // store result
    results.push(result);
  }

  return results;
}
```

Now let’s write this code with a transformative approach.

```typescript
// A function to pre-process notification content,
function processNotificationContent(notification, user) {
  const body = processTemplate(notification.body, user);
  const title = processTemplate(notification.title, user);

  return { body: body, title: title };
}

// A function that actually queues the notification, using some client library.
function queueNotification(registrationIds, notificationContent) {
  return registrationIds.map((regId) =>
    NotificationClientApi.queueNotification(regId, notificationContent)
  );
}

function sendNotification(notification, user) {
  // 1 - processNotificationContent
  const notificationContent = processNotificationContent(notification, user);
  // 2 - get registrationIds
  const registrationIds = user.getDevices().map((d) => d.registrationId);
  // 3 - queue notification
  return queueNotification(registrationIds, notificationContent);
}
```

We can immediately notice that the code is a lot more readable, along with better separation of concern, as each transformation only applies a specific operation on the data. The code has become much easier to change.

For example, if we add an additional templated field on the notification object, just by having a glance at the piece of code, we find that the only thing that needs to be updated is `processNotificationContent`.

## Transformative Programming in Real World

At this point, it’s possible to develop the impression that transformative programming is only applicable to functional programming. We will discover this to not be the case, since transformative programming can find its way into your code base and improve it regardless of the style you follow.

In fact, you might already be using aspects of transformative programming, with utility functions like `map`, `reduce`, `filter` etc. that most programming languages now provide within their standard library.

Let's have a look at yet another contrived example:

> Leys say, we have an e-commerce application, and need to calculate discounts for product orders in a shopping cart, based on a known fixed discount value. The discount might be applied to select orders or the whole cart (all orders).

```typescript

// Example:1 - Normal OOP
class Order {
  /* Everything that a typical order contains */

  // Applies discount and calculates final cost
  function calculateCostWithDiscount(discountValue) {
      // calculate new cost
      const newCost = this.cost - discountValue;
      // check if its within range
      return (newCost > this.lowestPossiblePrice)? newCost : this.lowestPossiblePrice;
  }

  // Applies discount to the order, mutating the order
  function applyDiscount(discountValue) {
     this.costAfterDiscount = calculateCostWithDiscount(discountValue);
  }
}

class Cart {
  /* Everything that a typical order contains */

  // Calculates total cost with discount applied to all orders in the card
  function calculateTotalCostWithDiscount(discountValue) {
    const totalDiscount = 0;
    // calculate cost after the discount has been applied
    for(const order of this.orders) {
      totalDiscount += order.calculateCostWithDiscount(discountValue);
    }
    return totalDiscount;
  }

  // Applies discount to all orders in the cart, mutating the cart
  function applyDiscount(discountValue) {
    // apply discount to each order, mutating the order in the process
    for(const order of orders) {
      this.order.applyDiscount(discountValue);
    }
  }

  // Applies discount to all orders in the cart-
  // which match the given orderIds, mutating the cart
  function applyDiscountOrders(orderIds, discountValue) {
    // iterate through each orders in the cart
    for(const cartOrder of this.orders) {
      // iterate through applicable orders
      for(const applicableOrderId of orderIds) {
        // apply discount if the order is applicable
        if(cartOrder.id == applicableOrderId) {
          cartOrder.applyDiscount(discountValue);
        }
      }
    }
  }
}
```

Here we have a basic cart-order system written with an Object-Oriented approach, which basically calculates or applies discount to the orders in cart. Everything is well-contained and the coupling is managed, but we could improve it with some `Transformative Programming`.

```typescript
// Example:2 - OOP with aspect of transformation
class Order {
  /* Everything that a typical order contains */

  function calculateCostWithDiscount(discountValue) {
    // we leverage Math.max instead of comparing values
    return Math.max(this.cost - discountValue, this.lowestPossiblePrice);
  }

  function applyDiscount(discountValue) {
    this.costAfterDiscount = calculateCostWithDiscount(discountValue);
  }
}

class Cart {
  /* Everything that a typical cart contains */

  function calculateTotalCostWithDiscount(discountValue) {
    // return this.orders.reduce((acc, curr) => acc + curr.calcluateDiscount(discountValue), 0);

    // 1 - we first transform orders into number (cost with discount)
    const discounts= this.orders.map((order) => order.calculateCostWithDiscount(discountValue))
    // 2 - we transform the numbers using reduce to a single summed value
    return discounts.reduce((acc, curr) => acc + curr, 0);
  }

  function applyDiscount(discountValue) {
    this.orders.forEach((order) => order.applyDiscount(discountValue));
  }

  function applyDiscountOnOrders(orderIds, discountValue) {
    // 1 - we transform cart orders into applicableOrders using filter + find
    const applicableOrders = this.orders.filter((cartOrder) => orderIds.find((applicableOrderId) => cartOrder == applicableOrderId));
    // 2 - apply discount to each order, mutating in on the process
    applicableOrders.forEach((order) => order.applyDiscount(discountValue));
    // 3 - apply another transformation to calculate total cost
    return applicableOrders.reduce((acc, curr) => acc + curr.costAfterDiscount, 0);
  }
}
```

Now the code is much more readable. It can be further improved by minimizing mutations and switching to structured types, which in-turn makes our code reusable in a way that wasn’t possible before. Also, I recommend using something like [lodash](https://lodash.com/) to compensate for lack of proper utilities in javascript.

```typescript
/**
 * order
 **/
type Order = {
  /* Everything that a typical order contains */
};

// Applies discount to the order, without mutating the order
function orderWithDiscount(order: Order, discountValue: number): Order {
  const costAfterDiscount = Math.max(
    this.price - discountValue,
    this.lowestPossibleCost
  );

  // return a new order with discount applied
  return clone(this, {
    costAfterDiscount: costAfterDiscount,
    discountValue: discountValue,
  });
}

function orderCostAfterDiscount(order): number {
  return order.costAfterDiscount;
}

/**
 * Cart
 **/
type Cart = {
  /* Everything that a typical cart contains */
};

// Calculate total cost including the discount
function cartCalcTotalCostWithDiscount(
  cart: Cart,
  discountValue: number
): number {
  // Since orderWithDiscount doesn't modify the order, we can use to to calculate total
  const orders = cart.orders.map((order) =>
    orderWithDiscount(order, discountValue)
  );
  return orders.reduce((acc, curr) => acc + orderCostAfterDiscount(order), 0);
}

// Applies discount to orders in a cart, without mutating the cart
function cartWithDiscount(cart: Cart, discountValue: number): Cart {
  const orders = cart.orders.map((order) =>
    orderWithDiscount(order, discountValue)
  );

  // return a new cart with updated orders
  return clone(cart, { orders: orders });
}

// using lodash
// Applies discount to orders that match the give order ids,
// without mutating the supplied cart
function cartWithDiscountOnOrders(
  cart: Cart,
  orderIds: OrderId[],
  discountValue: number
): Cart {
  // 1 - filter applicable orders
  const applicableOrders = _.intersectionBy(cart.orders, orders, "id");
  // 2 - apply discount to applicable orders
  const discountedOrders = _.map(applicableOrders, (order) =>
    orderWithDiscount(order, discountValue)
  );
  // 3 - create a set of orders - applicable orders
  const filteredOrders = _.difference(cart.orders, applicableOrders);

  // return new cart with updated orders
  return clone(cart, { orders: _.concat(filteredOrders, applicableOrders) });
}
```

Let’s add some additional functionality – like payment processing, and conditions for discounts to be applicable, which are just more transformations that contains steps required to process a payment.

> The discount should be applied only if the total cost of the cart is greater than a certain threshold.

```typescript
function processPayment(cart: Cart, payment: Options): Cart {
  const totalCost = cart.orders.map((order) => orderCostWithDiscount(order))
  /**
   * Apply necessary transformations, call payment service APIs
  **/
  const paymentStatus = ...;
  return clone(cart, {paymentStatus: paymentStatus})
}


/**
 Apply discount and process payment
**/
const config = ..;
const paymentOptions = ..;

// 1 - calculate total cost
const totalCost = cart.orders.reduce((acc, curr) => acc + curr.cost, 0);
// 2 - check if discount is applicable
const discountValue = (totalCost > config.discountThreshold)? config.discountValue : 0
// 3 - apply the discount
const discountedCart = cartWithDiscount(cart, constrainedDiscountValue);
// 4- process payment
const paymentProcessedCart = processPayment(cart, paymentOptions);
/**
  Some where down the line, we save our most recent state
 */
db.presist(paymentProcessedCart);
```

Here, we are able to represent our business logic in terms of data and transformation, where each transformation is self-descriptive and isolated. There is no mutation; each transformation leads to a new state that could be utilized by the caller as they prefer, like when we use `orderWithDiscount` inside `cartCalcTotalCostWithDiscount` in order to calculate total discount value, while avoiding mutation of orders in the existing cart.

## Conclusion

We now understand that thinking in terms of data transformation leads to a code that is easier to change and understand. Even though transformative programming has its roots in functional programming we can easily adapt its aspects to any form programming approach.

## References

- _The Pragmatic Programmer by David Thomas & Andrew Hunt_
