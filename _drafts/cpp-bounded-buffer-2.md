---
layout: post
title: "C++ Bounded Buffers, Part 2"
description: "Implementing a Thread-Safe Bounded Buffer in C++"
tags: [cpp, multithreading]
---

This is the second of two blog posts about bounded buffers. In [part
1](https://www.taylortechblog.com/posts/cpp-bounded-buffer-1) we discussed the
types of problems bounded buffers can solve and how bounded buffers function at
a high level. Here we will discuss the interface for a bounded buffer, my
implementation, and how to use my implementation. For reference the code is
located [here](https://github.com/jdtaylor7/bounded_buffer).

## Interface

I decided to base the bounded buffer's interface off that of most C++ standard
library containers, for consistency and ease of use. Since bounded buffers are
queues, it primarily draws inspiration from the C++
[queue](https://en.cppreference.com/w/cpp/container/queue) container. To that
end, the class has the following access operations:

* `empty`
* `size`
* `front`
* `back`

It also draws inspiration from the
[vector](https://en.cppreference.com/w/cpp/container/vector) container by
including the `capacity` and `clear` operations. Lastly, I wanted a
`dropped_elements` access operation which tracks the number of failed push
operations. This is a specific requirement for my use case and can be removed if
you are using the code for your own project.

For pushing/popping functionality, I implemented all of the access solutions
mentioned back in part 1:

1. `try_push`/`try_pop`
2. `push_wait`/`pop_wait`
3. `push_wait_for`/`pop_wait_for`
4. `force_push`

`try_push` and `push_wait_for` are the specific operations tracked by
`dropped_elements`.

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

The first order of business is to lock the buffer's mutex (line 4) to ensure
that race conditions do not occur, as discussed in part 1. Any other threads
attempting to interact with the buffer in the meantime must either wait or fail.
Utilizing a `lock_guard` is easy and enforces
[RAII](https://en.cppreference.com/w/cpp/language/raii) by automatically
releasing the lock upon function exit.

Next, we check to see whether there is any data to pop from the buffer (line 5).
If no data is present, the function fails immediately by returning a  `nullptr`
(line 7). This is a clean way of depicting function failure in C++, but there
are many other ways. The function could throw an exception, return a pair
consisting of a bool and the value, return a `std::optional`, etc.

If the function has not failed, it continues by retrieving the next value (line
9) and then popping it from the internal queue (line 10). Before returning the
value (line 12), it calls a notify function on the internal condition variable
`q_has_space` (line 11). This notification will allow the next `push_wait` or
`push_wait_for` function to be woken up and executed after this function has
finished.

*Quick note: Pop operations in queues/stacks sometimes only remove an element
from the data structure, without returning said value to the caller. Sometimes
they do both, as is the case with my implementation.*

Now let's look at `push_wait_for`:

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

This member function starts by locking the buffer's internal mutex (line 7),
just like all of the other member functions. It then calls a wait function on
one of the buffer's condition variables (lines 8-10), passing in the just-locked
mutex. Note that here a
[unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock) is used to
lock the mutex, as is required for the `condition_variable::wait_for` function.
It enforces RAII just like a `lock_guard`, so there's no need to manually unlock
the mutex at the end of the function.

There is some complexity to `condition_variable::wait_for`, so the next few
paragraphs will discuss the function in depth. First, one of its required
arguments is a mutex which has been locked by the current thread. Second, the
mutex must be shared by the other threads waiting on the same condition
variable. Failure to meet these conditions results in undefined behavior, as
documented in the [cppreference
page](https://en.cppreference.com/w/cpp/thread/condition_variable/wait_for).

In addition, `wait_for` has two function signatures. One accepts a mutex and a
timeout duration, while the other also accepts a
[predicate](https://en.cppreference.com/w/cpp/named_req/Predicate). A predicate
is just an expression which returns a boolean. The purpose of said predicate in
this case is to prevent [spurious
wakeups](https://en.wikipedia.org/wiki/Spurious_wakeup), which is a common
phenomenon with condition variables. Spurious wakeups occur when a waiting
thread is woken up without the required condition actually being met. In this
case that would mean the `push_wait_for` thread has been woken up without
`notify_one` having been called in any `pop` functions. Double checking the
actual state of the buffer is thus needed to prevent spurious wakeups from
causing bugs.

Lastly, the actual operation of `wait_for` is a bit counterintuitive. If the
function requires a locked mutex, how is it possible for another thread to be
woken up? It might seem this would cause a deadlock since no other thread can
use the mutex while the thread is waiting. Fortunately, this does not cause a
deadlock in practice. The `wait_for` function, after receiving a locked mutex,
unlocks the mutex while waiting. Upon waking up, it automatically re-locks the
mutex without any additional action from the programmer.

Now, onto `wait_for`'s return value. If the function succeeds before the timeout
has expired, it returns true. In this case, that would mean an empty space
became available in the buffer before the timeout expired. The element can thus
be pushed in to the buffer (lines 12-13). If instead the timeout expires before
an empty space is available, the buffer acknowledges this by incrementing
`dropped` and setting the `success` flag to false (lines 17-18). Regardless of
the success of this action, the buffer must now contain at least one element so
the `q_has_element` condition variable can be notified before the function
returns.

## Testing and Usage

Testing is accomplished with [GoogleTest](https://github.com/google/googletest)
and many usage examples of the bounded buffer can be found in the
[tests](https://github.com/jdtaylor7/bounded_buffer/blob/master/test/bounded_buffer_test.cpp) themselves.
Some usage examples are also discussed below.

The tests can be run with [Bazel](https://bazel.build/):

`bazel test --config=linux //test:bounded_buffer`

or with [CMake](https://cmake.org/) via a testing script:

`./test.sh`

Let's look at the first usage case. Here we create a bounded buffer object with
a capacity value and add/remove elements via `try_push` and `try_pop`,
respectively.

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

It's important to remember that the pop operations return `unique_ptrs` which
must be moved into a result pointer. Of course one should test whether that
result pointer is null before dereferencing it.

Next is a more complex example which demonstrates the bounded buffer being used
in a producer/consumer application.

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

Values are transferred from a large source vector to a large destination vector
using a much smaller bounded buffer. The consumer thread is delayed by one
second, while both threads wait for five seconds before giving up and failing.
Even if a failure does occur on a thread (due to a timeout), that thread will
keep running. This test should pass on most systems, but the timeout parameters
and buffer size can be tweaked if necessary.

If you don't recognize the syntax used in the thread constructors above, I would
highly recommend looking into
[lambdas](https://en.cppreference.com/w/cpp/language/lambda) as they are quite
convenient in these and other scenarios. Here they allow us to define a function
for a thread without writing a separate, named function.

## Conclusion

And that's it! While the data structures in the C++ standard library are not
inherently thread-safe, it is quite possible to wrap them in a thread-safe
interface and customize them for your specific needs. C++11 has provided many of
the mechanisms to do so among library headers such as `<thread>`, `<mutex>`,
`<condition_variable>`, etc. Even more functionality is added to the language
with every release.

The code is located [here](https://github.com/jdtaylor7/bounded_buffer).
Suggestions and pull requests are welcome.

## References
* [RAII - cppreference](https://en.cppreference.com/w/cpp/language/raii)
* [Predicates - cppreference](https://en.cppreference.com/w/cpp/named_req/Predicate)
* [Lambdas - cppreference](https://en.cppreference.com/w/cpp/language/lambda)
* [Spurious wakeups - Wikpedia](https://en.wikipedia.org/wiki/Spurious_wakeup)
