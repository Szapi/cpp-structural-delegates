_This document was created with the help of GitHub Copilot to speed up the process._

# Motivation

## Problem Statement

While C++ provides multiple mechanisms for abstraction and polymorphism—including inheritance, templates, and type erasure (e.g., `std::function`)—nominal typing and inheritance remain the primary built-in approaches for defining interfaces and enabling runtime polymorphism.
These approaches, when misused or omitted altogether (e.g. no clear indication for when they should be used, bad design decisions, lack of time/budget, inheriting a legacy codebase), can lead to tightly coupled designs—especially in large codebases, where classes can accumulate excessive responsibilities.
Such "hotspot" classes become difficult to extend, reason about, break up and refactor, or unit test, even though they contain critical business logic.
They tend to accumulate even more code over time if left unattended.
Even worse, codes that depend on such classes (e.g. take them by reference), are also rendered un-testable and un-modular.

Though there may be other situations where the proposed solution would shine, this was my main motivation to look for a better solution.

Writing software is difficult. It's hard to know in advance if an abstraction we create today, will also be useful in a couple years. Or in a decade.
There may be externally imposed restrictions or requirements on the codebase.
There may be discrepancies or miscommunication between software architects and the engineers that bring their ideas to life, even if everyone is skilled and strives for perfection. I believe most of us do. These are my thought and observations after having worked on a couple larger (multi-million lines of code) C++ projects in the past 7 years.

## Problem Example A:
Consider the following class:
```cpp
class component_configs
{
public:
    // Networking-related configs
    int max_connections() const;
    int conn_timeout_ms() const;
    int conn_port() const;
    int conn_protocol_version() const;
    ...

    // Logging-related configs
    int         logging_level() const;
    int         max_log_size_mb() const;
    string_view log_file_path() const;
    ...

    // Imagine dozens or hundreds of settings
};
```
This class is likely to be passed all around the codebase as `const component_configs&` without second thoughts, as it represents the ground truth to parts or all of our code.
This class, coupled with (external or member) methods, may be a useful abstraction, as it unites all settings that come from a single source, can be responsible for serialization, printing the settings to a `std::ostream&`, may provide default values, data validation, overrides, etc.

To test dependent codes, we must instantiate this class in every unit test. Is there a better way?

### _Virtual base classes_

We could break up the interface to multiple base classes, like `i_communication_configs`, `i_logging_configs`, etc, and make `component_configs` derive from all of them.
But we might also need `i_backup_service_configs` that requires a subset of the methods of `i_communication_configs`, and then an `i_remote_access_configs` that requires a different subset, and so on.
It violates the open-closed principle, as `component_configs` must change its structure to accomodate to the needs of the consumer code, and is not very elegant.
We would also need to create separate mocks and fakes for testing.

### _Just use structs_
A similar problem will arise if we try to assemble bigger structs out of smaller structs:
```cpp
struct backup_service_configs
{
    int conn_port;
    ...
};

struct remote_access_configs
{
    int conn_port;
    ...
};

...

struct communication_configs
{
    backup_service_configs backup;
    remote_access_configs  remote_access;
    // Now we have duplicate variables for conn_port, but they were intended to be the same
    // --> single source of truth?
    ...
};
```
This may also violate the open-closed principle again.

We could just keep the original class, and then copy all necessary settings into a temporary struct, and pass that to methods when needed, but this also requires at least some boilerplate...

There are other ways and methods to deal with these situations, but there is usually some kind of compromise to be made due to the rigidity of a nominal type system.

## Problem Example B:

It is quite common for a software module or component to develop singletons, which are major sources of dependency issues and present similar issues as discussed before.
Singletons (using `static` means, or just simply instantiated once) often travel around as references, and act as central hubs, perhaps for callbacks, signaling, data access, synchronization point, or as a supervisor of operations.
They are often 'wired together' upon construction, and depend on each other.
Conventional means of decoupling and refactoring can help, but may not be feasible or worth the effort, or may not be effective enough.
Refactoring large chunks of code is tedious and mundane, small mistakes can creep in, causing new bugs.

## Structural Typing with Templates

Templates can do exactly what we are looking for, however, their use for this kind of code decoupling is not practical, and would surely result in massive recompilation times and bloat by forcing virtually all code into headers.

### _Example: Decoupling a single method_
```cpp
template<class component_configs_t>
void do_something(const component_configs_t& configs, ...)
{
    // Only a handful of methods are required for component_configs_t
    const int timeout = configs.conn_timeout_ms();
    const int port    = configs.conn_port();
    ...
}
```

### _Example: Decoupling an overgrown class_
```cpp
template<
    class work_thread_t,
    class component_configs_t,
    class trace_engine_t,
    class error_handler_t,
    class offline_backup_strategy_t,
    class mutex_t,
    ...
    // These types were previously hard-wired, and lack polymorphism support for e.g. testing.
    // Will only ever be instantiated with a single set of types in production code,
    // but cannot be unittested otherwise...
>
class storage_administrator
{
    // Very important singleton for managing multiple open databases
public:
    template<class work_thread_factory_t>
    storage_administrator(const component_configs_t& configs,
                          work_thread_factory_t&     thread_factory,
                          trace_engine_t&            tracing,
                          error_handler_t&           err_handler
                          ...);

private:
    const component_configs_t& configs;
    std::vector<work_thread_t> workers;
    trace_engine_t& tracing;
    error_handler_t& err_handler;
    ...
};
```

This is hardly practical, but it would work for decoupling classes from each other. If only there was a way to do this without templates, in a fashion that still fits the core principles of C++...

## Proposed Solution: Primitive Structural Class Delegates

