---
title: Writing C++20 coroutine scheduler
description: Details on how to write a coroutine scheduler
date: 2025-07-31
tags: c++
draft: true
permalink: cpp20-coroutine-scheduler/
---
# Preamble
In my [previous article](/cpp20-coroutine-basics/) I discussed the fundamentals of coroutines as they are in C++20. One of the crucial things in order to make coroutines truly useful that is missing from the standard library is a coroutine scheduler, so we'll write one today. Of course space in a blog article is always limited, so I'll only discuss the core concepts and will link the full implementation in a GitHub repo that you can check out and play with. It is probably also worth mentioning that despite the ideas here being solid this scheduler is not battle-tested and you probably should not be using it in production.

# Await transform
In order to start writing our scheduler we'll need some new tools compared to what was explained in my coroutine basics article. On this journey `await_transform()` will be our first new friend. Previously we have talked about `initial_suspend()` and `final_suspend()` with regards to the promise type, and as their name implies these functions are responsible for defining the behavior of our coroutine at its initial and final potential suspension points. So a followup question naturally arises - what if we want to have a function that gets called every time the coroutine function body is potentially suspended? Something like an await listener between `initial_suspend()` and `final_suspend()`? Enter `await_transform()`.

At first the name of `await_transform()` might be confusing, but that is because it is more powerful than just a simple await listener. It has exactly one parameter which is the awaiter that was `co_await`ed, and it is also expected to return an awaiter. `await_transform()` will be called for every `co_await` in the coroutine body and its returned awaiter will replace the original awaiter of the `co_await` call. So as you might imagine this can be useful to intercept await operations, but with a little thinking we can use this for all sorts of magic.

Before we get too deep into shenanigans that use `await_transform()`, let us see a basic example.

```cpp
struct my_return_type {
    struct promise_type {
        my_return_type get_return_object() { return {}; }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}

        template<typename T>
        std::suspend_always await_transform(T awaiter) {
            return {};
        }
    };
};

my_return_type example_coroutine() {
    std::cout << "Foo";
    co_await std::suspend_never{};
    std::cout << "bar" << std::endl;
}

int main() {
    example_coroutine();
    return 0;
}
```
[[Try it out on Godbolt]](https://godbolt.org/z/foz84694E)

Now if you were to just read the body of `example_coroutine()` you'd think that this program will print "Foobar" to standard out. The twist is that the `await_tranform()` will catch the await operation and transform our awaiter from a `std::suspend_never` to a `std::suspend_always`, suspending the coroutine before printing "bar" and letting main exit, ultimately only printing "Foo".

There are two important things to notice here:
- `await_transform()` actually receives the original awaiter
- because of this, you need to be able to specify the type of the awaiter you want to transform in your function signature. If you want to transform all awaiters you can of course just template `await_transform()` like in the example.

# Starting work on the scheduler
The previous chapter has further implications than what might be immediately obvious. It means that we can modify the behavior of our coroutines through the return type. If you take the body of a coroutine function and specify two different return types for it, the behavior of the coroutine suddenly might be entirely different. This might seem obscure at first, but it is the sort of mechanism that we can use to write code that is heralded as dark magic. The good kind.

Before we proceed let me explain what is a coroutine scheduler. A coroutine scheduler is nothing more than a good old thread pool, except instead of submitting invocables you submit coroutines. When a thread in the pool is looking for work it simply resumes a coroutine handle which will run until the next suspension point - in contrast to a traditional thread pool where the submitted task needs to be executed in one go, potentially blocking a thread in the pool. 

As I mentioned in my previous article, coroutine support in the language is barebones since there is no standard library support, which in turn means no built-in goodies such as coroutine schedulers. Despite many coroutine libraries already existing for C++ (such as [concurrencpp](https://github.com/David-Haim/concurrencpp), [cppcoro](https://github.com/lewissbaker/cppcoro), [libcoro](https://github.com/jbaldwin/libcoro) and more) as far as I'm aware none of them have quite what I want: A coroutine scheduler that can schedule coroutines without those coroutines being aware of the scheduler itself.

You see, most of these coroutine libaries expect you to write code such as this:
```cpp
task my_coroutine(std::shared_ptr<scheduler> scheduler) {
    co_await scheduler->schedule_me();
    // ...
    auto data = co_await fetch_me_some_data();
    co_await scheduler->schedule_me();
    // ...
}

int main() {
    auto scheduler = create_scheduler();
    my_coroutine(scheduler);
    return 0;
}
```
It is immediately obvious `my_coroutine()` is directly tied to the scheduler. Not only it needs to start off by scheduling itself on the scheduler (which is implemented through `await_suspend()` saving the coroutine handle to the work queue of the scheduler, which later resumes it on one of it's threads), but after every single await operation you are expected to reschedule the coroutine. This is because awaiters that do async operations often resume on their own threads which is unwanted (e.g. you don't want to continue running your coroutine on an I/O completion handler thread).

To illustrate what I mean let us imagine `fetch_me_some_data()` is implemented like this:
```cpp
auto fetch_me_some_data() {
    struct data_fetch_awaiter {

        DataType data;

        bool await_ready() { return false; }
        void await_suspend(std::coroutine_handle<promise_type> handle) {
            std::thread([this, handle]() {
                this->data = some_http_call();
                handle.resume();
            }).detach();
        }
        DataType await_resume() { return data; }
    };
    return data_fetch_awaiter{};
}
```
As you can see the data fetching is asynchronous and happens on a detached thread. Once we have the data the awaiter has no option but to resume handle and this is where our problem lies. You see we want to continue running our coroutine on our scheduler, not on this detached thread. So we need to reschedule ourselves to the scheduler. But how do we do this without either the coroutine itself or the awaiter being aware of the scheduler?

In an ideal world my original example woud look something like this:
```cpp
task my_coroutine() {
    // ...
    auto data = co_await fetch_me_some_data();
    // ...
}

int main() {
    auto scheduler = create_scheduler();
    scheduler->schedule(my_coroutine());
    return 0;
}
```
Now `my_coroutine()` not only has no dependence on the scheduler, but I would also expect it to get resumed on the scheduler after `fetch_me_some_data()` regardless of where the awaiter inside has resumed the handle. But how can we achieve this?

<center><img src="./pooh-think.gif" alt="Winnie the Pooh, thinking really hard"></center>

# Where the magic happens
Our solution will lie in `await_transform()` as you might have guessed, but it is not exactly what I'd call trivial. We cannot let the awaiters inside the coroutine actually resume the coroutine - but since they get a coroutine handle in `await_suspend()`, how do we prevent them from calling `.resume()` on it?

Well, we don't. Instead, we'll wrap the awaiters in one of our own and swap the handle that we pass to the original awaiter in `await_suspend()` to a different "dummy" coroutine's handle. Once they resume that dummy coroutine that is when we need to reschedule the original coroutine for execution.

We can do all of this using `await_transform()` and by writing a reschedule coroutine, but there is one more question - how do we pass the scheduler to the reschedule coroutine? Well in order to do that we need to store a pointer to the scheduler in the promise at the time of scheduling the coroutine in the beginning. Let's see:
```cpp
struct reschedule_task {
    struct promise_type {
        reschedule_task 
    };
};
```