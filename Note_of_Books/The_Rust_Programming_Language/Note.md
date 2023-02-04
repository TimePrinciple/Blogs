## Integer Overflow

- Compiling in debug mode, Rust includes checks for integer overflow that cause the program to *panic* at runtime if this behavior occurs.
- Compiling in release mode (with --release flag), Rust does not include checks for integer overflow that cause panics. Instead, if overflow occurs, Rust performs *two's complement wrapping*.

> To explicitly handle the possibility of overflow, `wrapping_<operation>` (like `wrapping_add`) methods to handle overflow or `checked_<operation>` returns the `None` if there is overflow.

## Expression-based Language

- Statements are instructions that perform some action and do not return a value.
    - Creating a variable and assigning a value to it with the `let` keyword is a statement.
    - Function definitions.
    - Ends with `;` marks a statement.
- Expressions evaluate to a resultant value.
    - A math operation, such as `5 + 6` which is an expression that evaluates to the value `11`.
    - Calling functions, macros.
    - A new scope block created with curly brackets.

## Returning Values from Loops

One use of a `loop` is to retry an operation might fail, such as checking whether a thread has completed its job. There also might be a need to pass the result of that operation out of the loop to the rest of code. To do this, add the value to be returned after the `break` expression, then that value will be returned out of the loop to be used.

## The Stack and the Heap

Both the stack and the heap are parts of memory available to code to use at runtime, but they are structured in different ways.

- Stack: stores values in the order it gets them and removes the values in the opposite order. This is referred to as *last in, first out*. Adding data is called *pushing onto the stack* and removing data is called *popping off the stack*. **All data stored on the stack must have a known, fixed size**.
- Heap: less organized. When put data on the heap, a request for a certain amount of space is made. The memory allocator finds an empty spot in the heap that is big enough, marks it as being in use, and returns a *pointer*, which is the address of that location. This process is called *allocating on the heap* and is sometimes abbreviated as just *allocating* (pushing values onto the stack is not considered allocating). Because the pointer to the heap is a known, fixed size, the pointer can be stored on the stack, but when the actual data is needed, follow the pointer to find the data.

Pushing to the stack is faster than allocating on the heap because the allocator never has to search for a place to store new data; that location is always at the top of the stack. Comparatively, allocating space on the heap requires more work because the allocator must first find a big enough space to hold the data and then perform bookkeeping to prepare for the next allocation.

Accessing data in the heap is slower than accessing data on the stack because the process of following the pointer (the pointer or reference is on the stack, the stack is first accessed) to there. Contemporary processors are faster if they jump around less in memory.

When a function is called, the values passed into the function (including, potentially, pointers to data on the heap) and the function's local variables get pushed onto the stack. When the function is over, those values get popped off the stack.

## Ownership

Ownership is Rust's most unique feature and has deep implications for the rest of the language. It enables Rust to make memory safety guarantees without needing a garbage collector (GC).

### Rules

- Each value in Rust has an owner.
- There can only be one owner at a time.
- When the owner goes out of scope, the value will be dropped.

## `Copy` and `Drop`

Rust won't allow a type with `Copy` if the type, or any of its parts, has implemented the `Drop` trait (might cause multiple-free).

## Restriction on Mutable Reference

Preventing multiple mutable references to the same data at the same time allows for mutation but in a very controlled fashion. The benefit of having this restriction is that Rust can prevent data races at compile time. A *data race* is similar to a race condition and happens when these tree behaviors occur:

- Two or more pointers access the same data at the same time.
- At least one of the pointers is being used to write to the data.
- There's no mechanism being used to synchronize access to the data.

Data races cause undefined behavior and can be difficult to diagnose and fix when trying to track them down at runtime; Rust prevents this problem by refusing to compile code with data races.

## Scope of Reference

Starts from where it is introduced (declared) and continues through the last time that reference is used.

## Rules of Reference

- At any given time, there can be either one mutable reference or any number of immutable references.
- References must always be valid.

## Struct

To use a struct after we've defined it, we create an *instance* of that struct by specifying concrete values for each of the fields. To create an instance by stating the name of the struct and then add curly brackets containing *key: value* pairs, where the keys are the names of the fields and the values are the data we want to store in those fields. These fields can be specified in a different order as which we declared them in the struct.

