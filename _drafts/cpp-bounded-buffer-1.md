---
layout: post
title: "C++ Bounded Buffer Part 1"
description: "Introduction to Bounded Buffers"
tags: [cpp, concurrency]
---

As part of an autonomous drone project I'm working on, I created a program which
displays the drone's position and orientation in real-time. This program
required a mechanism to manage the flow of data coming from a telemetry
receiver.

To address this, I implemented a thread-safe bounded buffer in C++ with multiple
interfaces to satisfy different use-cases. The data structure is fully tested
with [GoogleTest](https://github.com/google/googletest) and is available [on
Github](https://github.com/jdtaylor7/bounded_buffer) as a header-only library.

This post discusses the background and motivation of bounded buffers, along with
some of the requirements for an implementation and necessary multithreading
concepts. The next post, part 2, will dive into the buffer interface and
implementation.

## Producer/Consumer Pattern

Before I dive into the bounded buffer itself, let me first explain the main
scenario in which bounded buffers can be useful. The producer/consumer pattern
is a general software pattern which describes the interaction between two
parties, a producer and a consumer.

Consider the following real-world analogy: A donut shop (the producer) generates
donuts continuously while customers (the consumers) purchase donuts throughout
the day. The donut uses traffic from previous days to predict how many donuts
are needed at different times of the day. That being said, the customers
themselves still have a element of unpredictability and as a whole do not
purchase donuts at some exact rate. The shop uses racks to store the donuts,
which are being emptied and replenished continuously.

In this example, we have the following elements:
* Resource: Donuts
* Producer: Donut shop
* Consumer: Customers
* Storage mechanism: Donut racks

This is an example of the producer/consumer scenario. Resources are generated
and used by various parties, while some mechanism holds the resources which are
awaiting consumption. Resources can be produced/consumed at fixed rates,
intermittently, or both. Multiple producers and/or consumers may be present.

The producer/consumer problem arises in multiple areas of software. Some
examples:

* An operating system managing packet flow through network interfaces
* A hardware device passing received data to a software application for
processing, such as a thermostat passing temperature data to a home automation
system
* A game server receiving data from multiple clients and updating the game
environment accordingly

## Bounded Buffers

In the above donut shop scenario, the donut racks serve as a literal buffer
between the donut-making machines and the customers. They also happen to be
bounded by their size, which determines how many donuts they can hold. They
allow for discrepancies between the production and consumption rates inherently
present in a donut shop. In the same way, bounded buffers are used to solve the
general problem related to mismatched production/consumption rates among
multiple parties.

In practice, a bounded buffer is just a queue with specific access procedures.
Let's take a look at a graphical representation of a bounded buffer:

{% include bounded_buffer.svg %}

First, the buffer has a **capacity** which defines the maximum number of
elements that can be stored. This is why the buffer is called bounded. The
**size** of the buffer, in keeping with the terminology of C++'s standard
containers, is the current number of elements in the buffer.

Since the buffer is a queue, elements are removed in the same order that they
were placed into it: first in, first out (FIFO). Multiple producers and multiple
consumers are permitted. Producers may only insert elements into the buffer,
while consumers may only remove elements from it.

#### Bounded Buffer Input/Output

Bounded buffers have a few complications, the first of which is controlling
producer/consumer access to the data. Namely, what to when adding data to a full
buffer or removing data from an empty buffer. Remembering the donut shop
example, what should be done when more donuts are made than can be held in the
storage racks? Conversely, what should a customer do when no donuts are
available in the storage racks? To answer the first problem, the theoretical
solutions are:

1. Discard the extra donuts
2. Pause the donut assembly line until space is available
3. Wait some finite amount of time for space to become available, but start
throwing donuts out when that time passes
4. Discard the oldest donuts from the racks to make space, then add the new
donuts

In real life, may not be sufficient. In an actual donut shop, additional
solutions would likely be employed. More storage would be used, donut production
would be slowed, and donut production would likely be slowed under similar
future situations to prevent the problem from occurring again.

These additional measures may or may not be under one's control in a software
system, but regardless they are separate from the bounded buffer itself. The
fourth option above would likely be used in concert with the additional
solutions mentioned in the previous paragraph.

Now let's look at a customer's options when no donuts are ready:

1. Leave the donut shop
2. Wait indefinitely for donuts to be available
3. Wait a finite amount of time for donuts to be ready, then leave once this time
passes.

In real life, all but the second option are feasible. These options are the
consumer's versions of the first three producer's options, while there is no
consumer option corresponding to "discarding the oldest and using the newest
one".

Finally, let's map these donut shop options to those for a bounded buffer:

For the producer:

1. Do not add the element and mark the operation as failed
2. Wait until there is space to add the element
3. Wait some set amount of time for space to be created, fail if the time
expires
4. Create space by removing an element, then add the element

For the consumer:

1. Do not remove any elements and mark the operation as failed
2. Wait until an element is present
3. Wait a set amount of time for an element be available, fail if the time
expires

#### Race Conditions

Another complication regarding bounded buffers is that of **race conditions**. A
race condition occurs when multiple threads "race" to access a resource at "the
same time". If two threads are attempting to execute a function at the same
time, it's possible that one function will start executing while the other is
already running. This can lead to incorrect results if not explicitly
disallowed. Let's see an example of this. Given the following function:

{% highlight cpp %}
int global_val;

void increment_global()
{
    int tmp = global_val;
    tmp++;
    global_val = tmp;
}
{% endhighlight %}

If two threads were to execute the function at the same time, the results could
look like this (in pseudocode):

{% highlight cpp %}
int global_val = 1;

int tmp = 1;  // thread A
int tmp = 1;  // thread B
tmp++;  // thread A
tmp++;  // thread B
global_val = 2;  // thread A
global_val = 2;  // thread B --> wrong value!
{% endhighlight %}

As you can see, the code of the two functions have become interleaved. Thread B
is allowed to execute the function while thread A is still running the same
function, resulting in thread B deducing the wrong value for `tmp` (it should be
2 instead of 1). To be clear, the issue is not quite that two of the same
function are being run in parallel - the same problem could arise if the two
threads were running different functions. The real problem is that both threads
access the same shared data (`global_val`) without a care in the world. Some
kind of **access control** must be provided for the shared data. This is
achieved with a **lock**, also called a
[**mutex**](https://en.cppreference.com/w/cpp/thread/mutex) (short for mutual
exclusion).

A mutex can be compared to the lock on a porta potty. When approaching the
stall, you first check the condition of the lock. If the lock is red, it means
that someone is already using the toilet and you must wait. If the lock is
green, the stall is unoccupied and access can be obtained. Just walking into the
porta potty is insufficient, though. Upon entering the bathroom you must lock
the stall, setting it to the occupied state so others know not to enter. Mutexes
are similar in the sense that they are technically optional and may not always
end up being required (for example, maybe no one tried to enter the bathroom
while you were using it), but not using them can result in some unfortunate
situations.

There is some additional terminology concerning mutexes. They are said to be
**acquired** and **released** when a thread obtains them or gives them up
(locking and unlocking the bathroom stall, accordingly). Their operation is also
said to be **atomic**, which means that no instructions from other threads can
interrupt their operations. Other "normal" data types like regular Booleans are
not atomic and thus cannot guarantee the prevention of race conditions like
mutexes can.

If we require that a lock be obtained before acting on a shared resource, then
every instruction is safe while the lock is held. This enforces the following
scenario, consistently (more pseudocode):

{% highlight cpp %}
int global_val = 1;
mutex global_val_mtx;

// both threads attempt to acquire the lock, thread B succeeds (could have been
// thread A)
lock(global_val_mtx);    // thread B
int tmp = 1;             // thread B
tmp++;                   // thread B
global_val = 2;          // thread B
unlock(global_val_mtx);  // thread B

lock(global_val_mtx);    // thread A
int tmp = 2;             // thread A
tmp++;                   // thread A
global_val = 3;          // thread A --> correct value!
unlock(global_val_mtx);  // thread A
{% endhighlight %}

All of this discussion is to say the following: a thread-safe bounded buffer
implementation requires a mutex to provide access control to the underlying
data. Proper mutex usage means that multiple producers/consumers are able to
interact with the bounded buffer without causing problems. While a
non-thread-safe bounded buffer would technically not require such a mechanism,
this kind of bounded buffer would have very limited utility if threading was
disallowed.

#### Condition Variables

For bounded buffers, there is one additional construct required for correct
operation. This is called either a signal or a [condition
variable](https://en.cppreference.com/w/cpp/thread/condition_variable).
Condition variables threads to communicate that some condition has been met.

In a bounded buffer, this is necessary when one or more consumers are waiting
for data to be placed into an empty buffer. Instead of constantly polling the
buffer, which would use CPU cycles and not guarantee which consumer gets to
claim the next piece of inputted data, all consumer threads will instead wait on
a shared condition variable (in a queue). Producer threads manually notify the
condition variable after putting data into the buffer. The condition variable
will then wake up the next waiting consumer thread automatically.

Remembering back to the donut shop example, this would be akin to customers
sitting down and waiting for donuts to become available (if no donuts are ready)
instead of having to keep looking themselves. Once a donut is made, the customer
waiting the longest is told.

The same pattern works in the other direction: producer threads are put into a
queue when the buffer is full, while consumer threads notify a second condition
variable when empty space becomes available in the buffer.

## Conclusion

Now that we've covered the motivation for implementing bounded buffers and some
of the high-level features associated with them, we are ready to get into the
weeds of an actual implementation. That will be covered in part 2. We will
discuss the API design, the C++ implementations of mutexes/condition variables,
and the source code of some of the bounded buffer functions.

[Here's](https://github.com/jdtaylor7/bounded_buffer) the repository.
Suggestions and pull requests are welcome.
