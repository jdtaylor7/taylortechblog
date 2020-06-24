---
layout: post
title: "C++ Bounded Buffer"
description: "Implementing a Thread-Safe Bounded Buffer in C++"
tags: [cpp, concurrency]
---

As part of my autonomous drone project, I created a visualization program which
displays telemetry from the drone in real-time and displays the drone's
corresponding position and orientation in space. This program required a
mechanism to manage the flow of data from the telemetry receiver and the run
loop of the visualizer.

To address this, I implemented a thread-safe bounded buffer in C++ with
different sets of interfaces to satisfy different use-cases. The data structure
is fully tested with [GoogleTest](https://github.com/google/googletest) and
located [here](https://github.com/jdtaylor7/bounded_buffer) as a header-only
library. Here I will talk through the implementation decisions I made, along
with how I approached testing

### Producer/Consumer Pattern

The producer/consumer pattern is a general software pattern which describes the
interaction between two parties, a producer and a consumer. Consider the
following restaurant analogy: The kitchen prepares meals at some average rate.
Customers order meals at another, likely different, average rate. Meals are
represented by receipts which ensure meals are processed in a timely manner and
not lost. Ticket holders and computer systems bolster this meal tracking effort.

In this example, we have the following elements:
* Resource: Meals
* Producer: Kitchen
* Consumer: Customers
* Production rate: Rate meals are made
* Consumption rate: Rate meals are ordered
* Receipts, ticket holders, computers: Tracking system

This is the core of the producer/consumer scenario. Resources are generated and
used at different rates and some system must be in place in order to manage said
resources effectively.

### Bounded Buffers

In the above restaurant scenario, the receipt tracking system is the "solution"
to the meal tracking "problem". In software, the generalized solution to the
producer/consumer problem is a bounded buffer. A bounded buffer is just a queue
with specific access methods. Let's take a look at a graphical representation of
a bounded buffer:

{% include bounded_buffer.svg %}

First, the buffer has a **capacity** which defines the maximum number of
elements that can be stored. This is why the buffer is called "bounded". The
**size** of the buffer, in keeping with the terminology of C++'s standard
containers, is the current number of elements in the buffer.

Since the buffer is a queue, elements are removed in the same order that they
were placed into it: first in, first out (FIFO). Multiple producers and multiple
consumers are permitted. Producers may only insert elements into the buffer,
while consumers may only remove elements from it.

### Complications

Before we continue, there are two main complications which must be addressed:

1. Adding elements to a full buffer (and removing elements from an empty buffer)
2. Simultaneous access by multiple entities

There are multiple ways to address the first complication of adding data to a
full buffer. Of course it would be possible to use additional data structures to
catch overflow data, but for the sake of this post let's assume we don't want to
do that. With that in mind, there are four main solutions:

1. Do not add the element and mark the operation as failed
2. Wait until there is space to add the element
3. Wait some set amount of time for space to be created, then fail if the time
expires
4. Create space by removing an element, then add the element

The first three solutions can also be applied to the corresponding situation of
removing elements when the buffer is empty.

The second complication is called a **race condition**, when two or more threads
attempt to access a piece of data at the same time. Since bounded buffers are
commonly used in multi-threaded cases, this is a common requirement for them. In
short, thread-safety mechanisms must be employed to address this.

Exploring all of the potential thread-safety mechanisms is beyond the scope of
this post, but suffice it to say that a single
[mutex](https://en.cppreference.com/w/cpp/thread/mutex) and two [condition
variables](https://en.cppreference.com/w/cpp/thread/condition_variable) are
sufficient for a bounded buffer. The buffer's mutex must be locked by a thread
before any of its class methods can be executed. The condition variables serve a
different purpose, ensuring thread safety with regards to the buffer being empty
or full. The implementation of these concepts in C++ is detailed below, and of
course can be seen in the library's source code.

### Interface

I decided to base the bounded buffer's interface off that of most standard
library containers, for consistency and ease of use. Specifically, the
[queue](https://en.cppreference.com/w/cpp/container/queue) is the most similar
C++ container to the bounded buffer. To that end, the class has the following
access operations:

* `empty`
* `size`
* `front`
* `back`

I also drew inspiration from the
[vector](https://en.cppreference.com/w/cpp/container/vector) container and thus
included `capacity` and `clear` operations. Lastly, I wanted a
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

### Implementation

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
conditions do not occur. No other threads will be able to perform any actions on
the bounded buffer, so we know the state of the buffer at the beginning of the
function is the same at the end of the function. Utilizing a `lock_guard` is
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
be woken up, if one is waiting. The wake up will actually occur after the current
`try_pop` function completes and unlocks the mutex, not immediately.

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
buffer's condition variables, passing in the just-locked mutex.

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
condition variables outside of C++.

Now, onto `wait_for`'s return value. If true, the condition has been satisfied
before the timeout expired and thus, in the case of this specific function, an
empty space is available in the buffer for data to be pushed into. If instead
the timeout had expired before an empty space was available, the buffer would
acknowledge this by incrementing `dropped` and set the `success` flag
appropriately. Regardless of the success of this action, the buffer must now
contain at least one element so the other condition variable can be notified.

### Usage

Many usage examples are present in the
[tests](https://github.com/jdtaylor7/bounded_buffer/blob/master/test/bounded_buffer_test.cpp). Let's but we'll look at one of those tests here.

### Testing

The bounded buffer has been tested in a number of ways, using [GoogleTest](https://github.com/google/googletest):

* BasicFuncTest: Basic functionality of each member function
* UnstickPushTest: Ensure `push_wait_for` correctly waits for space to be made in
the buffer
* UnstickPopTest: Same as above for `pop_wait_for`
* ThroughputTest:
* ThroughputTestTimeout:
* ForcePushTest: `BoundedBuffer::force_push`

### Conclusion

The code is located here [here](https://github.com/jdtaylor7/bounded_buffer).
Suggestions and pull requests are welcome.

### References
1. \
2. \
3. \
