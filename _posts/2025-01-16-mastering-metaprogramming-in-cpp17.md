---
layout: post
title: Metaprogramming in C++17 Dive into Compile Time Introspection and SFINAE
subtitle:  Brace Yourself for Error Messages You Didn't Know Existed.
tags: tags: [C++, Metaprogramming, C++17, Templates, Type Traits, SFINAE, Compile-time, decltype, std::declval, static_assert, C++ Programming]
comments: false
mathjax: true
author: Rebraws
---



#### Metaprogramming in C++17: Dive into Compile-Time Introspection and SFINAE

C++17 wields a powerful arsenal of compile-time tools, most notably through type traits, type traits allow us to investigate the properties of a type in C++ like: Is it a class? Is it integral? Does it have a specific member function?

Some of these inquires are more straightforward than others. For instance, determining if a type `T` is an non-union class is as simple as using the `std::is_class<T>` type trait. This will evaluate to `true` if the condition is met and `false` otherwise.

But what if we need to dive deeper---say, to find out whether a class posseses certain member function? Unfortunately, the C++17 standard library doesn't provide a `std::has_member_function` type trait out of the box. However, we can leverage the existing compile-time mechanisms to build our own solution, effectively creating  our own `has_member_function` functionality.

#### Beyond the constructor: Entering the unevaluated zone.

Before we can effectively craft our own `has_member_function` type trait there are two fundamental concepts we need to grasp: `decltype` and `std::declval`. These are essential tools that will allow us to work with types and expressions in the way necessary for compile-time introspection.
#####  Unveiling Expressions Types and Unevaluated Objects at Compile Time 
First on our list is `decltype`. In essence, `decltype` provides us with a mechanism to ask the compiler, "What is the type of this expression?". The answer it provides is the exact type, preserving crucial information like `const` and reference qualifiers.  

An essential aspect of `decltype` is that no temporary object is created when determining the type of an expression. Moreover, the type in question need not be complete, require a constructor or destructor, it can even be an abstract class. 

Let's take a look at a very simple example:

```
#include <type_traits>

class Foo;

auto f() -> void {
	Foo* foo;
	static_assert(
		std::is_same_v<std::remove_poiner_t<decltype(foo)>, Foo>);
}

```

We start by forward-declaring `class Foo`, and then we declare a pointer `Foo* foo` inside a function. Now the interesting part, `decltype(foo)` will deduce the type of the expression `foo`, which is `Foo*`, we then use `std::remove_pointer_t`, to strip away the pointer, leaving us the underlying type `Foo`. Finally the static assertion with `std::is_same_v<..., Foo>` confirms at compile time that the type we got after removing the pointer is indeed `Foo`. This demonstrates that `decltype` can successfully determine the type of `foo` even though is only forward-declared and thus an incomplete type.

