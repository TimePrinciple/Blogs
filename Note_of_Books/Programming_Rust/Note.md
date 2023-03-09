# Programming Rust

## Fundamental Types

Rust uses the `u8` type for byte values. For example, reading data from a binary file or socket yields a stream of `u8` values.

Rust treats characters as distinct from the numeric types: a `char` is not a `u8`, not is it a `u32` (though it is 32 bits long).

The `usize` and `isize` types are analogous to `size_t` and `ptrdiff_t` in C and C++. Their precision matches the size of address space on the target machine: they are 32 bits long on 32-bit architectures, and 64 bits long on 64-bit architectures. Rust requires array indices to be `usize` values.

The prefixes `0x`, `0o`, and `0b` designate hexadecimal, octal, and binary literals.

Although numeric types and the `char` type are distinct, Rust does provide *byte literals*, character-like literals for `u8` values: `b'A'` represents the ASCII code for the character A, as a `u8` value. For example, since the ASCII code for A is 65, the literal `b'A'` and `65u8` are exactly equivalent. **Only ASCII characters may appear in byte literals**.

Rust uses an entire byte for a `bool` value in memory, so you can create a pointer to it.

Rust code often uses `tuple` types to return multiple values from a function. One commonly used tuple type is the `zero-tuple ()`, this is traditionally called the *unit type* where there's no meaningful value to carry, but context requires some sort of type nonetheless (the `void` concept). There are even tuples that contain a single value. The literal `("lonely", )` is a tuple containing a single string; its type is `(&str, )`. Here, the comma after the value is necessary to distinguish the singleton tuple from a simple parenthetic expression.

### Pointers

At run time, a reference to an `i32` is a single machine word holding the address of the `i32`, which may be on the stack or in the heap. The expression `&x` produces a reference to `x`; in Rust terminology, we say that it *borrows a reference* to `x`. Given a reference `r`, the expression `*r` refers to the value `r` points to.

Rust references come in two flavors:
- `&T`: An immutable, shared reference. You can have many shared references to a given value at a time, but they are read-only: modifying the value they point to is forbidden, as with `const T*` in C.
- `&mut T`: A mutable, exclusive reference. You can read and modify the value it points to, as with a `T*` in C. But for as long as the reference exists, you may **not have any other references of any kind to that value**. In fact, the only way you may access the value at all is through the mutable reference.

"Single writer or multiple readers".

### Boxes

The simplest way to allocate a value in the **heap** is to use `Box::new()`.

### Raw Pointers

`*mut T` and `*const T`. Using raw pinter is unsafe, because Rust makes no effort to track what it points to. And they can be only dereferenced in `unsafe` block.

## Arrays, Vectors and Slices

- The type `[T; N]` represents an array of `N` values, each of type `T`. An array's size is a constant determined at compile time and is part of the type; it cannot be appended or narrowed.
- The type `Vec<T>`, called a vector of `T`s, is a dynamically allocated, growable sequence of values of type `T`. A vector's elements live on the heap, so its size is allowed to change.
- The type `&[T]` and `&mut [T]`, called a *shared slice* of `T`s and *mutable slice* of `T`s, are references to a series of elements that are a part of some other value, like an array or vector. It can be thought as a **pointer to its first element, together with a count of the number of elements you can access starting at that point**.

### Arrays

For the common case of a long array filled with some value, `[V; N]` can be used, where `V` is the value each element should have, and `N` is the length. The `N` cannot be a variable.

### Vectors

There are several ways to create vectors. The simplest is to use the `vec!` macro, which gives us a syntax for vectors that looks very much like an array literal. `vec![V; N]` can also be used, but this time the `N` can be a variable.

A `Vec<T>` consists of three values: a *pointer* to the heap-allocated buffer for the elements, which is created and owned by the `Vec<T>`; the number of elements that buffer has the *capacity* to store; and the number it actually contains now (its *length*).

> When the buffer has reached its capacity, adding another element to the vector entails allocating a larger buffer, copying the present contents into it, updating the vector's pointer and capacity to describe the new buffer, and finally freeing the old one.
> If the number of elements a vector will need can be known in advance, instead of `Vec::new`, `Vec::with_capacity` should be called to create a vector with a buffer large enough to hold them all, right from the start; then elements can be added to the vector without causing any reallocation. The `vec!` macro uses a trick like this.

### Slices

A slice, written `[T]` without specifying the length, is a region of an array or vector. Since a slice can be any length, slices can't be stored directly in variables or passed as function arguments. Slices are always passed by reference.

A reference to a slice is a `fat pointer`, a two-word value comprising a *pointer* to the slice's first element, and the *number* of elements in the slice.

## String Types

### String Literals

A raw string is tagged with the lowercase letter `r`.

### Byte Strings

A string literal with the `b` prefix is a `byte string`. Such a string is a slice of `u8` values (bytes) rather than Unicode text. Raw byte strings start with `br`.

### Strings in Memory

Rust strings are sequences of Unicode characters, but they are not stored in memory as arrays of `char`s. Instead, they are stored using `UTF-8`, a variable-width encoding. Each ASCII character in a string is stored in one byte. Other characters take up multiple bytes.

A `String` has a resizable buffer holding UTF-8 text.

A string literal is a `&str` that refers to preallocated text, typically stored in read-only memory along with the program's machine code.

A `String` or `&str`'s `.len()` method returns its length. The length is measured in bytes, not characters, to count by characters, use `.chars().count()` instead.

### `&str`

A `&str` cannot be modified. For creating new strings at run time, using `String`.  
The type `&mut str` does exist, but it is not very useful, since almost any operation on UTF-8 can change its overall byte length, and a slice cannot reallocate its referent. In fact, the only operations available on `&mut str` are `make_ascii_uppercase` and `make_ascii_lowercase`, which modify the text in place and affect only single-byte characters, by definition.

> Arrays, slices, and vectors of strings have two methods, `.concat()` and `.join(sep)`, that form a `String` from many strings:
```Rust
let bits = vec!["veni", "vidi", "vici"];
assert_eq!(bits.concat(), "venividivici");
assert_eq!(bits.join(", "), "veni, vidi, vici");
```
### Other String-Like Types

- Stick to `String` and `&str` for Unicode text.
- When working with filenames, use `std::path::PathBuf` and `&Path` instead.
- When working with binary data that isn't UTF-8 encoded at all, use `Vec<u8>` and `&[u8]`.
- When working with environment variable names and command-line arguments in the native form presented by the operating system, use `OsString` and `&OsStr`.
- When interoperating with C libraries that use null-terminated strings, use `std::ffi::CString` and `&CStr`.

## Type Aliases

The `type` keyword can be used like `typedef` in C++ to declare a new name for an existing type:
```Rust
type Bytes = Vec<u8>;
```