> In other words, the struct definition is like a general template for the type, and instances fill in that template with particular data to create values of the type.

## Reference on Containers

When a program has a valid reference, the borrow checker enforces the ownership and borrowing rules to ensure this reference and any other references to the contents of the vector remain valid.

Why should a reference to the first element care about changes at the end of the vector? This error is due to the way vectors work: because vectors put the values next to each other in memory, adding a new element onto the end of the vector might require allocating new memory and copying the old elements to the new space, if there isn't enough room to put all the elements next to each other where the vector is currently stored. In that case, the reference to the first element would be pointing to deallocated memory. The borrowing rules prevent programs from ending up in that situation.

## Hasher in HashMap

By default, `HashMap` uses a hashing function called *SipHash* that can provide resistance to Denial of Service (DoS) attacks involving hash tables. This is not the fastest hashing algorithm available, but the trade-off for better security that comes with the drop in performance is worth it.

## Unwinding the Stack or Aborting in Response to a Panic

By default, when a panic occurs, the program starts *unwinding*, which means Rust walks back up the stack and cleans up the data from each function it encounters. However, this walking back and cleanup is a lot of work. Rust, therefore, allows to choose the alternative of immediately *aborting*, which ends the program without cleaning up.

Memory that the program was using will then need to be cleaned up by the operating system. If in the project the resulting binary is needed to be made as small as possible, switch from unwinding to aborting upon a panic by adding `panic = 'abort'` to the appropriate `[profile]` section in the *Cargo.toml* file.

## Performance of Generics

Using generic types won't make the program run any slower than it would with concrete types. Rust accomplishes this be performing *monomorphization* of the code using generics at compile time. 

*Monomorphization* is the process of turning generic code into specific code by filling in the concrete types that are used when compiled. In this process, the compiler does the opposite of the steps used to create the generic function: the compiler looks at all the places where generic code called and generates code for the concrete types the generic code is called with. Because Rust compiles generic code into code that specifies the type in each instance, so there is no runtime cost for using generics. When the code runs, it performs just as it would if each definition is duplicated by hand. The process of monomorphization makes Rust's generics extremely efficient at runtime.

## Lifetime

Lifetime on function or method parameters are called *input lifetimes*, and lifetimes on return values are called *output lifetimes*.

### Rules of Lifetime