Now that we have an understanding of `decltype`, let's turn our attention to `std::declval`, this helper template converts any type to an expression of that type. More precisely, it generates an rvalue reference even when we can't (or don't want to) construct those objects, this makes possible to use member functions of the underlying type without the need to go through constructors.

Let's consider the following illustrative example.

```
#include <type_traits>

struct Foo {
	virtual auto bar() -> int = 0;
};

static_assert(std::is_same_v<decltype(std::declval<Foo>().bar()), int>);

```

In this case, `Foo` is an abstract class due to the pure virtual function `bar()`, meaning we cannot create instances of `Foo` directly (e.g. `Foo foo` would cause a compile error). Here, `std::declval<Foo>()` comes into play by providing a way to refer to a hypothetical, unevaluated object of type `Foo`. This allows us to form the expression `std::declval<Foo>().bar()` that appears to call the member function `bar()`, though no actual call occurs at runtime, `decltype` then analyzes the return type of this expression, correctly deducing it as an `int`. 

All of this works flawlessly so far, but there's a few caveats , like what happens if the expression we provide to `decltype` is ill-formed? 

Let's take a look at a very similar example where this happens.

```
#include <type_traits>

struct Foo {
	Foo() = delete;
	auto foobar() -> void;
};

static_assert(std::is_same_v<decltype(std::declval<Foo>().bar()), int>);
```

Here, we have a `Foo` class with a deleted default constructor and without a `bar()` member function,  therefore the expression `std::declval<Foo>().bar()` is invalid and attempting to compile this code as it is will indeed result in a compilation error , likely something along the lines of:
{:. box-error}
**error:** no member named 'bar' in 'Foo'

So as we can see, `decltype` isn't magic. It relies on the compiler's ability to parse and understand the expression. If the expression we give it is nonsensical from the compiler's perspective, `decltype` cannot determine its type, leading to a compilation failure.


Now, you're probably thinking, and quite rightly so: if the compiler throws an error when an expression is il-formed, how are we going to detect whether a class has a member function - something that might not even exists? It seems counterintuitive, doesn't it?

This is where the magic of C++ templates comes into play. Our initial `static_assert` example illustrated a direct compilation error. However, templates provide a more flexible environment where the compiler can explore different possibilities before committing to a specific piece of code.


#### Embracing the Dark Side: Venturing into the SFINAE Wilderness

The flexibility afforded by C++ templates relies on a key mechanism during function template resolution: ***Substitution Failure Is Not An Error (SFINAE)***. Imagine the compiler trying to instantiate a template. If, during this process, it encounters an error in the immediate context of substitution, SFINAE dictates that this isn't necessarily a fatal error. Instead, the compiler discards that particular instantiation and continues searching for other viable templates candidates.

Let's take a look at a typical example

```
#include <type_traits>
#include <iostream>

struct Foo {
	template <class Integer, std::enable_if_t<std::is_integral_v<Integer>, void>* = nullptr>
	Foo(Integer) {
		std::cout << "We've got an integer\n";
	}

	template <class Float, std::enable_if_t<std::is_floating_point_v<Float>, void>* = nullptr>
	Foo(Float) {
		std::cout << "We've got a floating point\n";
	}
};

Foo f(2.4f);

```

In this example, the class `Foo` provides too templated constructors. Each constructor uses `std::enable_if_t` to conditionally enable itself based on the type of its argument.

{: .box-note}
**Note:** The way `std::enable_if` works is quite simple, when the condition that it evaluates is `true` it defines a member type alias, this type alias can be accessed either via `std::enable_if<>::type` or via `std::enable_if_t<>`. When the condition it evaluates is `false` there's no member type alias defined.

In our example, when the compiler processes the statement: `Foo f(2.4f)`, it evaluates both constructors.

For the first constructor, `std::is_integral_v<Integer>` evaluates to `false` because `2.4f` is a floating point value. As a result, `std::enable_if` doesn't have a member type alias, and we end with an invalid expression, but, thanks to SFINAE, the compiler silently discards this constructor from the overload set instead of raising a hard compilation error.

For the second constructor, `std::is_floating_point_v<Float>` evaluates to `true`. Therefore `std::enable_if` has a member type alias defined and this constructor remains in the overload set and is ultimately selected for constructing the object.

#### Putting It All Together: Crafting Our `has_member_function` Type Trait

Now that we've explored the power of `decltype`, `std::declval`, and the crucial role of *SFINAE*, we have all the pieces necessary to create our type trait. 

As we've seen, the core idea is to use a template that attempts to form an expression involving the member function we are interested in (.i.e `std::declval<T>().bar()`).  If the member function exists, the template substitution will succeed. If not, SFINAE will kick in, allowing us to detect the absence of the member function at compile time.

To detect this at compile time, we'll use a variable template. Let's begin by defining a basic template that assumes the member function doesn't exists.

```
#include <type_traits>
template <class Type, class = bool>
inline constexpr bool has_member_function = false;

```

Here, we define a `constexpr` boolean variable template. This template variable takes two template parameters. 
	- `class Type`: This is the type we want to inspect.
	- `class = bool` This is a non-type template parameter with a default value of bool, we need this to enable template specialization with SFINAE.

Now, we need to create a specialization of this template that will be selected by the compiler only when the expression `std::declval<Type>().bar()` is valid. 

```
#include <type_traits>

template <class Type, class = bool>
inline constexpr bool has_member_function = false;

template <class Type>
inline constexpr bool has_member_function<Type, decltype(std::declval<Type>().bar(), bool())> = true;

```

In the specialization, if the expression is valid, `decltype(std::declval<Type>().bar(), bool())` evaluates to the type bool.

In essence, when we use `has_member_function<Foo>`, the compiler will consider both the base template and the specialization. If `Foo` has a valid `bar` member function, the `dectype` in the specialization evaluates to bool, making the specialization a valid match. If `Foo` does not have a valid `bar` member function the expression in the specialization is ill-formed and therefore the specialization is discarded due to SFINAE, and the compiler falls back to the base template.

There are many ways to implement the same thing in C++. We could use, classes, functions, variables. W could even use `std::void_t` or `std::enable_if` instead of `decltype(..., bool())`. However, what's important is that all these alternatives rely on the same principle: *SFINAE*. 

Here's an example of the same code but using `std::enable_if`
```
#include <type_traits>
template <class T>
inline constexpr bool helper = true;

template <class Type, class = bool>
inline constexpr bool has_member_function = false;

template <class Type>
inline constexpr bool has_member_function<Type, std::enable_if_t<helper<decltype(std::declval<Type>().bar())>, bool>> = true;

```

#### Wrapping up and a challenge for you!

In this post, we've navigated through several metaprogramming concepts in C++17. We've seen how to craft a simple type trait that detects whether a class contains a member function with a specific name. While our implementation works for the simplest cases, there's plenty of room to explore further. 

Now that you've built a foundation in compile-time introspection and SFINAE, it's time to put your skills to the test. 

- Write a type trait that detects whether a given type `T` has a member function named `foo` that is callable with a given set of argument types. This implementation should be general enough to work with member functions that take any number of arguments (including zero). 
- What are the limitations of using `decltype(&Type::foo, bool())` for detecting member functions?


```
template <class Type, class = bool>
inline constexpr bool has_member_function = false;

template <class Type>
inline constexpr bool has_member_function<Type, decltype(&Type::foo, bool())> = true;

```



