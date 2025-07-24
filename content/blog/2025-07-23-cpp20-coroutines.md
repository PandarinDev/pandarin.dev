---
title: Yet Another C++20 Coroutine Tutorial
description: A basic guide to how C++20 coroutines work under the hood.
date: 2025-07-23
tags: c++
draft: true
---
# Preamble
C++20 adds a bunch of cool new features to an already complicated language full of footguns: Concepts, modules, [spaceships](https://devblogs.microsoft.com/cppblog/simplify-your-code-with-rocket-science-c20s-spaceship-operator/) and even coroutines.
While most of the new features are relatively straightforward, coroutines are not.
This is further proven by the myriad of existing coroutine tutorials and guides already present on the internet. Everyone and their dog seemingly wants to write an article on coroutines, so I figured why not pile on?!

To be honest I found most of the existing documentation and guides unnecessarily complicated and confusing (often even misleading or straight up wrong), so I figured I'll try to explain things my way, and hopefully make things easier to grasp for people.

This is going to be a long article and I firmly believe this is a topic that cannot be taught in short form, so grab a snack and drink and read on. I promise I'll try to make this as short and painless as possible, but this is complicated topic, so it will require some patience and head scratching from you as well. Do not feel bad if something is not immediately obvious to you, it took me multiple attempts to mostly understand the things I write about here.

# Theory
In order to clear up confusion and start from a common ground I want to start with some theory. I know you are just about to scroll past this to see some code, but don't. If you do, you will be banging your head on your desk in confusion, I will point at you specifically and say the ugly words "I told you so" with a smug smile.

## What are coroutines?
Coroutines often confuse developers as a concept - most devs have heard about them, but have only foggy, vague definitions for them. Let me clear that up in one sentence:

**A coroutine is a function that can be suspended and resumed at will.** That's it.

It is essentially a control flow mechanism, where you can say "let's give back control to the caller of this function, and if it wants to, it can later resume this function exactly in the state that it was left in".

One thing that often confuses developers is "suspension" and "resuming", somehow thinking that either of those have anything to do with parallel programming and threads. **They do not.** When you suspend a coroutine control is handed back to the caller and when you resume a coroutine you do not spawn a new thread or anything like that - it is just like any other function call, except you will resume that function in the same state it was supended in (from the same line, with it's state variables being the same).

This already should make experienced engineers scratch their heads - how can you resume a function from the state that it was left in without threads and context switching? Well, the answer is that you store the state of the function not on the stack as you would normally do, but on a separate frame that is heap allocated. That frame has to be large enough to store state variables of the function as well as an "instruction pointer" that tells the coroutine mechanism where to continue the function from.

## But I thought coroutines are a way to make parallel programs more efficient?!

Yes and no. The reason why many developers are confused by this is because coroutines are often used in conjunction with threads to make parallelism more efficient through a concept called ["cooperative multitasking"](https://en.wikipedia.org/wiki/Cooperative_multitasking). Coroutines are a key part of writing efficient parallel programs, and coroutines and thread(pool)s go together like peanut butter and jelly. But just like peanut butter is not jelly, coroutines are not threads.

Without going into too much detail before you understand the full theory, when you are waiting for something (e.g. reading a chunk of bytes from a socket) you can tell the kernel to begin the reading operation, then suspend the coroutine, and as such, return the control back to the caller - which is often a thread in a thread pool when working on parallel programs.

This way the thread in the pool is not blocked while we are reading the bytes, it can pick up a new task and start executing it. When the kernel finished reading that chunk of bytes we told it to read, it will flag the data available, for which we'll have a way of detection in place and mark the coroutine ready to run again (but not resume it immediately). This way when a thread in the threadpool is looking for work again it will see that the task is ready to be resumed and will call `.resume()` on the coroutine handle to resume it exactly from the same state.

The brilliance of this - and the entire point of coroutines - is that not only you get to utilize your threads more efficiently, but also this way you get to avoid chaining together async operations by passing down lambdas everywhere as callbacks (aka ["callback hell"](https://medium.com/@raihan_tazdid/callback-hell-in-javascript-all-you-need-to-know-296f7f5d3c1)). You just write your function as you would normally do, the "coroutine scheduler" will take care of the rest.

It is also super important to mention at this point that in C++20 coroutines are just a language feature, there is no library support, there is no scheduler built-in. If that's what you are after, take a look at something like [cppcoro](https://github.com/lewissbaker/cppcoro) once you read this article and understood coroutines.

## How C++ coroutines work
Let me start off by explaining how the C++20 coroutine mechanism works. You are not supposed to understand this fully yet and I do encourage you to scroll back to this section occasionally and check your understanding against this chapter.

In order to define a coroutine you need a function - it only makes sense, after all coroutines are just suspendable functions. In other languages you often need to prefix these functions with keywords such as `async`, but not in C++, we do not have any keywords in the function signature for coroutines.

What makes a coroutine a coroutine in C++ however, is that it has at least 1 of 3 keywords in its function body: `co_await`/`co_yield`/`co_return`. It needs at least one of those, otherwise the function will not be considered a coroutine by the compiler.

Of course with any good function, you'll also need to define a return type (functions returning void cannot be coroutines, you'll see why). The C++ compiler will put restrictions on what return type you can use for coroutines: The return type is required to have a member called `promise_type`. That is the only restriction the return type needs to fulfill. However, the `promise_type` in turn is required to have much more logic:
- It must have a `get_return_object` function that returns the return object of the coroutine
- It must have a `initial_suspend` function that returns an awaitable
- It must have a `final_suspend` function that returns an awaitable and is non-throwing (`noexcept`)
- It must have either a `return_void` or `return_value` function that unintuitively returns nothing
- It must have an `unhandled_exception` function

The last piece of this puzzle (for now) is the `std::coroutine_handle<promise_type>` that is a handle to the coroutine itself. This coroutine handle object is the one that has the heap allocated frame inside for the function. You can use the handle to `.resume()` the coroutine or `.destroy()` it to deallocate the coroutine frame (if the coroutine is suspended, otherwise UB). The handle is a non-owning object, meaning that just because a handle pointing to the frame is destructed the frame will not be destructed - for that you need to let the coroutine run to completion or explicitly call `.destroy()` on a handle.

When you call a coroutine the compiler expands that function call and spices in some magic. That magic is:
- it allocates the coroutine frame on the heap
- it instantiates the promise type (which it figures out from the return type's `promise_type` member)
- it instantiates the return object using `promise.get_return_object()`
- it checks whether the coroutine needs to be immediately suspended by calling `promise.initial_suspend()`
- if an exception happens in the function body `promise.unhandled_exception()` is called
- once the execution is complete either by reaching the end of the function or a `co_return` the mechanism checks if the coroutine needs to be suspended one final time by calling `promise.final_suspend()`
- after the `final_suspend()` is resumed the coroutine frame is automatically destroyed

Let us see an example.

# The simplest possible coroutine in C++20
```cpp
struct example_return_type {
    struct promise_type {
        auto get_return_object() { return example_return_type{}; }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}
    };
};

example_return_type my_first_coroutine() {
    co_return;
}

int main() {
    my_first_coroutine();
    return 0;
}
```
[[Try it out on Godbolt]](https://godbolt.org/z/rY66GhcT5)

Rolls off the tongue, doesn't it? Let us take in order what this does.
- we have `example_return_type` for the return type
- inside we define `promise_type` that does the following:
    - defines `get_return_object()` that returns an instance of `example_return_type`
    - specifies `std::suspend_never` for `initial_suspend()`, meaning that we do not want to immediately suspend the function, keep executing it please
    - specifies `std::suspend_never` for `final_suspend()`, meaning we do not want to suspend the coroutine after it finished executing the function
    - defines `return_void()` that gets called when we call `co_return;` in the function (or when execution would naturally fall off at the end of the function)
        - you can think of it as a listener, in case you want to set something in the promise when a coroutine returns
    - defines `unhandled_exception()` as an empty function, hoping nothing bad would happen (of course don't do this in production)
- finally, we define the coroutine itself in the form of `my_first_coroutine`
    - the function actually does nothing, but we need to specify `co_return` in it for the compiler to recognize as a coroutine
    - `co_await` or `co_yield` would also work, but they do not make sense in the context of this example

# Making use of the promise
Now that we have seen the simplest example let us see one where the promise object is actually used. Note that in the previous example even if we could access the promise object somehow in `main()` it would already be deallocated, and we'd have a "use-after-free" bug on our hands. That is because the `final_suspend()` is `std::suspend_never` for the coroutine, meaning that after the function body finished execution the coroutine is **not** suspended and thus the frame is automatically freed up at the end of the coroutine.

If we want to use the promise we can do so when the coroutine is in a suspended state, e.g. when control is handed back to us during a `co_await`/`co_yield` or even after the coroutine function body finished executing if we suspend the coroutine before the frame is deallocated. We can do that in `final_suspend()` by e.g. returning `std::suspend_always`. **But this also means that we become responsible for telling the coroutine mechanism when to clean up the coroutine frame.**

To do that we need to store the `std::coroutine_handle<promise_type>` in our return object and explicitly call `.destroy()` on it in the destructor, thus avoiding leaking the frame. It is also worth mentioning again that calling `.destroy()` on a non-suspended coroutine is undefined behavior, so be careful.

Let's see the necessary changes to our code:
```cpp
struct example_return_type {
    struct promise_type {

        int x = 42;

        auto get_return_object() {
            return example_return_type{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }
        /* ... */
        std::suspend_always final_suspend() noexcept { return {}; }
        /* ... */
    };

    std::coroutine_handle<promise_type> handle;

    ~example_return_type() {
        if (handle) {
            handle.destroy();
        }
    }
}

/* ... */

int main() {
    auto return_obj = my_first_coroutine();
    // Now this is safe because the coroutine is suspended in final_suspend() and the lifetime
    // of the coroutine frame is explicitly tied to the lifetime of the return object (see ~example_return_type).
    std::cout << return_obj.handle.promise().x << std::endl;
    return 0;
}
```
[[Try it out on Godbolt]](https://godbolt.org/z/KcnjjTrTz)

The changes are the following:
- we return `std::suspend_always` in `final_suspend()` to suspend the coroutine before it gets deallocated
- we now store a `std::coroutine_handle<promise_type>` field called `handle` in our return object
- in order to create the return object we need to pass in this handle that we create from the promise
- we can access the promise object through the handle using `handle.promise()`
- the coroutine is getting explicitly `.destroy()`ed in the destructor of `example_return_type`

You also might want to delete the copy constructor for `example_return_type` and only allow it to be moved, otherwise if the return object is copied you might end up `.destroy()`ing the handle multiple times which leads to all sorts of headaches and UB - but I didn't want to complicate the example any further.

# What is an awaiter?
Now that we've seen some coroutines let us also see an example for a coroutine that actually suspends during execution.
In order to demonstrate `co_await` we need to understand "awaitables" and how `co_await` works. When we call `co_await` we always need supply an awaiter as an argument and that awaiter will dictate whether or not we are actually going to suspend execution. This means that just because you see a `co_await` it doesn't necessarily mean that execution will be suspended - it is a **potential** suspension point.

Previously we have used `std::suspend_always` and `std::suspend_never` from the standard library for awaitables, but now we are going define our own.

An awaitable needs to define 3 things:
- an `await_ready()` function that returns a boolean
    - if `true` there is no need to suspend, whatever we are waiting for is already ready, resume the coroutine
    - if `false` suspend the coroutine
- an `await_suspend()` function
    - this function is called when the coroutine is suspended, think of it as a "suspension listener"
    - this function receives the `std::coroutine_handle<promise_type>` as an argument
        - this is useful in case you want to manipulate the promise object or if you want to forward the handle somewhere to `.resume()` it from there later
    - for now let's just say the return type is `void`, but it can get more complicated in advanced use cases
- an `await_resume()` function
    - this function is called when a coroutine is resumed, think of it as a "resume listener"
    - the return value of this function will be the return value of the `co_await`

Armed with this knowledge we could write `std::suspend_always` ourselves trivially. Let's do that as a practice example.
```cpp
struct suspend_always {
    bool await_ready() { return false; }
    void await_suspend(std::coroutine_handle<> handle) {}
    void await_resume() {}
};
```
We simply return `false` when asked if the thing we are waiting for is ready, thus triggering the suspension of the coroutine. Also we don't do anything when the coroutine gets suspended or resumed.

Of course this is a boring (but useful) awaiter, but we can use it to demonstrate something more novel.

```cpp
example_return_type my_second_coroutine() {
    std::cout << "Hello ";
    co_await std::suspend_always{};
    std::cout << "world" << std::endl;
}

int main() {
    auto return_obj = my_second_coroutine();
    std::cout << "beautiful ";
    return_obj.handle.resume();
    return 0;
}
```
[[Try it out on Godbolt]](https://godbolt.org/z/c4zeG56EM)

This is the point in the article where you first truly get to test your understanding of the things you read so far. What will this program print to standard out?

Of course the answer is somewhat obviously "Hello beautiful world", but let's break it down.
- We first call `my_second_coroutine()`
- `initial_suspend()` returns `std::suspend_never`, execution begins
- We print "Hello "
- We `co_await std::suspend_always{}` which returns `false` for `await_ready()`, suspending the coroutine
- Control gets transferred back to `main`, we print "beautiful "
- We resume the coroutine using `return_obj.handle.resume()` which first calls `await_resume()` on the awaited `std::suspend_always` instance
- Execution of the coroutine continues, printing "world"
- `return_void()` is called on the promise object
- `final_suspend()` returns `std::suspend_always`, suspending the coroutine
- When the return object goes out-of-scope we `.destroy()` the coroutine frame

At this point you should understand `co_await`, how suspension and resuming transfers the control flow back and forth between the caller and the coroutine, and the role that awaiters play in that.

# Let us write a generator
Generators are a thing that can be very well modelled using a coroutine and are a great candidate for demonstrating `co_yield`.

We will write the simplest possible generator that will return an integer value that gets incremented with every call.

```cpp
struct generator_t {
    struct promise_type {

        int current_value;

        auto get_return_object() {
            return generator_t{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}
        std::suspend_always yield_value(int value) {
            current_value = value;
            return {};
        }
    };

    std::coroutine_handle<promise_type> handle;

    ~generator_t() {
        if (handle) {
            handle.destroy();
        }
    }

    int operator()() {
        handle.resume();
        return handle.promise().current_value;
    }

};

generator_t generator() {
    int x = 0;
    while (true) co_yield x++;
}

int main() {
    auto gen_obj = generator();
    std::cout << gen_obj() << std::endl; // 0
    std::cout << gen_obj() << std::endl; // 1
    std::cout << gen_obj() << std::endl; // 2
    return 0;
}
```
[[Try it out on Godbolt]](https://godbolt.org/z/qcbrba8KE)

As you can see when we are using `co_yield` in the generator function, it will call a function that is newly defined in our `promise_type`: `yield_value()`.
Now the interesting thing about `yield_value` is that it does not actually yield anything, it is just a listener that gets called whenever the function yields something. This is because `co_yield <expression>;` is just syntactic sugar for `co_await yield_value(expression);`.

The approach that I chose here is that during the yield I store the value in the promise, then define an `operator()` on the return object which will simply resume the coroutine until the next suspension point, then return the value stored in the promise object.

Breaking this down we get:
- we call `generator()` which gets immediately suspended because `initial_suspend()` is `std::suspend_always`
- control immediately returns to `main`
- we call the `operator()` of the return object (repeatedly), each time resuming the coroutine which will execute until the next `co_yield` statement
- at that point `yield_value()` stores the yielded value in the promise
- because the returned awaitable of `yield_value()` is `std::suspend_always` the coroutine is suspended and control returns to `operator()` of the return object
- the `operator()` will return the stored value from the promise which we print out in `main`
- once the return object goes out of scope the coroutine is `.destroy()`ed

Perhaps it is also worth mentioning that in this case it does not really matter what we choose for `final_suspend()`, the end of the `generator()` function body will never be reached, we will always "exit" the coroutine by destroying it through its handle.

# My thoughts on coroutines in C++20
And with that it's a wrap. We have talked about coroutines, handles, promises, awaitables, `co_await`/`co_yield`/`co_return` and more. At this point the only thing left to do is editorializing this article a bit by talking about my feelings with regards to coroutines as they are in C++20.

While I'm glad the feature is there, and no doubt it will be useful for highly parallel programs I can't help but feel that in typical C++ fashion the feature is overly complicated and verbose.

I get that library support is yet to come, and as such this is a barebones way of interacting with coroutines, but do you know how the same generator example looks like in Python? I will show you now out of spite.
```python
def generator():
    x = 0
    while True:
        yield x
        x += 1

gen_obj = generator()
print(next(gen_obj)) # 0
print(next(gen_obj)) # 1
print(next(gen_obj)) # 2
```

This is it. This is how simple I expected C++ coroutines to be when they were announced. Of course Python being an interpreted language this is a bit of an unfair comparison, but you get the point.

Anyways, I'll let everyone make up their own mind about how they feel about the implementation of the feature, I only wanted to clarify the concept through this article. In case any of my examples make no sense to you, if you have any questions or you spotted a mistake feel free to reach out.

Have a great day, and thanks for taking the time to read through all this, hope you learned something.
