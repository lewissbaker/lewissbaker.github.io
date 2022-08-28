---
title: "C++ Coroutines: Understanding the Compiler Transform"
excerpt:
  In this post I am going to go a bit deeper to show how all the concepts from
  the previous posts come together. I'll show what happens when a coroutine reaches
  a suspend-point by walking through the lowering of a coroutine into equivalent
  non-coroutine, imperative C++ code.
commentIssueId: 6
---

# Introduction

Previous blogs in the series on "Understanding C++ Coroutines" talked about the different
kinds of transforms the compiler performs on a coroutine and its `co_await`, `co_yield`
and `co_return` expressions. These posts described how each expression was lowered by
the compiler to calls to various customisation points/methods on user-defined types.

1. [Coroutine Theory](https://lewissbaker.github.io/2017/09/25/coroutine-theory)
2. [C++ Coroutines: Understanding operator co_await](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await)
3. [C++ Coroutines: Understanding the promise type](https://lewissbaker.github.io/2018/09/05/understanding-the-promise-type)
4. [C++ Coroutines: Understanding Symmetric Transfer](https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer)

However, there was one part of these descriptions that may have left you unsatisfied.
The all hand-waved over the concept of a "suspend-point" and said something vague
like "the coroutine suspends here" and "the coroutine resumes here" but didn't really
go into detail about what that actually means or how it might be implemented by the
compiler.

In this post I am going to go a bit deeper to show how all the concepts from
the previous posts come together. I'll show what happens when a coroutine reaches
a suspend-point by walking through the lowering of a coroutine into equivalent
non-coroutine, imperative C++ code.

Note that I am not going to describe exactly how a particular compiler lowers coroutines
into machine code (compilers have extra tricks up their sleeves here), but rather just one
possible lowering of coroutines into portable C++ code.

Warning: This is going to be a fairly deep dive!

# Setting the Scene

For starters, let's assume we have a basic `task` type that acts as both an awaitable
and a coroutine return-type. For the sake of simplicity, let's assume that this coroutine
type allows producing a result of type `int` asynchronously.

In this post we are going to walk through how to lower the following coroutine function
into C++ code that does not contain any of the coroutine keywords `co_await`, `co_return`
so that we can better understand what this means.

```c++
// Forward declaration of some other function. Its implementation is not relevant.
task f(int x);

// A simple coroutine that we are going to translate to non-C++ code
task g(int x) {
    int fx = co_await f(x);
    co_return fx * fx;
}
```

# Defining the `task` type

To begin, let us first declare the `task` class that we will be working with.

For the purposes of understanding how the coroutine is lowered, we do not need to
know the definitions of the methods for this type. The lowering will just be inserting
calls to them.

The definitions of these methods are not complicated, and I will leave them as an
exercise for the reader as practice for understanding the previous posts.

```c++
class task {
public:
    struct awaiter;

    class promise_type {
    public:
        promise_type() noexcept;
        ~promise_type();

        struct final_awaiter {
            bool await_ready() noexcept;
            std::coroutine_handle<> await_suspend(std::coroutine_handle<promise_type>) noexcept;
            void await_resume() noexcept;
        };

        task get_return_object() noexcept;
        std::suspend_always initial_suspend() noexcept;
        final_awaiter final_suspend() noexcept;
        void unhandled_exception() noexcept;
        void return_value(int result) noexcept;

    private:
        friend task::awaiter;
        std::coroutine_handle<> continuation_;
        std::variant<std::monostate, int, std::exception_ptr> result_;
    };

    task(task&& t) noexcept;
    ~task();
    task& operator=(task&& t) noexcept;

    struct awaiter {
        explicit awaiter(std::coroutine_handle<promise_type> h) noexcept;
        bool await_ready() noexcept;
        std::coroutine_handle<promise_type> await_suspend(std::coroutine_handle<> h) noexcept;
        int await_resume();
    private:
        std::coroutine_handle<promise_type> coro_;
    };

    awaiter operator co_await() && noexcept;

private:
    explicit task(std::coroutine_handle<promise_type> h) noexcept;

    std::coroutine_handle<promise_type> coro_;
};
```

The structure of this task type should be familiar to those that have read the
[C++ Coroutines: Understanding Symmetric Transfer](https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer) post.

# Step 1: Determining the promise type

```c++
task g(int x) {
    int fx = co_await f(x);
    co_return fx * fx;
}
```

When the compiler sees that this function contains one of the three coroutine keywords
(`co_await`, `co_yield` or `co_return`) it starts the coroutine transformation process.

The first step here is determining the `promise_type` to use for this coroutine.

This is determined by substituting the return-type and argument-types of the signature as
template arguments to the `std::coroutine_traits` type.

e.g. For our function, `g`, which has return type `task` and a single argument of type `int`,
the compiler will look this up using `std::coroutine_traits<task, int>::promise_type`.

Let's define an alias so we can refer to this type later:
```c++
using __g_promise_t = std::coroutine_traits<task, int>::promise_type;
```

**Note: I am using leading double-underscore here to indicate symbols internal to the**
**compiler that the compiler generates. Such symbols are reserved by the implementation**
**and should _not_ be used in your own code.**

Now, as we have not specialised `std::coroutine_traits` this will instantiate the primary template
which just defines the nested `promise_type` as an alias of the nested `promise_type` name of the
return-type. i.e. this should resolve to the type `task::promise_type` in our case.

# Step 2: Creating the coroutine state

A coroutine function needs to preserve the state of the coroutine, parameters and local variables
when it suspends so that they remain available when the coroutine is later resumed.

This state, in C++ standardese, is called the _coroutine state_ and is typically heap allocated.

Let's start by defining a struct for the coroutine-state for the coroutine, `g`.

We don't know what the contents of this type are going to be yet, so let's just 
leave it empty for now.

```c++
struct __g_state {
  // to be filled out
};
```

The coroutine state contains a number of different things:

* The promise object
* Copies of any function parameters
* Information about the suspend-point that the coroutine is currently suspended at and how to resume/destroy it
* Storage for any local variables / temporaries whose lifetimes span a suspend-point

Let's start by adding storage for the promise object and parameter copies.

```c++
struct __g_state {
    int x;
    __g_promise_t __promise;

    // to be filled out
};
```

Next we should add a constructor to initialise these data-members.

Recall that the compiler will first attempt to call the promise constructor with lvalue-references to the parameter copies,
if that call is valid, otherwise fall back to calling the default constructor of the promise type.

Let's create a simple helper to assist with this:
```c++
template<typename Promise, typename... Params>
Promise construct_promise([[maybe_unused]] Params&... params) {
    if constexpr (std::constructible_from<Promise, Params&...>) {
        return Promise(params...);
    } else {
        return Promise();
    }
}
```

Thus the coroutine-state constructor might look something like this:

```c++
struct __g_state {
    __g_state(int&& x)
    : x(static_cast<int&&>(x))
    , __promise(construct_promise<__g_promise_t>(x))
    {}

    int x;
    __g_promise_t __promise;
    // to be filled out
};
```

Now that we have the beginnings of a type to represent the coroutine-state, let's also start to
stub out the beginnings of the lowered implementation of `g()` by having it heap-allocate
an instance of the `__g_state` type, passing the function parameters so they can be copied/
moved into the coroutine-state.

Some terminology - I use the term "ramp function" to refer to the part of the coroutine
implementation containing the logic that initialises the coroutine state and gets it ready
to start executing the coroutine.
i.e. it is like an on-ramp for entering execution of the coroutine body.

```c++
task g(int x) {
    auto* state = new __g_state(static_cast<int&&>(x));
    // ... implement rest of the ramp function
}
```

Note that our promise-type does not define its own custom `operator new` overloads,
and so we are just calling global `::operator new` here.

If the promise type _did_ define a custom `operator new` then we'd call that instead of the
global `::operator new`. We would first check whether `operator new` was callable with the
argument list `(size, paramLvalues...)` and if so call it with that argument list. Otherwise,
we'd call it with just the `(size)` argument list. The ability for the `operator new` to get
access to the parameter list of the coroutine function is sometimes called "parameter preview"
and is useful in cases where you want to use an allocator passed as a parameter to allocate
storage for the coroutine-state.

If the compiler found any definition of `__g_promise_t::operator new` then we'd lower to the
following logic instead:
```c++
template<typename Promise, typename... Args>
void* __promise_allocate(std::size_t size, [[maybe_unused]] Args&... args) {
  if constexpr (requires { Promise::operator new(size, args...); }) {
    return Promise::operator new(size, args...);
  } else {
    return Promise::operator new(size);
  }
}

task g(int x) {
    void* state_mem = __promise_allocate<__g_promise_t>(sizeof(__g_state), x);
    __g_state* state;
    try {
        state = ::new (state_mem) __g_state(static_cast<int&&>(x));
    } catch (...) {
        __g_promise_t::operator delete(state_mem);
        throw;
    }
    // ... implement rest of the ramp function
}
```

Also, this promise-type does not define the `get_return_object_on_allocation_failure()` static
member function. If this function is defined on the promise-type then the allocation here would
instead use the `std::nothrow_t` form of `operator new` and upon returning `nullptr` would then
`return __g_promise_t::get_return_object_on_allocation_failure();`.

i.e. it would look something like this instead:
```c++
task g(int x) {
    auto* state = ::new (std::nothrow) __g_state(static_cast<int&&>(x));
    if (state == nullptr) {
        return __g_promise_t::get_return_object_on_allocation_failure();
    }
    // ... implement rest of the ramp function
}
```

For simplicity for the rest of the example, we'll just use the simplest form that calls the
global `::operator new` memory allocation function.

# Step 3: Call `get_return_object()`

The next thing the ramp function does is to call the `get_return_object()` method on the promise
object to obtain the return-value of the ramp function.

The return value is stored as a local variable and is returned at the end of the ramp function
(after the other steps have been completed).

```c++
task g(int x) {
    auto* state = new __g_state(static_cast<int&&>(x));
    decltype(auto) return_value = state->__promise.get_return_object();
    // ... implement rest of ramp function
    return return_value;
}
```

However, now it's possible that the call to `get_return_object()` might throw, and in which
case we want to free the allocated coroutine state. So for good measure, let's give ownership
of the state to a `std::unique_ptr` so that it's freed in case a subsequent operation throws
an exception:

```c++
task g(int x) {
    std::unique_ptr<__g_state> state(new __g_state(static_cast<int&&>(x)));
    decltype(auto) return_value = state->__promise.get_return_object();
    // ... implement rest of ramp function
    return return_value;
}
```

# Step 4: The initial-suspend point

The next thing the ramp function does after calling `get_return_object()` is to start executing
the body of the coroutine, and the first thing to execute in the body of the coroutine is the
initial suspend-point. i.e. we evaluate `co_await promise.initial_suspend()`.

Now, ideally we'd just treat the coroutine as initially suspended and then just implement the
launching of the coroutine as a resumption of the initially suspended coroutine. However, the
specification of the initial-suspend point has a few quirks with regards to how it handles
exceptions and the lifetime of the coroutine state. This was a late tweak to the semantics
of the initial-suspend point just before C++20 was released to fix some perceived issues here.

Within the evaluation of the initial-suspend-point, if an exception is thrown either from:
* the call to `initial_suspend()`,
* the call to `operator co_await()` on the returned awaitable (if one is defined),
* the call to `await_ready()` on the awaiter, or
* the call to `await_suspend()` on the awaiter

Then the exception propagates back to the caller of the ramp function and the coroutine state is
automatically destroyed.

If an exception is thrown either from:
* the call to `await_resume()`,
* the destructor of the object returned from `operator co_await()` (if applicable), or
* the destructor of the object returned from `initial_suspend()`

Then this exception is caught by the coroutine body and `promise.unhandled_exception()` is called.

This means we need to be a bit careful how we handle transforming this part, as some parts will
need to live in the ramp function and other parts in the coroutine body.

Also, since the objects returned from `initial_suspend()` and (optionally) `operator co_await()`
will have lifetimes that span a suspend-point (they are created before the point at which the
coroutine suspends and are destroyed after it resumes) the storage for those objects will need to be
placed in the coroutine state.

In our particular case, the type returned from `initial_suspend()` is `std::suspend_always`, which
happens to be an empty, trivially constructible type. However, logically we still need to store an
instance of this type in the coroutine state, so we'll add storage for it anyway just to show how
this works.

This object will only be constructed at the point that we call `initial_suspend()`, so we need to
add a data-member of a certain type that that allows us to explicitly control its lifetime.

To support this, let's first define a helper class, `manual_lifetime` that is trivally constructible
and trivially destructible but that lets us explicitly construct/destruct the value stored there
when we need to.

```c++
template<typename T>
struct manual_lifetime {
    manual_lifetime() noexcept = default;
    ~manual_lifetime() = default;

    // Not copyable/movable
    manual_lifetime(const manual_lifetime&) = delete;
    manual_lifetime(manual_lifetime&&) = delete;
    manual_lifetime& operator=(const manual_lifetime&) = delete;
    manual_lifetime& operator=(manual_lifetime&&) = delete;

    template<typename Factory>
        requires
            std::invocable<Factory&> &&
            std::same_as<std::invoke_result_t<Factory&>, T>
    T& construct_from(Factory factory) noexcept(std::is_nothrow_invocable_v<Factory&>) {
        return *::new (static_cast<void*>(&storage)) T(factory());
    }

    void destroy() noexcept(std::is_nothrow_destructible_v<T>) {
        std::destroy_at(std::launder(reinterpret_cast<T*>(&storage)));
    }

    T& get() & noexcept {
        return *std::launder(reinterpret_cast<T*>(&storage));
    }

private:
    alignas(T) std::byte storage[sizeof(T)];
};
```

Note that the `construct_from()` method is designed to take a lambda here rather than
taking the constructor arguments. This allows us to make use of the guaranteed copy-elision
when initialising a variable with the result of a function-call to construct the object
in-place. If it were instead to take the constructor arguments then we'd end up calling
an extra move-constructor unnecessarily.

Now we can declare a data-member for the temporary returned by `promise.initial_suspend()`
using this `manual_lifetime` structure.

```c++
struct __g_state {
    __g_state(int&& x);

    int x;
    __g_promise_t __promise;
    manual_lifetime<std::suspend_always> __tmp1;
    // to be filled out
};
```

The `std::suspend_always` type does not have an `operator co_await()` so we do not need to
reserve storage for an extra temporary for the result of that call here.

Once we've constructed this object by calling `intial_suspend()`, we then need to call the
trio of methods to implement the `co_await` expression: `await_ready()`, `await_suspend()`
and `await_resume()`.

When invoking `await_suspend()` we need to pass it a handle to the current coroutine.
For now we can just call `std::coroutine_handle<__g_promise_t>::from_promise()` and pass
a reference to that promise. We'll look at the internals of what this does a little later.

Also, the result of the call to `.await_suspend(handle)` has type `void` and so we do not
need to consider whether to resume this coroutine or another coroutine after calling
`await_suspend()` like we do for the `bool` and `coroutine_handle`-returning flavours.

Finally, as all of the method invocations on the `std::suspend_always` awaiter are declared
`noexcept`, we don't need to worry about exceptions. If they were potentially throwing then
we'd need to add extra code to make sure that the temporary `std::suspend_always` object
was destroyed before the exception propagated out of the ramp function.

Once we get to the point where `await_suspend()` has returned successfully or
where we are about to start executing the coroutine body we enter the phase where we
no longer need to automatically destroy the coroutine-state if an exception is thrown.
So we can call `release()` on the `std::unique_ptr` owning the coroutine state to
prevent it from being destroyed when we return from the function.

So now we can implement the first part of the initial-suspend expression as follows:

```c++
task g(int x) {
    std::unique_ptr<__g_state> state(new __g_state(static_cast<int&&>(x)));
    decltype(auto) return_value = state->__promise.get_return_object();

    state->__tmp1.construct_from([&]() -> decltype(auto) {
        return state->__promise.initial_suspend();
    });
    if (!state->__tmp1.get().await_ready()) {
        //
        // ... suspend-coroutine here
        //
        state->__tmp1.get().await_suspend(
            std::coroutine_handle<__g_promise_t>::from_promise(state->__promise));

        state.release();

        // fall through to return statement below.
    } else {
        // Coroutine did not suspend.

        state.release();

        //
        // ... start executing the coroutine body
        //
    }
    return __return_val;
}
```

The call to `await_resume()` and the destructor of `__tmp1` will appear in the coroutine
body and so they do not appear in the ramp function.

We now have a (mostly) functional evaluation of the initial-suspend point, but we still
have a couple of TODO's in the code for this ramp function. To be able to resolve these we
will first need to take a detour to look at the strategy for suspending a coroutine and later
resuming it.

# Step 5: Recording the suspend-point

When a coroutine suspends, it needs to make sure it resumes at the same point in the
control flow that it suspended at.

It also needs to keep track of which objects with automatic-storage duration are alive
at each suspend-point so that it knows what needs to be destoyed if the coroutine is
destroyed instead of being resumed.

One way to implement this is to assign each suspend-point in the coroutine a unique number
and then store this in an integer data-member of the coroutine state.

Then whenever a coroutine suspends, it writes the number of the suspend-point at which it is
suspending to the coroutine state, and when it is resumed/destroyed we then inspect
this integer to see which suspend point it was suspended at.

Note that this is not the only way of storing the suspend-point in the coroutine state,
however all 3 major compilers (MSVC, Clang, GCC) use this approach as the time this post
was authored (c. 2022).
Another potential solution is to use separate resume/destroy function-pointers for each
suspend-point, although we will not be exploring this strategy in this post.

So let's extend our coroutine-state with an integer data-member to store the suspend-point
index and initialise it to zero (we'll always use this as the value for the initial-suspend
point).

```c++
struct __g_state {
    __g_state(int&& x);

    int x;
    __g_promise_t __promise;
    int __suspend_point = 0;  // <-- add the suspend-point index
    manual_lifetime<std::suspend_always> __tmp1;
    // to be filled out
};
```

# Step 6: Implementing `coroutine_handle::resume()` and `coroutine_handle::destroy()`

When a coroutine is resumed by calling `coroutine_handle::resume()` we need this to end up
invoking some function that implements the rest of the body of the suspended coroutine.
The invoked body function can then look up the suspend-point index and jump to the appropriate
point in the control-flow.

We also need to implement the `coroutine_handle::destroy()` function so that it invokes
the appropriate logic to destroy any in-scope objects at the current suspend-point and
we need to implement `coroutine_handle::done()` to query whether the current suspend-point
is a final-suspend-point.

The interface of the `coroutine_handle` methods does not know about the concrete coroutine
state type - the `coroutine_handle<void>` type can point to _any_ coroutine instance.
This means we need to implement them in a way that type-erases the coroutine state type.

We can do this by storing function-pointers to the resume/destroy functions for that coroutine
type and having `coroutine_handle::resume/destroy()` invoke those function-pointers.

The `coroutine_handle` type also needs to be able to be converted to/from a `void*` using the
`coroutine_handle::address()` and `coroutine_handle::from_address()` methods.

Furthermore, the coroutine can be resumed/destroyed from _any_ handle to that coroutine -
not just the handle that was passed to the most recent `await_suspend()` call.

These requirements lead us to define the `coroutine_handle` type so that it only contains a
pointer to the coroutine-state and that we store the resume/destroy function pointers as
data-members of the coroutine state, rather than, say, storing the resume/destroy function
pointers in the `coroutine_handle`.

Also, since we need the `coroutine_handle` to be able to point to an arbitrary coroutine-state
object we need the layout of the function-pointer data-members to be consistent across all
coroutine-state types.

One straight forward way of doing this is having each coroutine-state type inherit from some
base-class that contains these data-members.

e.g. We can define the following type as the base-class for all coroutine-state types
```c++
struct __coroutine_state {
    using __resume_fn = void(__coroutine_state*);
    using __destroy_fn = void(__coroutine_state*);

    __resume_fn* __resume;
    __destroy_fn* __destroy;
};
```

Then the `coroutine_handle::resume()` method can simply call `__resume()`, passing a
pointer to the `__coroutine_state` object.
Similarly, we can do this for the `coroutine_handle::destroy()` method and the `__destroy` function-pointer.

For the `coroutine_handle::done()` method, we choose to treat a null `__resume` function pointer
as an indication that we are at a final-suspend-point. This is convenient since the final suspend
point does not support `resume()`, only `destroy()`. If someone tries to call `resume()` on a
coroutine suspended at the final-suspend-point (which has undefined-behaviour) then they end up
calling a null function pointer which should fail pretty quickly and point out their error.

Given this, we can implement the `coroutine_handle<void>` type as follows:
```c++
namespace std
{
    template<typename Promise = void>
    class coroutine_handle;

    template<>
    class coroutine_handle<void> {
    public:
        coroutine_handle() noexcept = default;
        coroutine_handle(const coroutine_handle&) noexcept = default;
        coroutine_handle& operator=(const coroutine_handle&) noexcept = default;

        void* address() const {
            return static_cast<void*>(state_);
        }

        static coroutine_handle from_address(void* ptr) {
            coroutine_handle h;
            h.state_ = static_cast<__coroutine_state*>(ptr);
            return h;
        }

        explicit operator bool() noexcept {
            return state_ != nullptr;
        }
        
        friend bool operator==(coroutine_handle a, coroutine_handle b) noexcept {
            return a.state_ == b.state_;
        }

        void resume() const {
            state_->__resume(state_);
        }
        void destroy() const {
            state_->__destroy(state_);
        }

        bool done() const {
            return state_->__resume == nullptr;
        }

    private:
        __coroutine_state* state_ = nullptr;
    };
}
```

# Step 7: Implementing `coroutine_handle<Promise>::promise()` and `from_promise()`

For the more general `coroutine_handle<Promise>` specialisation, most of the implementations
can just reuse the `coroutine_handle<void>` implementations. However, we also need to be able
to get access to the promise object of the coroutine-state, returned from the `promise()` method,
and also construct a `coroutine_handle` from a reference to the promise-object.

However, again we cannot simply point to the concrete coroutine state type since the
`coroutine_handle<Promise>` type must be able to refer to any coroutine-state whose
promise-type is `Promise`.

We need to define a new coroutine-state base-class that inherits from `__coroutine_state`
and which contains the promise object so we can then define all coroutine-state types that use
a particular promise-type to inherit from this base-class.

```c++
template<typename Promise>
struct __coroutine_state_with_promise : __coroutine_state {
    __coroutine_state_with_promise() noexcept {}
    ~__coroutine_state_with_promise() {}

    union {
        Promise __promise;
    };
};
```

You might be wondering why we declare the `__promise` member inside an anonymous union here...

The reason for this is that the derived class created for a particular coroutine function
contains the definition for the argument-copy data-members. Data members from derived classes
are by default initialised after data-members of any base-classes, so declaring the promise
object as a normal data-member would mean that the promise object was constructed before the
argument-copy data-members.

However, we need the constructor of the promise to be called _after_ the constructor of the
argument-copies - references to the argument-copies might need to be passed to the promise
constructor.

So we reserve storage for the promise object in this base-class so that it has a consistent
offset from the start of the coroutine-state, but leave the derived class responsible for
calling the constructor/destructor at the appropriate point after the argument-copies have
been initialised. Declaring the `__promise` as a union-member provides this control.

Let's update the `__g_state` class to now inherit from this new base-class.

```c++
struct __g_state : __coroutine_state_with_promise<__g_promise_t> {
    __g_state(int&& __x)
    : x(static_cast<int&&>(__x)) {
        // Use placement-new to initialise the promise object in the base-class
        ::new ((void*)std::addressof(this->__promise))
            __g_promise_t(construct_promise<__g_promise_t>(x));
    }

    ~__g_state() {
        // Also need to manually call the promise destructor before the
        // argument objects are destroyed.
        this->__promise.~__g_promise_t();
    }

    int __suspend_point = 0;
    int x;
    manual_lifetime<std::suspend_always> __tmp1;
    // to be filled out
};
```

Now that we have defined the promise-base-class we can now implement the `std::coroutine_handle<Promise>` class template.

Most of the implementation should be largely identical to the equivalent methods in `coroutine_handle<void>`
except with a `__coroutine_state_with_promise<Promise>` pointer instead of `__coroutine_state` pointer.

The only new part is the addition of the `promise()` and `from_promise()` functions.

* The `promise()` method is straight-forward - it just returns a reference to the `__promise` member of the coroutine-state.
* The `from_promise()` method requires us to calculate the address of the coroutine-state from the
  address of the promise object. We can do this by just subtracting the offset of the `__promise` member
  from the address of the promise object.

Implementation of `coroutine_handle<Promise>`:
```c++
namespace std
{
    template<typename Promise>
    class coroutine_handle {
        using state_t = __coroutine_state_with_promise<Promise>;
    public:
        coroutine_handle() noexcept = default;
        coroutine_handle(const coroutine_handle&) noexcept = default;
        coroutine_handle& operator=(const coroutine_handle&) noexcept = default;

        operator coroutine_handle<void>() const noexcept {
            return coroutine_handle<void>::from_address(address());
        }

        explicit operator bool() const noexcept {
            return state_ != nullptr;
        }

        friend bool operator==(coroutine_handle a, coroutine_handle b) noexcept {
            return a.state_ == b.state_;
        }

        void* address() const {
            return static_cast<void*>(static_cast<__coroutine_state*>(state_));
        }

        static coroutine_handle from_address(void* ptr) {
            coroutine_handle h;
            h.state_ = static_cast<state_t*>(static_cast<__coroutine_state*>(ptr));
            return h;
        }

        Promise& promise() const {
            return state_->__promise;
        }

        static coroutine_handle from_promise(Promise& promise) {
            coroutine_handle h;

            // We know the address of the __promise member, so calculate the
            // address of the coroutine-state by subtracting the offset of
            // the __promise field from this address.
            h.state_ = reinterpret_cast<state_t*>(
                reinterpret_cast<unsigned char*>(std::addressof(promise)) -
                offsetof(state_t, __promise));

            return h;
        }

        // Define these in terms of their `coroutine_handle<void>` implementations

        void resume() const {
            static_cast<coroutine_handle<void>>(*this).resume();
        }

        void destroy() const {
            static_cast<coroutine_handle<void>>(*this).destroy();
        }

        bool done() const {
            return static_cast<coroutine_handle<void>>(*this).done();
        }

    private:
        state_t* state_;
    };
}
```

Now that we have defined the mechanism by which coroutines are resumed, we can now
return to our "ramp" function and update it to initialise the new function-pointer
data-members we've added to the coroutine-state.

# Step 8: The beginnings of the coroutine body

Let's now forward-declare resume/destroy functions of the right signature and
update the `__g_state` constructor to initialise the coroutine-state
so that the resume/destroy function-pointers point at them:

```c++
void __g_resume(__coroutine_state* s);
void __g_destroy(__coroutine_state* s);

struct __g_state : __coroutine_state_with_promise<__g_promise_t> {
    __g_state(int&& __x)
    : x(static_cast<int&&>(__x)) {
        // Initialise the function-pointers used by coroutine_handle methods.
        this->__resume = &__g_resume;
        this->__destroy = &__g_destroy;

        // Use placement-new to initialise the promise object in the base-class
        ::new ((void*)std::addressof(this->__promise))
            __g_promise_t(construct_promise<__g_promise_t>(x));
    }

    // ... rest omitted for brevity
};


task g(int x) {
    std::unique_ptr<__g_state> state(new __g_state(static_cast<int&&>(x)));
    decltype(auto) return_value = state->__promise.get_return_object();

    state->__tmp1.construct_from([&]() -> decltype(auto) {
        return state->__promise.initial_suspend();
    });
    if (!state->__tmp1.get().await_ready()) {
        state->__tmp1.get().await_suspend(
            std::coroutine_handle<__g_promise_t>::from_promise(state->__promise));
        state.release();
        // fall through to return statement below.
    } else {
        // Coroutine did not suspend. Start executing the body immediately.
        __g_resume(state.release());
    }
    return return_value;
}
```

This now completes the ramp function and we can now focus on the resume/destroy functions for `g()`.

Let's start by completing the lowering of the initial-suspend expression.

When `__g_resume()` is called and the `__suspend_point` index is 0 then we need it to
resume by calling `await_resume()` on `__tmp1` and then calling the destructor of `__tmp1`.

```c++
void __g_resume(__coroutine_state* s) {
    // We know that 's' points to a __g_state.
    auto* state = static_cast<__g_state*>(s);

    // Generate a jump-table to jump to the correct place in the code based
    // on the value of the suspend-point index.
    switch (state->__suspend_point) {
    case 0: goto suspend_point_0;
    default: std::unreachable();
    }

suspend_point_0:
    state->__tmp1.get().await_resume();
    state->__tmp1.destroy();

    // TODO: Implement rest of coroutine body.
    //
    //  int fx = co_await f(x);
    //  co_return fx * fx;
}
```

And when `__g_destroy()` is called and the `__suspend_point` index is 0 then we need it
to just destroy `__tmp1` before then destroying and freeing the coroutine-state.

```c++
void __g_destroy(__coroutine_state* s) {
    auto* state = static_cast<__g_state*>(s);

    switch (state->__suspend_point) {
    case 0: goto suspend_point_0;
    default: std::unreachable();
    }

suspend_point_0:
    state->__tmp1.destroy();
    goto destroy_state;

    // TODO: Add extra logic for other suspend-points here.

destroy_state:
    delete state;
}
```

# Step 9: Lowering the `co_await` expression

Next, let's take a look at lowering the `co_await f(x)` expression.

First we need to evaluate `f(x)` which returns a temporary `task` object.

As the temporary `task` is not destroyed until the semicolon at the end of the statement and
the statement contains a `co_await` expression, the lifetime of the `task` therefore spans a
suspend-point and so it must be stored in the coroutine-state.

When the `co_await` expression is then evaluated on this temporary `task`, we need to call
the `operator co_await()` method which returns a temporary `awaiter` object. The lifetime of
this object also spans the suspend-point and so must be stored in the coroutine-state.

Let's add the necessary members to the `__g_state` type:
```c++
struct __g_state : __coroutine_state_with_promise<__g_promise_t> {
    __g_state(int&& __x);
    ~__g_state();

    int __suspend_point = 0;
    int x;
    manual_lifetime<std::suspend_always> __tmp1;
    manual_lifetime<task> __tmp2;
    manual_lifetime<task::awaiter> __tmp3;
};
```

Then we can update the `__g_resume()` function to initialise these temporaries and then evaluate
the 3 `await_ready`, `await_suspend` and `await_resume` calls that comprise the rest of the
`co_await` expression.

Note that the `task::awaiter::await_suspend()` method returns a coroutine-handle so we need to
generate code that resumes the returned handle.

We also need to update the suspend-point index before calling `await_suspend()` (we'll use the
index 1 for this suspend-point) and then add an extra entry to the jump-table to ensure that
we resume back at the right spot.

```c++
void __g_resume(__coroutine_state* s) {
    // We know that 's' points to a __g_state.
    auto* state = static_cast<__g_state*>(s);

    // Generate a jump-table to jump to the correct place in the code based
    // on the value of the suspend-point index.
    switch (state->__suspend_point) {
    case 0: goto suspend_point_0;
    case 1: goto suspend_point_1; // <-- add new jump-table entry
    default: std::unreachable();
    }

suspend_point_0:
    state->__tmp1.get().await_resume();
    state->__tmp1.destroy();

    //  int fx = co_await f(x);
    state->__tmp2.construct_from([&] {
        return f(state->x);
    });
    state->__tmp3.construct_from([&] {
        return static_cast<task&&>(state->__tmp2.get()).operator co_await();
    });
    if (!state->__tmp3.get().await_ready()) {
        // mark the suspend-point
        state->__suspend_point = 1;

        auto h = state->__tmp3.get().await_suspend(
            std::coroutine_handle<__g_promise_t>::from_promise(state->__promise));
        
        // Resume the returned coroutine-handle before returning.
        h.resume();
        return;
    }

suspend_point_1:
    int fx = state->__tmp3.get().await_resume();
    state->__tmp3.destroy();
    state->__tmp2.destroy();

    // TODO: Implement
    //  co_return fx * fx;
}
```

Note that the `int fx` local variable has a lifetime that does not span a suspend-point and
so it does not need to be stored in the coroutine-state. We can just store it as a normal
local variable in the `__g_resume` function.

We also need to add the necessary entry to the `__g_destroy()` function to handle when
the coroutine is destroyed at this suspend-point.

```c++
void __g_destroy(__coroutine_state* s) {
    auto* state = static_cast<__g_state*>(s);

    switch (state->__suspend_point) {
    case 0: goto suspend_point_0;
    case 1: goto suspend_point_1; // <-- add new jump-table entry
    default: std::unreachable();
    }

suspend_point_0:
    state->__tmp1.destroy();
    goto destroy_state;

suspend_point_1:
    state->__tmp3.destroy();
    state->__tmp2.destroy();
    goto destroy_state;

    // TODO: Add extra logic for other suspend-points here.

destroy_state:
    delete state;
}
```

So now we have finished implementing the statement:
```c++
int fx = co_await f(x);
```

However, the function `f(x)` is not marked `noexcept` and so it can potentially throw an exception.
Also, the `awaiter::await_resume()` method is also not marked `noexcept` and can also potentially
throw an exception.

When an exception is thrown from a coroutine-body the compiler generates code to catch the exception
and then invoke `promise.unhandled_exception()` to give the promise an opportunity to do something
with the exception. Let's look at implementing this aspect next.

# Step 10: Implementing `unhandled_exception()`

The specification for coroutine definitions [`[dcl.fct.def.coroutine]`](https://eel.is/c++draft/dcl.fct.def.coroutine)
says that the coroutine behaves as if its function-body were replaced by:

```c++
{
    promise-type promise promise-constructor-arguments ;
    try {
        co_await promise.initial_suspend() ;
        function-body
    } catch ( ... ) {
        if (!initial-await-resume-called)
            throw ;
        promise.unhandled_exception() ;
    }
final-suspend :
    co_await promise.final_suspend() ;
}
```

We have already handled the `initial-await_resume-called` branch separately in the ramp function,
so we don't need to worry about that here.

Let's adjust the `__g_resume()` function to insert the try/catch block around the body.

Note that we need to be careful to put the `switch` that jumps to the right place inside
the try-block as we are not allowed to enter a try-block using a `goto`.

Also, we need to be careful to call `.resume()` on the coroutine handle returned from
`await_suspend()` outside of the try/catch block. If an exception is thrown from the
call `.resume()` on the returned coroutine then it should not be caught by the current
coroutine, but should instead propagate out of the call to `resume()` that resumed
this coroutine. So we stash the coroutine-handle in a variable declared at the top
of the function and then `goto` a point outside of the try/catch and execute the call
to `.resume()` there.

```c++
void __g_resume(__coroutine_state* s) {
    auto* state = static_cast<__g_state*>(s);

    std::coroutine_handle<void> coro_to_resume;

    try {
        switch (state->__suspend_point) {
        case 0: goto suspend_point_0;
        case 1: goto suspend_point_1; // <-- add new jump-table entry
        default: std::unreachable();
        }

suspend_point_0:
        state->__tmp1.get().await_resume();
        state->__tmp1.destroy();

        //  int fx = co_await f(x);
        state->__tmp2.construct_from([&] {
            return f(state->x);
        });
        state->__tmp3.construct_from([&] {
            return static_cast<task&&>(state->__tmp2.get()).operator co_await();
        });
        
        if (!state->__tmp3.get().await_ready()) {
            state->__suspend_point = 1;
            coro_to_resume = state->__tmp3.get().await_suspend(
                std::coroutine_handle<__g_promise_t>::from_promise(state->__promise));
            goto resume_coro;
        }

suspend_point_1:
        int fx = state->__tmp3.get().await_resume();
        state->__tmp3.destroy();
        state->__tmp2.destroy();

        // TODO: Implement
        //  co_return fx * fx;
    } catch (...) {
        state->__promise.unhandled_exception();
        goto final_suspend;
    }

final_suspend:
    // TODO: Implement
    // co_await promise.final_suspend();

resume_coro:
    coro_to_resume.resume();
    return;
}
```

There is a bug in the above code, however. In the case that the `__tmp3.get().await_resume()` call
exits with an exception, we would fail to call the destructors of `__tmp3` and `__tmp2` before
catching the exception.

Note that we cannot simply catch the exception, call the destructors and rethrow the exception
here as this would change the behaviour of those destructors if they were to call
`std::unhandled_exceptions()` since the exception would be "handled", however in the destructor
calls during unwind, the call to `std:::unhandled_exceptions()` should return non-zero.

We can instead define an RAII helper class to ensure that the destructors get called on scope
exit in the case an exception is thrown.

```c++
template<typename T>
struct destructor_guard {
    explicit destructor_guard(manual_lifetime<T>& obj) noexcept
    : ptr_(std::addressof(obj))
    {}

    // non-movable
    destructor_guard(destructor_guard&&) = delete;
    destructor_guard& operator=(destructor_guard&&) = delete;

    ~destructor_guard() noexcept(std::is_nothrow_destructible_v<T>) {
        if (ptr_ != nullptr) {
            ptr_->destroy();
        }
    }

    void cancel() noexcept { ptr_ = nullptr; }

private:
    manual_lifetime<T>* ptr_;
};

// Parital specialisation for types that don't need their destructors called.
template<typename T>
    requires std::is_trivially_destructible_v<T>
struct destructor_guard<T> {
    explicit destructor_guard(manual_lifetime<T>&) noexcept {}
    void cancel() noexcept {}
};

// Class-template argument deduction to simplify usage
template<typename T>
destructor_guard(manual_lifetime<T>& obj) -> destructor_guard<T>;
```

Using this utility, we can now use this type to ensure that variables stored in the coroutine-state
are destroyed when an exception is thrown.

Let's also use this class to call the destructors of the existing varibles so that it also calls
their destructors when they naturally go out of scope.

```c++
void __g_resume(__coroutine_state* s) {
    auto* state = static_cast<__g_state*>(s);

    std::coroutine_handle<void> coro_to_resume;

    try {
        switch (state->__suspend_point) {
        case 0: goto suspend_point_0;
        case 1: goto suspend_point_1; // <-- add new jump-table entry
        default: std::unreachable();
        }

suspend_point_0:
        {
            destructor_guard tmp1_dtor{state->__tmp1};
            state->__tmp1.get().await_resume();
        }

        //  int fx = co_await f(x);
        {
            state->__tmp2.construct_from([&] {
                return f(state->x);
            });
            destructor_guard tmp2_dtor{state->__tmp2};

            state->__tmp3.construct_from([&] {
                return static_cast<task&&>(state->__tmp2.get()).operator co_await();
            });
            destructor_guard tmp3_dtor{state->__tmp3};

            if (!state->__tmp3.get().await_ready()) {
                state->__suspend_point = 1;

                coro_to_resume = state->__tmp3.get().await_suspend(
                    std::coroutine_handle<__g_promise_t>::from_promise(state->__promise));

                // A coroutine suspends without exiting scopes.
                // So cancel the destructor-guards.
                tmp3_dtor.cancel();
                tmp2_dtor.cancel();

                goto resume_coro;
            }

            // Don't exit the scope here.
            //
            // We can't 'goto' a label that enters the scope of a variable with a
            // non-trivial destructor. So we have to exit the scope of the destructor
            // guards here without calling the destructors and then recreate them after
            // the `suspend_point_1` label.
            tmp3_dtor.cancel();
            tmp2_dtor.cancel();
        }

suspend_point_1:
        int fx = [&]() -> decltype(auto) {
            destructor_guard tmp2_dtor{state->__tmp2};
            destructor_guard tmp3_dtor{state->__tmp3};
            return state->__tmp3.get().await_resume();
        }();

        // TODO: Implement
        //  co_return fx * fx;
    } catch (...) {
        state->__promise.unhandled_exception();
        goto final_suspend;
    }

final_suspend:
    // TODO: Implement
    // co_await promise.final_suspend();

resume_coro:
    coro_to_resume.resume();
    return;
}
```

Now our coroutine body will now destroy local variables correctly in the presence of any exceptions and
will correctly call `promise.unhandled_exception()` if those exceptions propagate out of the coroutine
body.

It's worth noting here that there can also be special handling needed for the case where the
`promise.unhandled_exception()` method itself exits with an exception (e.g. if it rethrows
the current exception).

In this case, the coroutine would need to catch the exception, mark the coroutine as suspended
at a final-suspend-point, and then rethrow the exception.

For example: The `__g_resume()` function's catch-block would need to look like this:
```c++
try {
  // ...
} catch (...) {
    try {
        state->__promise.unhandled_exception();
    } catch (...) {
        state->__suspend_point = 2;
        state->__resume = nullptr; // mark as final-suspend-point
        throw;
    }
}
```
and we'd need to add an extra entry to the `__g_destroy` function's jump table:
```c++
switch (state->__suspend_point) {
case 0: goto suspend_point_0;
case 1: goto suspend_point_1;
case 2: goto destroy_state; // no variables in scope that need to be destroyed
                            // just destroy the coroutine-state object.
}
```

Note that in this case, the final-suspend-point is not necessarily the same suspend-point as the
final-suspend-point as the `co_await promise.final_suspend()` suspend-point.

This is because the `promise.final_suspend()` suspend-point will often have some extra temporary
objects related to the `co_await` expression which need to be destroyed when `coroutine_handle::destroy()`
is called. Whereas, if `promise.unhandled_exception()` exits with an exception then those temporary
objects will not exist and so won't need to be destroyed by `coroutine_handle::destroy()`.

# Step 11: Implementing `co_return`

The next step is to implement the `co_return fx * fx;` statement.

This is relatively straight-forward compared to some of the previous steps.

The `co_return <expr>` statement gets mapped to:
```c++
promise.return_value(<expr>);
goto final-suspend-point;
```

So we can simply replace the TODO comment with:
```c++
state->__promise.return_value(fx * fx);
goto final_suspend;
```

Easy.

# Step 12: Implementing `final_suspend()`

The final TODO in the code is now to implement the `co_await promise.final_suspend()` statement.

The `final_suspend()` method returns a temporary `task::promise_type::final_awaiter` type, which will
need to be stored in the coroutine-state and destroyed in `__g_destroy`.

This type does not have its own `operator co_await()`, so we don't need an additional temporary
object for the result of that call.

Like the `task::awaiter` type, this also uses the coroutine-handle-returning form of
`await_suspend()`. So we need to ensure that we call `resume()` on the returned handle.

If the coroutine does not suspend at the final-suspend-point then the coroutine-state is implicitly
destroyed. So we need to delete the state object if execution reaches the end of the coroutine.

Also, as all of the final-suspend logic is required to be noexcept, we don't need to worry about
exceptions being thrown from any of the sub-expressions here.

Let's first add the data-member to the `__g_state` type.
```c++
struct __g_state : __coroutine_state_with_promise<__g_promise_t> {
    __g_state(int&& __x);
    ~__g_state();

    int __suspend_point = 0;
    int x;
    manual_lifetime<std::suspend_always> __tmp1;
    manual_lifetime<task> __tmp2;
    manual_lifetime<task::awaiter> __tmp3;
    manual_lifetime<task::promise_type::final_awaiter> __tmp4; // <---
};
```

Then we can implement the body of the final-suspend expression as follows:
```c++
final_suspend:
    // co_await promise.final_suspend
    {
        state->__tmp4.construct_from([&]() noexcept {
            return state->__promise.final_suspend();
        });
        destructor_guard tmp4_dtor{state->__tmp4};

        if (!state->__tmp4.get().await_ready()) {
            state->__suspend_point = 2;
            state->__resume = nullptr; // mark as final suspend-point

            coro_to_resume = state->__tmp4.get().await_suspend(
                std::coroutine_handle<__g_promise_t>::from_promise(state->__promise));

            tmp4_dtor.cancel();
            goto resume_coro;
        }

        state->__tmp4.get().await_resume();
    }

    //  Destroy coroutine-state if execution flows off end of coroutine
    delete state;
    return;
```
 
And now we also need to update the `__g_destroy` function to handle this new suspend-point.
```c++
void __g_destroy(__coroutine_state* state) {
    auto* state = static_cast<__g_state*>(s);

    switch (state->__suspend_point) {
    case 0: goto suspend_point_0;
    case 1: goto suspend_point_1;
    case 2: goto suspend_point_2;
    default: std::unreachable();
    }

suspend_point_0:
    state->__tmp1.destroy();
    goto destroy_state;

suspend_point_1:
    state->__tmp3.destroy();
    state->__tmp2.destroy();
    goto destroy_state;

suspend_point_2:
    state->__tmp4.destroy();
    goto destroy_state;

destroy_state:
    delete state;
}
```

We now have a fully functional lowering of the `g()` coroutine function.

We're done! That's it!

Or is it....

# Step 13: Implementing symmetric-transfer and the noop-coroutine

It turns out there is actually a problem with the way we have implemented our `__g_resume()` function above.

The problems with this were discussed in more detail in the previous blog post so if you
want to understand the problem more deeply please take a look at the post
[C++ Coroutines: Understanding Symmetric Transfer](https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer).

The specification for [\[expr.await\]](https://eel.is/c++draft/expr.await) gives a little hint about
how we should be handling the coroutine-handle-returning flavour of `await_suspend`:
> If the type of _await-suspend_ is `std​::​coroutine_­handle<Z>`, _await-suspend_`.resume()` is evaluated.
> 
> [_Note_ 1: This resumes the coroutine referred to by the result of _await-suspend_.
> Any number of coroutines can be successively resumed in this fashion, eventually returning control
> flow to the current coroutine caller or resumer ([\[dcl.fct.def.coroutine\]](https://eel.is/c++draft/dcl.fct.def.coroutine)). —- _end note_]

The note there, while non-normative and thus non-binding, is strongly encouraging compilers to 
implement this in such a way that it performs a tail-call to resume the next coroutine rather
than resuming the next coroutine recursively. This is because resuming the next coroutine
recursively can easily lead to unbounded stack growth if coroutines resume each other in a loop.

The problem is that we are calling `.resume()` on the next coroutine from within the body of
the `__g_resume()` function and then returning, so the stack space used by the `__g_resume()`
frame is not freed until after the next coroutine suspends and returns.

Compilers are able to do this by implementing the resumption of the next coroutine as a
tail-call. In this way, the compiler generates code that first pops the the current stack
frame, preserving the return-address, and then executes a `jmp` to the next coroutine's
resume-function.

As we don't have a mechanism in C++ to specify that a function-call in the tail-position
should be a tail-call we will need to instead actually return from the resume-function so
that its stack-space can be freed, and then have the caller resume the next coroutine.

As the next coroutine may also need to resume another coroutine when it suspends, and this
may happen indefinitely, the caller will need to resume the coroutines in a loop.

Such a loop is typically called a "trampoline loop" as we return back to the loop from one
coroutine and then "bounce" off the loop back into the next coroutine.

If we modify the signature of the resume-function to return a pointer to the next coroutine's
coroutine-state instead of returning void, then the `coroutine_handle::resume()` function can
then just immediately call the `__resume()` function-pointer for the next coroutine to resume
it.

Let's change the signature of the `__resume_fn` for a `__coroutine_state`:
```c++
struct __coroutine_state {
    using __resume_fn = __coroutine_state* (__coroutine_state*);
    using __destroy_fn = void (__coroutine_state*);

    __resume_fn* __resume;
    __destroy_fn* __destroy;
};
```

Then we can write the `coroutine_handle::resume()` function something like this:
```c++
void std::coroutine_handle<void>::resume() const {
    __coroutine_state* s = state_;
    do {
        s = s->__resume(s);
    } while (/* some condition */)
}
```

The next question then becomes: "What should the condition be?"

This is where the `std::noop_coroutine()` helper comes into the picture.

The `std::noop_coroutine()` is a factory function that returns a special coroutine
handle that has a no-op `resume()` and `destroy()` method. If a coroutine suspends
and returns the noop-coroutine-handle from the `await_suspend()` method then this
indicates that there is no more coroutine to resume and that the invocation of
`coroutien_handle::resume()` that resumed this coroutine should return to its caller.

So we need to implement `std::noop_coroutine()` and the condition in `coroutine_handle::resume()`
so that the condition returns false and the loop exits when the `__coroutine_state` pointer
points to the noop-coroutine-state.

One strategy we can use here is to define a static instance of `__coroutine_state` that is
designated as the noop-coroutine-state. The `std::noop_coroutine()` function can return a
coroutine-handle that points to this object, and we can compare the `__coroutine_state`
pointer to the address of that object to see if a particular coroutine handle is the
noop-coroutine.

First let's define this special noop-coroutine-state object:
```c++
struct __coroutine_state {
    using __resume_fn = __coroutine_state* (__coroutine_state*);
    using __destroy_fn = void (__coroutine_state*);

    __resume_fn* __resume;
    __destroy_fn* __destroy;

    static __coroutine_state* __noop_resume(__coroutine_state* state) noexcept {
        return state;
    }

    static void __noop_destroy(__coroutine_state*) noexcept {}

    static const __coroutine_state __noop_coroutine;
};

inline const __coroutine_state __coroutine_state::__noop_coroutine{
    &__coroutine_state::__noop_resume,
    &__coroutine_state::__noop_destroy
};
```

Then we can implement the `std::coroutine_handle<noop_coroutine_promise>` specialisation.
```c++
namespace std
{
    struct noop_coroutine_promise {};

    using noop_coroutine_handle = coroutine_handle<noop_coroutine_promise>;

    noop_coroutine_handle noop_coroutine() noexcept;

    template<>
    class coroutine_handle<noop_coroutine_promise> {
    public:
        constexpr coroutine_handle(const coroutine_handle&) noexcept = default;
        constexpr coroutine_handle& operator=(const coroutine_handle&) noexcept = default;

        constexpr explicit operator bool() noexcept { return true; }

        constexpr friend bool operator==(coroutine_handle, coroutine_handle) noexcept {
            return true;
        }

        operator coroutine_handle<void>() const noexcept {
            return coroutine_handle<void>::from_address(address());
        }

        noop_coroutine_promise& promise() const noexcept {
            static noop_coroutine_promise promise;
            return promise;
        }

        constexpr void resume() const noexcept {}
        constexpr void destroy() const noexcept {}
        constexpr bool done() const noexcept { return false; }

        constexpr void* address() const noexcept {
            return const_cast<__coroutine_state*>(&__coroutine_state::__noop_coroutine);
        }
    private:
        constexpr coroutine_handle() noexcept = default;

        friend noop_coroutine_handle noop_coroutine() noexcept {
            return {};
        }
    };
}
```

And we can update `coroutine_handle::resume()` to exit when the noop-coroutine-state
is returned.

```c++
void std::coroutine_handle<void>::resume() const {
    __coroutine_state* s = state_;
    do {
        s = s->__resume(s);
    } while (s != &__coroutine_state::__noop_coroutine);
}
```

And finally, we can update our `__g_resume()` function to now return the `__coroutine_state*`.

This just involves updating the signature and replacing:
```c++
coro_to_resume = ...;
goto resume_coro;
```
with
```c++
auto h = ...;
return static_cast<__coroutine_state*>(h.address());
```

and then at the very end of the function (after the `delete state;` statement) adding
```c++
return static_cast<__coroutine_state*>(std::noop_coroutine().address());
```

# One last thing

Those with a keen eye may have noticed that the coroutine-state type `__g_state` is actually
larger than it needs to be.

The data-members for the 4 temporary values each reserve storage for their respective values.
However, the lifetimes of some of the temporary values do not overlap and so in theory we can
save space in the coroutine-state by reusing the storage of an object for the next object after
its lifetime has ended.

To be able to take advantage of this we can instead define the data-members in an anonymous
union where appropriate

Looking at the lifetimes of the temporary varaibles we have:
- `__tmp1` - exists only within `co_await promise.initial_suspend();` statement
- `__tmp2` - exists only within `int fx = co_await f(x);` statement
- `__tmp3` - exists only within `int fx = co_await f(x);` statement - nested inside lifetime of `__tmp2`
- `__tmp4` - exists only within `co_await promise.final_suspend();` statement

Since lifetimes of `__tmp2` and `__tmp3` overlap we must place them in a struct together
as they both need to exist at the same time.

However, the `__tmp1` and `__tmp4` members do not have lifetimes that overlap and so they can
be placed together in an anonymous `union`.

Thus we can change our data-member definition to:
```c++
struct __g_state : __coroutine_state_with_promise<__g_promise_t> {
    __g_state(int&& x);
    ~__g_state();

    int __suspend_point = 0;
    int x;

    struct __scope1 {
        manual_lifetime<task> __tmp2;
        manual_lifetime<task::awaiter> __tmp3;
    };

    union {
        manual_lifetime<std::suspend_always> __tmp1;
        __scope1 __s1;
        manual_lifetime<task::promise_type::final_awaiter> __tmp4;
    };
};
```

Then, because the `__tmp2` and `__tmp3` variables are now nested inside the `__s1` object,
we need to update references to them to now be e.g. `state->__s1.tmp2`. But otherwise the
rest of the code stays the same.

This should save an additional 16 bytes of the coroutine-state size as we no longer need
extra storage + padding for the `__tmp1` and `__tmp4` data-members - which would otherwise
be padded to the size of a pointer, despite being empty types.

# Tying it all together

Ok, so the final code we have generated for the coroutine function:
```c++
task g(int x) {
    int fx = co_await f(x);
    co_return fx * fx;
}
```

is the following:
```c++
/////
// The coroutine promise-type

using __g_promise_t = std::coroutine_traits<task, int>::promise_type;

__coroutine_state* __g_resume(__coroutine_state* s);
void __g_destroy(__coroutine_state* s);

/////
// The coroutine-state definition

struct __g_state : __coroutine_state_with_promise<__g_promise_t> {
    __g_state(int&& x)
    : x(static_cast<int&&>(x)) {
        // Initialise the function-pointers used by coroutine_handle methods.
        this->__resume = &__g_resume;
        this->__destroy = &__g_destroy;

        // Use placement-new to initialise the promise object in the base-class
        // after we've initialised the argument copies.
        ::new ((void*)std::addressof(this->__promise))
            __g_promise_t(construct_promise<__g_promise_t>(this->x));
    }

    ~__g_state() {
        this->__promise.~__g_promise_t();
    }

    int __suspend_point = 0;

    // Argument copies
    int x;

    // Local variables/temporaries
    struct __scope1 {
        manual_lifetime<task> __tmp2;
        manual_lifetime<task::awaiter> __tmp3;
    };

    union {
        manual_lifetime<std::suspend_always> __tmp1;
        __scope1 __s1;
        manual_lifetime<task::promise_type::final_awaiter> __tmp4;
    };
};

/////
// The "ramp" function

task g(int x) {
    std::unique_ptr<__g_state> state(new __g_state(static_cast<int&&>(x)));
    decltype(auto) return_value = state->__promise.get_return_object();

    state->__tmp1.construct_from([&]() -> decltype(auto) {
        return state->__promise.initial_suspend();
    });
    if (!state->__tmp1.get().await_ready()) {
        state->__tmp1.get().await_suspend(
            std::coroutine_handle<__g_promise_t>::from_promise(state->__promise));
        state.release();
        // fall through to return statement below.
    } else {
        // Coroutine did not suspend. Start executing the body immediately.
        __g_resume(state.release());
    }
    return return_value;
}

/////
//  The "resume" function

__coroutine_state* __g_resume(__coroutine_state* s) {
    auto* state = static_cast<__g_state*>(s);

    std::coroutine_handle<void> coro_to_resume;

    try {
        switch (state->__suspend_point) {
        case 0: goto suspend_point_0;
        case 1: goto suspend_point_1; // <-- add new jump-table entry
        default: std::unreachable();
        }

suspend_point_0:
        {
            destructor_guard tmp1_dtor{state->__tmp1};
            state->__tmp1.get().await_resume();
        }

        //  int fx = co_await f(x);
        {
            state->__s1.__tmp2.construct_from([&] {
                return f(state->x);
            });
            destructor_guard tmp2_dtor{state->__s1.__tmp2};

            state->__s1.__tmp3.construct_from([&] {
                return static_cast<task&&>(state->__s1.__tmp2.get()).operator co_await();
            });
            destructor_guard tmp3_dtor{state->__s1.__tmp3};

            if (!state->__s1.__tmp3.get().await_ready()) {
                state->__suspend_point = 1;

                auto h = state->__s1.__tmp3.get().await_suspend(
                    std::coroutine_handle<__g_promise_t>::from_promise(state->__promise));

                // A coroutine suspends without exiting scopes.
                // So cancel the destructor-guards.
                tmp3_dtor.cancel();
                tmp2_dtor.cancel();

                return static_cast<__coroutine_state*>(h.address());
            }

            // Don't exit the scope here.
            // We can't 'goto' a label that enters the scope of a variable with a
            // non-trivial destructor. So we have to exit the scope of the destructor
            // guards here without calling the destructors and then recreate them after
            // the `suspend_point_1` label.
            tmp3_dtor.cancel();
            tmp2_dtor.cancel();
        }

suspend_point_1:
        int fx = [&]() -> decltype(auto) {
            destructor_guard tmp2_dtor{state->__s1.__tmp2};
            destructor_guard tmp3_dtor{state->__s1.__tmp3};
            return state->__s1.__tmp3.get().await_resume();
        }();

        //  co_return fx * fx;
        state->__promise.return_value(fx * fx);
        goto final_suspend;
    } catch (...) {
        state->__promise.unhandled_exception();
        goto final_suspend;
    }

final_suspend:
    // co_await promise.final_suspend
    {
        state->__tmp4.construct_from([&]() noexcept {
            return state->__promise.final_suspend();
        });
        destructor_guard tmp4_dtor{state->__tmp4};

        if (!state->__tmp4.get().await_ready()) {
            state->__suspend_point = 2;
            state->__resume = nullptr; // mark as final suspend-point

            auto h = state->__tmp4.get().await_suspend(
                std::coroutine_handle<__g_promise_t>::from_promise(state->__promise));

            tmp4_dtor.cancel();
            return static_cast<__coroutine_state*>(h.address());
        }

        state->__tmp4.get().await_resume();
    }

    //  Destroy coroutine-state if execution flows off end of coroutine
    delete state;

    return static_cast<__coroutine_state*>(std::noop_coroutine().address());
}

/////
// The "destroy" function

void __g_destroy(__coroutine_state* s) {
    auto* state = static_cast<__g_state*>(s);

    switch (state->__suspend_point) {
    case 0: goto suspend_point_0;
    case 1: goto suspend_point_1;
    case 2: goto suspend_point_2;
    default: std::unreachable();
    }

suspend_point_0:
    state->__tmp1.destroy();
    goto destroy_state;

suspend_point_1:
    state->__s1.__tmp3.destroy();
    state->__s1.__tmp2.destroy();
    goto destroy_state;

suspend_point_2:
    state->__tmp4.destroy();
    goto destroy_state;

destroy_state:
    delete state;
}

```

For a fully compilable version of the final code, see:
[https://godbolt.org/z/xaj3Yxabn](https://godbolt.org/z/xaj3Yxabn)

This concludes the 5-part series on understanding the mechanics of C++ coroutines.

This is probably more information than you ever wanted to know about coroutines, but
hopefully it helps you to understand what's going on under the hood and demystifies
them just a bit.

Thanks for making it through to the end!

Until next time, Lewis.
