---
title: Dependency Injection Using Reflection in C++26
description: Using C++26 reflections to implement dependency injection.
date: 2025-07-24
tags: c++
draft: true
---
# Foreword
If you asked me just a few years ago what will happen to C++ in the future, I would have given you a doomer answer. I honestly thought for a long time (between C++11 and C++20) that the resistance to adopting new features to the language is too strong both in the committee and the C++ community, which will ultimately end up in C++ becoming obselete.

However, if you were to ask me the same question today, my answer would be completely different. Ever since it was announced that [compile-time reflection will be available in C++26](https://herbsutter.com/2025/06/21/trip-report-june-2025-iso-c-standards-meeting-sofia-bulgaria/), I have a big fat smile on my face every time I think about the future of C++.

I honestly think C++26 will be revered the same way C++11 was in terms of fundamentally changing the language for the better. Or sometimes for worse. This one new language feature adds so many possibilities, it's hard to foresee all of the consequences. I'm already imagining [aspect-oriented programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming) frameworks coming to the language, something like [Spring](https://spring.io/projects/spring-framework) for C++. I'm simultaneously giggling in horror and feeling inspired.

In order to celebrate this great news, I started thinking what would be a good way of demonstrating the uses of reflection. Now of course when talking about reflection everybodys' first thought is always serialization. Booooring. Don't get me wrong, serialization is very useful, but I wanted to put a more unique spin on the topic.

So let's talk about [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) and [inversion of control](https://en.wikipedia.org/wiki/Inversion_of_control)! But only after a quick word on reflection.

# What is Reflection
Reflection is the ability to query the structure of the code. It can be further categorized into compile-time reflection and runtime reflection, depending on whether you are able to query this data at runtime as well. As you can imagine runtime reflection is more powerful out of the two but it also incures a runtime overhead in that you have to store the reflection data for each of your types in the compiled binary. That in turn not only increases the size of the compiled binary, but also potentially increases memory usage, can leak data about symbols,

# Dependency Injection
Dependency injection is a form of inversion of control (IoC from now on), in that instead of hardcoding your dependencies you get them from the IoC container. You can imagine the IoC container as a singleton where all of your components ("beans" when using Java terminology) are registered, and when you want to instantiate an object every dependency it has is fulfilled from those registered components.

This promotes good programming habits in that you'll have all of your dependencies typically listed as constructor parameters which makes your classes easier to test. All you have to do in your test code is to change the registered components in the IoC container to mock objects and boom, you've got yourself a sweet unit-test. Furthermore, this reduces verbosity because you will rarely be directly instantiating your objects, meaning that you don't have manually pass all the constructor parameters.

But enough talk, let us see some code.