A structural delegate could be defined with a new keyword like so:
```cpp
delegate communication_configs
{
    // A list of simple (non-static, non-template) member function declarations, possibly with
    // const/volatile/noexcept qualifiers, as these are part of the function signature.
    int max_connections() const;
    int conn_timeout_ms() const;

    // Possibly default values and nodiscard attribute.
    // Ret, P1, P2 are concrete types.
    [[nodiscard]] Ret foo(const P1& p1, const P2* p2 = nullptr) noexcept;
    // Overloaded functions are OK
    void foo(int a, int b, double d);
    ...

    // No access specifiers.
    // No constructors, member variables, operators, etc. are allowed (for now)
};
```

From this delegate definition, the compiler could automatically generate:
 - a C++ 20 concept that checks if a class's public interface matches with the delegate,
 - a class definition (the actual delegate) that has the same public interface, can bind to classes that have the same set of public named methods, and forward the method calls to the underlying object.

The delegate class should be
 - lightweight,
 - trivially copyable,
 - usable as if we had a reference to the underlying (bound) object, as in, the stated methods can be called, with proper const-correctness, and perfect argument forwarding,
 - predeclarable, just like a regular class or struct.

This is already doable with existing C++ features, altough each delegate class must be coded manually.

### Rudimentary Implementation with Existing C++ Tools

#### Interface Concept
The generated concept would be equivalent to this:
```cpp
template<class T>
concept communication_configs_concept = requires(T& mut, const T& cnst)
{
    // max_connections()
    { cnst.max_connections() } -> std::same_as<int>;      // Check if max_connections is a public method
    static_cast<int (T::*)() const>(&T::max_connections); // Check the exact signature

    // Ret foo(const P1&, const P2* p2) noexcept
    requires requires (const P1& p1, const P2* p2)
    {
        { mut.foo(p1, p2) } -> std::same_as<Ret>;
    };
    static_cast<Ret (T::*)(const P1&, const P2*) noexcept>(&T::foo);

    // void foo(int, int, double)
    requires requires (int a, int b, double d)
    {
        mut.foo(a, b, d);
    };
    static_cast<void (T::*)(int,int,double)>(&T::foo);
};
```

#### Delegate Class
The delegate itself is a simple, inline class, with type erasure, and an 'ad-hoc function table':

```cpp
class communication_configs_delegate final
{
public:
    int max_connections() const;
    int conn_timeout_ms() const;

    [[nodiscard]] Ret foo(const P1& p1, const P2* p2 = nullptr) noexcept;
    void foo(int a, int b, double d);
    ...

    // No public constructor for binding, explained shortly...
private:
    struct FnTable
    {
        std::tuple<
            int (*)(const void*)
        > max_connections;

        std::tuple<
            int (*)(const void*)
        > conn_timeout_ms;

        std::tuple<
            Ret (*)(void*,const P1&,const P2*) noexcept,
            void (*)(void*,int,int,double)
        > foo;
    };

    void* obj;
    const FnTable* fnTable;
};

// Function implementations
inline Ret communication_configs_delegate::foo(const P1& p1, const P2* p2) noexcept
{
    return std::get<
        Ret (*)(void*,const P1&,const P2*) noexcept // Choose the correct signature
    >(*(this->fnTable)) (this->obj, p1, p2);
}
...
```

How the function table can be filled when constructing a delegate:
```cpp
template<class T> requires communication_configs_concept<T>
consteval FnTable MakeFnTable()
{
    struct Invoker
    {
        static int max_connections(const void* obj) { return static_cast<const T*>(obj)->max_connections(); }
        ...
    };

    return FnTable {
        .max_connections = std::make_tuple(&Invoker::max_connections),
        ...
    };
}
```

Delegate binding must be explicitly stated, e.g.
```cpp
communication_configs configs;
auto del = delegate_cast<communication_configs_delegate>(communication_configs);
```

Let us address some early concerns regarding the delegates.

#### Possible loss of strong nominal contracts (proof-of-intention)

Nominal typing gives explicitness: we can see in the class definition which interfaces it intends to support (it derives from them explicitly, overrides virtual methods).

Structural typing by delegate means it’s easier to silently mismatch semantics: just because a class has `draw(canvas&)` doesn’t mean it’s meaningful as a `renderable`.

Solution: explicit opt-in for delegation as a proof-of-intention.
Imagine the following code block as if it was a compilation unit:
```cpp
// Implicitly 'assumed' by the compiler
template<class delegate, class underlying>
constexpr bool allow_delegate_bind = false;

// Included from one header
// Notice: my_class doesn't necessarily need to know about my_delegate to be compatible
class my_callbacks { ... }

// Included from another header
delegate my_delegate { ... };
void do_work(my_delegate callback);

// Contents of .cpp

// TODO: Come up with syntax for these
// State explicitly that my_class was intended to be bound to a my_delegate
// Notice: This can be done 'retroactively' without the delegate or the class having to know about each other
static_assert(my_delegate_concept<my_class>);
template<> constexpr bool allow_delegate_bind<my_delegate, my_class> = true;

void work()
{
    my_class callbacks;
    do_work(delegate_cast<my_delegate>(callbacks);
}
```

#### Possible object lifetime issues
The exact same concerns already apply to references, so there's nothing new here.
Delegates are not meant to manage the lifetime of the underlying object, they should behave just like references. If lifetime management is needed, it should be done with conventional methods.

#### Passing delegates through ABI boundaries
The exact same concerns already apply to regular polymorphic objects passed as pointer-to-base (e.g. DLL of the underlying object is unloaded early). Delegates are not primarily meant to be long-lived or passed between code modules, they are best used within a single module, in a closed ecosystem, though they do open up possibilities for libraries. Further consideration is needed.

### Benefits of Delegates

_To be continued..._