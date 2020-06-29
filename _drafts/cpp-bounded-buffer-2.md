---
layout: post
title: "C++ Bounded Buffer Part 2"
description: "Implementing a Thread-Safe Bounded Buffer in C++"
tags: [cpp, concurrency]
---

Introduction

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
