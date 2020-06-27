---
layout: post
title: "C++ Bounded Buffer"
description: "Implementing a Thread-Safe Bounded Buffer in C++"
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
Here I will talk through what bounded buffers are and how I implemented one.

## Producer/Consumer Pattern

Before I dive into the bounded buffer itself, let me first explain the main
scenario in which bounded buffers can be useful. The producer/consumer pattern
is a general software pattern which describes the interaction between two
parties, a producer and a consumer.

Consider the following real-world analogy: A donut shop (the producer) generates
donuts constantly while customers (the consumers) purchase donuts throughout the
day. The donut uses traffic from previous days to predict how many donuts are
needed at different times of the day. That being said, the customers themselves
still have a element of unpredictability and as a whole do not purchase donuts
at some exact rate. The shop uses racks to store the donuts, which are being
emptied and replenished continuously.

In this example, we have the following elements:
* Resource: Donuts
* Producer: Donut shop
* Consumer: Customers
* Storage mechanism: Donut racks

This is an example of the producer/consumer scenario. Resources are generated
and used by various parties, while some mechanism holds the resources which are
awaiting consumption. Resources can be produced/consumed at fixed rates,
intermittently, or both. Multiple producers and/or consumers may be present.

Now let's explore an example more relevant to software.


uploading file online
program receiving data from hardware


## Bounded Buffers

In the above donut shop scenario, the donut racks serve as a literal buffer
between the donut-making machines and the customers. They also happen to be
bounded by their size, which determines how many donuts they can hold. They
allow for discrepancies between the production and consumption rates inherently
present in a donut shop. In the same way, bounded buffers are used to solve
problems related to mismatched production/consumption rates among multiple
parties. In practice, a bounded buffer is just a queue with specific access
procedures. Let's take a look at a graphical representation of a bounded buffer:

{% include bounded_buffer.svg %}

First, the buffer has a **capacity** which defines the maximum number of
elements that can be stored. This is why the buffer is called bounded. The
**size** of the buffer, in keeping with the terminology of C++'s standard
containers, is the current number of elements in the buffer.

Since the buffer is a queue, elements are removed in the same order that they
were placed into it: first in, first out (FIFO). Multiple producers and multiple
consumers are permitted. Producers may only insert elements into the buffer,
while consumers may only remove elements from it.

#### Buffer Access Procedures

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

Now let's map these donut shop options to a bounded buffer.

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

The other main complication with bounded buffers is that of **race conditions**.
This is when multiple threads "race" to access a resource at "the same time". If
two threads are attempting to execute a function on a shared piece of data, it's
possible for them to perfectly interleave. This can lead to incorrect results if
not prevented. Let's see an example of this. Given the following function:

{% highlight cpp %}
void increment_global()
{
    int tmp = global_val;
    tmp++;
    global_val = tmp;
}
{% endhighlight %}

If two threads were to execute the code at the function at the same time, the
results could look like this (in pseudocode):

{% highlight cpp %}
int global_val = 1;

int tmp = 1;  // thread A
int tmp = 1;  // thread B
tmp++;  // thread A
tmp++;  // thread B
global_val = 2;  // thread A
global_val = 2;  // thread B --> wrong value!
{% endhighlight %}

Now of course this is a silly pointless function, but the same situation can
happen with nontrivial functions. The solution to this problem, with a bounded
buffer, is to manage access to the buffer with a "lock".

A lock is a construct which can be "acquired" by a thread "atomically". This
means that only one thread can own the lock at a time, and race conditions
cannot occur while acquiring/releasing a lock. If we require that a lock be
obtained before acting on a shared resource, then every instruction is safe
while the lock is held. This enforces the following scenario,
consistently (more pseudocode):

{% highlight cpp %}
int global_val = 1;
lock lk;

// both threads attempt to acquire the lock, thread B succeeds
lock(lk);        // thread B
int tmp = 1;     // thread B
tmp++;           // thread B
global_val = 2;  // thread B
unlock(lk);      // thread B

lock(lk);        // thread A
int tmp = 2;     // thread A
tmp++;           // thread A
global_val = 3;  // thread A
unlock(lk);      // thread A --> correct value!
{% endhighlight %}