1. The compiler assigns a lifetime parameter to each parameter that's a reference. In other words, a function with one parameter gets one lifetime parameter: `fn foo<'a>(x: &'a i32)`; a function with two parameters gets two separate lifetime parameters: `fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`; and so on.
2. If there is exactly one input lifetime parameter, that lifetime is assigned to all output lifetime parameters: `fn foo<'a(x: &'a i32) -> &'a i32`.
3. If there are multiple input lifetime parameters, but one of them is `&self` or `&mut self` because this is a method, the lifetime of `self` is assigned to all output lifetime parameters. This third rule makes methods much nicer to read and write because fewer symbols are necessary.

The compiler uses three rules to figure out the lifetimes of the references when there aren't explicit annotations. The first rule applies to input lifetimes, and the second and third rules apply to output lifetimes. If the compiler gets to the end of the three rules and there are still references for which it can't figure out lifetimes, the compiler will stop with an error. These rule apply to `fn` definitions as well as `impl` blocks.

## Write Unit Tests

Add the block (usually at the end of the file):
```Rust
#[cfg(test)]
mod tests {
    use super::*; // Import code outside this test block but still in this file.

    #[test]
    #[should_panic] // Check the code handles error conditions as expected.
    #[ignore] // Exclude this test while `cargo test`, run ignored test with `cargo -- --ignored`, run all tests with `cargo -- --include-ignored`.
    fn test_func() {
        // --snip--
        
        // Assertions.
    }
}
```
Then run the tests with `cargo test`.

```
$ cargo test -- --test-threads=1
```
Set the number of test threads to `1`, telling the program not to use any parallelism. Running the tests using one thread will take longer than running them in parallel, but the tests won't interfere with each other if they share state.

```
$ cargo test -- --show-output
```
Show printed values for passing tests.

## Closures

Once a closure has captured a reference or captured ownership of a value from the environment where the closure is defined (thus affecting what, if anything, is moved *into* the closure), the code in the body of the closure defines what happens to the references or values when the closure is evaluated later (thus affecting what, if anything, is moved *out of* the closure). A closure body can do any of the following: move a captured value out of the closure, mutate the captured value, neither move nor mutate the value, or capture nothing from the environment to begin with.

The way a closure captures and handles values from the environment affects which traits the closure implements, and traits are how functions and structs can specify what kinds of closures they can use. Closures will automatically implement one, two, or all three of these `Fn` traits, in an additive fashion, depending on how the closure's body handles the values:
1. `FnOnce` applies to closures that can be called once. All closures implement at least this trait, because all closures can be called. A closure that moves captured values out of its body will only implement `FnOnce` and none of the other `Fn` traits, because it can only be called once.
2. `FnMut` applies to closures that don't move captured values out of their body, but that might mutate the captured values. These closures can be called more than once.
3. `Fn` applies to closures that don't move captured values out of their body and that don't mutate captured values, as well as closures that capture nothing from their environment. These closures can be called more than once without mutating their environment, which is important in case such as calling a closure multiple times concurrently.

## Pointer

A *pointer* is a general concept for a variable that contains an address in memory. This address refers to, or "points at", some other data. The most common kind of pointer in Rust is a reference.

## Smart Pointer

Data structures that act like a pointer but also have additional metadata and capabilities.

Rust, with its concept of ownership and borrowing, has an additional difference between references and smart pointers:
- References only borrow data.
- Smart pointers own the data.

Smart pointers are usually implemented using structs. Unlike an ordinary struct, smart pointers implement the `Deref` and `Drop` traits. The `Deref` trait allows an instance of the smart pointer struct to behave like a reference. The `Drop` trait allows to customize the code that's run when an instance of the smart pointer goes out of scope.

### `Box<T>`

The most straightforward  smart pointer is a *box*, whose type is written `Box<T>`. Boxes allow to store data on the heap rather than on the stack. What remains on the stack is the pointer to the heap data. Box don't have performance overhead, other than storing their data on the heap instead of on the stack. Used in situations like:
- When there is a type whose size can't be known at compile time and a need to use a value of that type in a context that requires an exact size. (Like recursive type)
- When there is a large amount of data and a need to transfer ownership but ensure the data won't be copied when do so.
- When there is a need to own a value and the only thing matters is whether the type implements a particular trait rather than being of a specific type.

## Drop

The point of the `Drop` trait is that it's taken care of automatically. Rust doesn't allow to call the `Drop` trait's `drop` method manually; instead to call the `std::mem::drop` function provided by the standard library if there is a need to force a value to be dropped before the end of its scope.

Variables are dropped in the reverse order of their creation. (Stack unwinding)

The ownership  system that makes sure references are always valid also ensures that `drop` gets called only once when the value is no longer being used.

## `Rc<T>`

> For use in single-threaded scenarios.

The reference counted smart pointer. This type keeps track of the number of references to a value to determine whether or not the value is still in use. If there are zero references to a value, the value can be cleaned up without any references becoming invalid.

Use the `Rc<T>` type when there is a need to allocate some data on the heap for multiple parts of the program to read and can't determine at compile time which part will finish using the data last.

## `RefCell<T>`

> For use in single-threaded scenarios.

Unlike `Rc<T>`, the `RefCell<T>` type represents single ownership over the data it holds. With references and `Box<T>`, the borrowing rules' invariants are enforced at compile time. With `RefCell<T>`, these invariants are enforced at *runtime*. With references, if these rules were broke, a compiler error will generate. With `RefCell<T>`, if these rules were broke, the program will panic and exit.

The advantages of checking the borrowing rules at compile time are that errors will be caught sooner in the development process, and there is no impact on runtime performance because all the analysis is completed beforehand. For those reasons, checking the borrowing rules at compile time is the best choice in the majority of cases, which is why this is Rust's default.

The advantages of checking the borrowing rules at runtime instead is that certain memory-safe scenarios are then allowed, where they would've been disallowed by the compile-time checks. Static analysis, like the Rust compiler, is inherently conservative. Some properties of code are impossible to detect by analyzing the code: the most famous example is the Halting Problem.

Because some analysis is impossible, if the Rust compiler can't be sure the code compiles with the ownership rules, it might reject a correct program; in this way, it's conservative. If Rust accepted an incorrect program, users wouldn't be able to trust in the guarantees Rust makes. However, if Rust rejects a correct program, the programmer will be inconvenienced, but nothing catastrophic can occur. The `RefCell<T>` type is useful when the programmer is sure the code follows the borrowing rules but the compiler is unable to understand and guarantee that.

#### Halting Problem

> In computability theory, the **halting problem** is the problem of determining, from a description of an arbitrary computer program and an input, whether the program will finish running, or continue to run forever. Alan Turing proved in 1936 that a general algorithm to solve the halting problem for all possible program-input pairs cannot exist.

For any program ***f*** that might determine whether programs halt, a "pathological" problem ***g***, called with some input, can pass its own source and its input to ***f*** and then specifically do the opposite of what ***f*** predicts ***g*** will do. No ***f*** can exist that handles this case.

A key part of the proof is a mathematical definition of a computer and program, which is known as a Turing machine, the halting problem is undecidable over Turing machines. It is one of the first cases of decision problems proven to be unsolvable. This proof is significant to practical computing efforts, defining a class of applications which no programming invention can possibly perform perfectly.

## Process and Thread

An executed program's code is run in a *process*, and the operating system will manage multiple processes at once. Within a program, you can also have independent parts that run simultaneously. The feature that run these independent parts are called *threads*.

Splitting the computation in the program into multiple threads to run multiple tasks at the same time can improve performance, but it also adds complexity. Because thread can run simultaneously, there's no inherent guarantee about the order in which parts of the code on different threads will run. This can lead to problems, such as:
- Race conditions, where threads are accessing data or resources in an inconsistent order.
- Deadlocks, where two threads are waiting for each other, preventing both threads from continuing.
- Bugs that happen only in certain situations and are hard to reproduce and fix reliably.

Programming languages implement threads in a few different ways, and many operating systems provide an API the language can call for creating new threads. The Rust standard library uses a *1:1* model of thread implementation, whereby a program uses one operating system thread per one language thread.

## Concurrency

Handling concurrent programming safely and efficiently is another of Rust's major goals:
- Concurrent programming: where different parts of a program execute independently.
- Parallel programming: where different parts of a program execute at the same time.

### Message-passing concurrency

`use std::sync::mpsc  // Multiple producer, single consumer`

The ownership rules play a vital role in message sending because they help you write safe, concurrent code. Preventing errors in concurrent programming is the advantage of thinking about ownership throughout the Rust programs. Once a value has been sent to another thread, that thread could modify or drop it before try to use the value again. Potentially, the other thread's modifications could cause errors or unexpected results due to inconsistent or nonexistent data. The `send` function takes ownership of its parameter, and when the value is moved, the receiver takes ownership of it. This stops accidentally using the value again after sending it; the ownership system checks that everything is okay.

### Shared-State Concurrency

> In a way, channels in any programming language are similar to single ownership, because once a value is transferred down a channel, that value should no longer be used. Shared memory concurrency is like multiple ownership: multiple threads can access the same memory location at the same time.

#### `Mutex<T>`

Mutex (Mutual Exclusion) allows only one thread to access some data at any given time. To access the data in a mutex, a thread must first signal that it wants access by asking to acquire the mutex's *lock*. The lock is a data structure that is part of the mutex that keeps track of who currently has exclusive access to the data. Therefore, the mutex is described as *guarding* the data it holds via the locking system.

Rules of mutex:
- Attempt to acquire the lock before using the data.
- Unlock the data when done so other threads can acquire the lock.

#### `Arc<T>`

> Thread safety comes with a performance penalty.

`Rc<T>` is not safe to share across threads. When `Rc<T>` manages the reference count, it adds to the count for each call to `clone` and subtracts from the count when each clone is dropped. **But it doesn't use any concurrency primitives to make sure that changes to the count can't be interrupted by another thread**. This could lead to wrong counts--subtle bugs that could in turn lead to memory leaks or a value being dropped before it is done using. Therefore need a type exactly like `Rc<T>` but one that makes changes to the reference count in a thread-safe way.

*a* stands for *atomic* (concurrency primitive, thus not interruptible?), meaning it's an *automatically reference counted*. A type like `Rc<T>` that is safe to use in concurrent situations. 

> `Rc<T>` & `RefCell<T>` and `Arc<T>` & `Mutex<T>`

## Object Oriented Programming

> Object-oriented programs are make up of objects. An *object* packages both data and the procedures that operate on that data. The procedures are typically called *methods* or *operations*.

Follow this definition, Rust is object-oriented: `struct` and `enum` have data, and `impl` blocks provide methods on structs and enums. Even though structs and enums with methods aren't *called* obejcts, they provide the same functionality.

> *Encapsulation*: the implementation details of an object aren't accessible to code using that object. Therefore, the only way to interact with an object is through its public API; code using the object shouldn't be able to reach into the object's internals and change data or behavior directly. This enables the programmer to change and refactor an object's internals without needing to change the code that uses the object.

If encapsulation is a required aspect for a language to be considered object-oriented, then Rust meets that requirement. The option to use `pub` or not for different parts of code enables encapsulation of implementation details.

> *Inheritance*: a mechanism whereby an object can inherit elements from another object's definition, thus gaining the parent object's data and behavior without having to define them again.

If a language must have inheritance to be an object-oriented language, then Rust is not one. There is no way to define a struct that inherits the parent struct's fields and method implementations without using a macro.

Inheritance would be chosen for two main reasons. One is for reuse of code: To implement particular behavior for one type, and inheritance enables to reuse that implementation for a different type. This can be achieved by using default trait method implementations (the default implementation can be override). The other reason to use inheritance relates to the type system: to enable a child type to be used in the same places as the parent type. This is also called *polymorphism*, which meas that multiple objects can be substitute for each other at runtime if they share certain characteristics.

> *Polymorphism*: is synonymous with inheritance. But it's actually a more general concept that refers to code that can work with data of multiple types. For inheritance, those types are generally subclasses.

Rust instead uses generics to abstract over different possible types and trait bounds to impose constraints on what those types must provide. This is sometimes called *bounded parametric polymorphism*.

> Inheritance has recently fallen out of favor as a programming design solution in many programming languages because it's often at risk of sharing more code than necessary. Subclasses shouldn't always share all characteristics of their parent class but will do so with inheritance. This can make a program's design less flexible. It also introduces the possibility of calling methods on subclasses that don't make sense or that cause errors because the methods don't apply to the subclass. In addition, some languages will only allow single inheritance (meaning a subclass can only inherit from one class), further restricting the flexibility of a program's design.

## Trait Object

The concept--of being concerned only with the messages a value responds to rather than the values concrete type--is similar to the concept of *duck typing* in dynamically typed languages: if it walks like a duck and quacks like a duck, then it must be a duck.

The advantage of using trait objects and Rust's type system to write code similar to code using duck typing is it frees us from checking whether a value implements a particular method at runtime or worrying about getting errors if a value doesn't implement a method but gets called anyway. Rust won't compile the code if the values don't implement the traits that the trait objects need.

- Static dispatch: Monomorphization process performed by the compiler when use trait bounds on generics: the compiler generates non-generic implementations of functions and methods for each concrete type that used in place of a generic type parameter. The code that results from monomorphization is doing *static dispatch*, which is when the compiler knows what method will get called at compile time.
- Dynamic dispatch: when using trait objects, Rust must use dynamic dispatch. The compiler doesn't know all the types that might be used with the code that's using trait objects, so it doesn't know which method implemented on which type to call. Instead, at runtime, Rust uses the pointers inside the trait object to know which method to call. This lookup incurs a runtime cost that doesn't occur with static dispatch. Dynamic dispatch also prevents the compiler from choosing to inline a method's code, which in turn prevents some optimizations. However, the extra flexibility in the code is gained, so it's a trade-off to consider.