The proper term for a lock is
[mutex](https://en.cppreference.com/w/cpp/thread/mutex), which means "mutual
exclusion". A bounded buffer requires one of these mutexes, along with two
"condition variables", which are another type of thread-safety mechanism.
[Condition
variables](https://en.cppreference.com/w/cpp/thread/condition_variable) serve as
alerts that some condition has been met.

When a consumer is attempting to remove data from an empty buffer, for example,
it may choose to wait until some data has been placed into the buffer. When a
producer thread generates a piece of data and puts it into the buffer, it will
alert any waiting consumer threads via a condition variable. The second
conditional variable works in the other direction, alerting waiting producer
threads that data has been removed from a full buffer.

The implementation of these concepts in C++ is detailed below, and of course can
be seen in the library's source code.

## Interface

I decided to base the bounded buffer's interface off that of most standard
library containers, for consistency and ease of use. Since bounded buffers are
queues, it first draws inspiration from the C++
[queue](https://en.cppreference.com/w/cpp/container/queue) container. To that
end, the class has the following access operations:

* `empty`
* `size`
* `front`
* `back`

It also draws inspiration from the
[vector](https://en.cppreference.com/w/cpp/container/vector) container and thus
includes `capacity` and `clear` operations. Lastly, I wanted a
`dropped_elements` access operation which tracks the number of failed push
operations. This is a specific requirement for my use case and can be removed if
you are using the code for your own project.

For pushing/popping functionality, I implemented all of the access solutions
mentioned in the previous section:

1. `try_push`/`try_pop`
2. `push_wait`/`pop_wait`
3. `push_wait_for`/`pop_wait_for`
4. `force_push`

I have not implemented `emplace` or `swap` operations but am open to doing so.
Please make an issue or submit a pull request to the repo if you would like.

## Implementation

Since the implementations of most of the buffer's functions are similar, we'll
dive into just two of them: `try_pop` and `push_wait_for`.

First, `try_pop`:

{% highlight cpp linenos %}
template <typename T>
std::unique_ptr<T> BoundedBuffer<T>::try_pop()
{
    std::lock_guard<std::mutex> lk(m);
    if (q.empty())
    {
        return nullptr;
    }
    auto rv = std::make_unique<T>(q.front());
    q.pop();
    q_has_space.notify_one();
    return rv;
}
{% endhighlight %}

The first order of business is to lock the buffer's mutex to ensure that race
conditions do not occur. Any other threads attempting to interact with the
buffer in the meantime must either wait or fail. Utilizing a `lock_guard` is
easy and enforces [RAII](https://en.cppreference.com/w/cpp/language/raii) by
automatically releasing the lock upon function completion.

Next, we check to see whether there is any data to pop from the buffer. If no
data is present, the function fails immediately and returns a  `nullptr`. This
is a clean way of depicting function failure in C++, but there are many other
ways. The function could throw an exception, return a pair consisting of a bool
and the value, return a `std::optional`, etc.

If there is data in the buffer, the function continues by retrieving the next
value and then popping it from the internal queue. Before returning the value,
it calls a notify function on one of the buffer's internal condition variables.
This notification will allow the next `pop_wait` or `pop_wait_for` function to
be woken up and executed after this function has finished.

Now to look at `push_wait_for`:

{% highlight cpp linenos %}
template <typename T>
bool BoundedBuffer<T>::push_wait_for(
    const T& e,
    std::chrono::milliseconds timeout)
{
    bool success;
    std::unique_lock<std::mutex> lk(m);
    if (q_has_space.wait_for(lk,
            timeout,
            [this]{ return q.size() != cap; }))
    {
        q.push(e);
        success = true;
    }
    else
    {
        dropped++;
        success = false;
    }

    q_has_element.notify_one();
    return success;
}
{% endhighlight %}

This member function starts by locking the buffer's internal mutex, just like
all of the other member functions. It then calls a wait function on one of the
buffer's condition variables, passing in the just-locked mutex. Note that here a
[unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock) is used, as
this is required for the `condition_variable::wait_for` function. It enforces
RAII just like a `lock_guard`, so there's no need to manually unlock the mutex
at the end of the function.

There is some complexity to `condition_variable::wait_for`. First, it must be
passed a mutex which has been locked by the current thread. The mutex must also
be shared by the other threads waiting on the same condition variable. Failure
to meet these conditions results in undefined behavior, as documented in the
[cppreference.com page](https://en.cppreference.com/w/cpp/thread/condition_variable/wait_for).

In addition, `condition_variable::wait_for` has two function signatures. One
accepts just a mutex and a timeout duration, while the other also accepts a
[predicate](https://en.cppreference.com/w/cpp/named_req/Predicate). A predicate
is essentially an expression which evaluates to a boolean result. The purpose of
the predicate is to prevent spurious wakeups, which is a phenomenon present with
condition variables in multiple languages. Spurious wakeups occur when a waiting
thread is woken up when the required condition has not actually been met, in
this when `condition_variable::notify_one` has not yet been called. Double
checking the actual state of the buffer is thus needed to prevent spurious
wakeups from causing bugs.

Now, onto `wait_for`'s return value. If true, the condition has been satisfied
before the timeout expired and thus, in the case of this specific function, an
empty space is available in the buffer for data to be pushed into. If instead
the timeout had expired before an empty space was available, the buffer would
acknowledge this by incrementing `dropped` and set the `success` flag
appropriately. Regardless of the success of this action, the buffer must now
contain at least one element so the other condition variable can be notified.

## Testing and Usage

Testing is done with [GoogleTest](https://github.com/google/googletest) and many
usage examples of the bounded buffer can be found in the
[tests](https://github.com/jdtaylor7/bounded_buffer/blob/master/test/bounded_buffer_test.cpp). Some usage examples are also discussed below.

The tests can be run with [Bazel](https://bazel.build/):

`bazel test --config=linux //test:bounded_buffer`

or with [CMake](https://cmake.org/) via a testing script:

`./test.sh`

Let's look at the first simple usage case. Here we create a bounded buffer
object with a capacity value and add/remove elements via `try_push` and
`try_pop`.

It's important to remember that the pop operations return `unique_ptrs` which
must be moved into a result pointer. Of course one should test whether that
result pointer is null before dereferencing it.

{% highlight cpp linenos %}
#include <cassert>
#include <memory>

#include "bounded_buffer.hpp"

int main()
{
    constexpr std::size_t capacity = 5;
    auto buf = std::make_unique<BoundedBuffer<int>>(capacity);

    buf->try_push(1);
    buf->try_push(2);

    assert(buf->front() == 1);
    assert(buf->back() == 2);

    assert(buf->capacity() == 5);
    assert(buf->size() == 2);
    assert(buf->empty() == false);

    auto result = std::move(buf->try_pop());
    assert(result != nullptr);
    assert(*result == 1);

    result = std::move(buf->try_pop());
    assert(result != nullptr);
    assert(*result == 2);
}
{% endhighlight %}

Here's a more complex example which demonstrates the bounded buffer being used
in a producer/consumer application. Values are transferred from a large source
vector to a large destination vector using a much smaller bounded buffer.

The consumer thread is delayed by one second, while both threads wait for five
seconds before giving up and failing. Even if a failure does occur on a thread,
it will continue to the end. This test should pass on most systems, but the
timeout parameters and buffer size could be tweaked if necessary.

{% highlight cpp linenos %}
#include <cassert>
#include <chrono>
#include <memory>
#include <thread>
#include <vector>

#include "bounded_buffer.hpp"

int main()
{
    using namespace std::chrono_literals;

    constexpr std::size_t buf_cap = 1024;
    constexpr auto timeout = 5s;
    auto buf = std::make_shared<BoundedBuffer<int>>(buf_cap);

    constexpr std::size_t vec_cap = 1'000'000;
    std::vector<int> producer(vec_cap, 0);
    std::vector<int> consumer{};

    for (std::size_t i = 0; i < producer.size(); i++)
        producer[i] = i;

    std::thread t1([&]{
        for (const auto& e : producer)
            assert(buf->push_wait_for(e, timeout) == true);
    });

    std::thread t2([&]{
        std::this_thread::sleep_for(1s);
        while (consumer.size() < vec_cap)
        {
            auto result = buf->pop_wait_for(timeout);
            if (result)
                consumer.push_back(*result));
        }
    });

    t1.join();
    t2.join();

    assert(producer.size() == consumer.size());
    for (std::size_t i = 0; i < consumer.size(); i++)
        assert(consumer[i] == i);
}
{% endhighlight %}

If you don't recognize the syntax used in the thread constructors above, I would
highly recommend you look into
[lambdas](https://en.cppreference.com/w/cpp/language/lambda), as they are quite
convenient in these and other scenarios.

## Conclusion

The code is located [here](https://github.com/jdtaylor7/bounded_buffer).
Suggestions and pull requests are welcome.
