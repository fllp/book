
[TOC]

# Smart Pointers

A *pointer* is a general concept for a variable that contains an address in
memory. This address refers to, or “points at”, some other data. The most
common kind of pointer in Rust is a *reference*, which we learned about in
Chapter 4. References are indicated by the `&` symbol and borrow the value that
they point to. They don’t have any special abilities other than referring to
data. They also don’t have any overhead, so they’re used the most often.

*Smart pointers*, on the other hand, are data structures that act like a
pointer, but they also have additional metadata and capabilities. The concept
of smart pointers isn’t unique to Rust; it originated in C++ and exists in
other languages as well. The different smart pointers defined in Rust’s
standard library provide extra functionality beyond what references provide.
One example that we’ll explore in this chapter is the *reference counting*
smart pointer type, which enables you to have multiple owners of data. The
reference counting smart pointer keeps track of how many owners there are, and
when there aren’t any remaining, the smart pointer takes care of cleaning up
the data.

In Rust, where we have the concept of ownership and borrowing, an additional
difference between references and smart pointers is that references are a kind
of pointer that only borrow data; by contrast, in many cases, smart pointers
*own* the data that they point to.

We’ve actually already encountered a few smart pointers in this book, such as
`String` and `Vec<T>` from Chapter 8, though we didn’t call them smart pointers
at the time. Both these types count as smart pointers because they own some
memory and allow you to manipulate it. They also have metadata (such as their
capacity) and extra capabilities or guarantees (such as `String` ensuring its
data will always be valid UTF-8).

Smart pointers are usually implemented using structs. The characteristics that
distinguish a smart pointer from an ordinary struct are that smart pointers
implement the `Deref` and `Drop` traits. The `Deref` trait allows an instance
of the smart pointer struct to behave like a reference so that we can write
code that works with either references or smart pointers. The `Drop` trait
allows us to customize the code that gets run when an instance of the smart
pointer goes out of scope. In this chapter, we’ll be discussing both of those
traits and demonstrating why they’re important to smart pointers.

Given that the smart pointer pattern is a general design pattern used
frequently in Rust, this chapter won’t cover every smart pointer that exists.
Many libraries have their own smart pointers and you can even write some
yourself. We’ll just cover the most common smart pointers from the standard
library:

* `Box<T>` for allocating values on the heap
* `Rc<T>`, a reference counted type that enables multiple ownership
* `Ref<T>` and `RefMut<T>`, accessed through `RefCell<T>`, a type that enforces
  the borrowing rules at runtime instead of compile time

Along the way, we’ll cover the *interior mutability* pattern where an immutable
type exposes an API for mutating an interior value. We’ll also discuss
*reference cycles*, how they can leak memory, and how to prevent them.

Let’s dive in!

## `Box<T>` Points to Data on the Heap and Has a Known Size

The most straightforward smart pointer is a *box*, whose type is written
`Box<T>`. Boxes allow you to store data on the heap rather than the stack. What
remains on the stack is the pointer to the heap data. Refer back to Chapter 4
if you’d like to review the difference between the stack and the heap.

Boxes don’t have performance overhead other than their data being on the heap
instead of on the stack, but they don’t have a lot of extra abilities either.
They’re most often used in these situations:

- When you have a type whose size can’t be known at compile time, and you want
  to use a value of that type in a context that needs to know an exact size
- When you have a large amount of data and you want to transfer ownership but
  ensure the data won’t be copied when you do so
- When you want to own a value and only care that it’s a type that implements a
  particular trait rather than knowing the concrete type itself

We’re going to demonstrate the first case in the rest of this section. To
elaborate on the other two situations a bit more: in the second case,
transferring ownership of a large amount of data can take a long time because
the data gets copied around on the stack. To improve performance in this
situation, we can store the large amount of data on the heap in a box. Then,
only the small amount of pointer data is copied around on the stack, and the
data stays in one place on the heap. The third case is known as a *trait
object*, and Chapter 17 has an entire section devoted just to that topic. So
know that what you learn here will be applied again in Chapter 17!

### Using a `Box<T>` to Store Data on the Heap

Before we get into a use case for `Box<T>`, let’s get familiar with the syntax
and how to interact with values stored within a `Box<T>`.

Listing 15-1 shows how to use a box to store an `i32` on the heap:

Filename: src/main.rs

```
fn main() {
    let b = Box::new(5);
    println!("b = {}", b);
}
```

Listing 15-1: Storing an `i32` value on the heap using a box

We define the variable `b` to have the value of a `Box` that points to the
value `5`, which is allocated on the heap. This program will print `b = 5`; in
this case, we can access the data in the box in a similar way as we would if
this data was on the stack. Just like any value that has ownership of data,
when a box goes out of scope like `b` does at the end of `main`, it will be
deallocated. The deallocation happens for both the box (stored on the stack)
and the data it points to (stored on the heap).

Putting a single value on the heap isn’t very useful, so you won’t use boxes by
themselves in the way that Listing 15-1 does very often. Having values like a
single `i32` on the stack, where they’re stored by default is more appropriate
in the majority of cases. Let’s get into a case where boxes allow us to define
types that we wouldn’t be allowed to if we didn’t have boxes.

### Boxes Enable Recursive Types

Rust needs to know at compile time how much space a type takes up. One kind of
type whose size can’t be known at compile time is a *recursive type* where a
value can have as part of itself another value of the same type. This nesting
of values could theoretically continue infinitely, so Rust doesn’t know how
much space a value of a recursive type needs. Boxes have a known size, however,
so by inserting a box in a recursive type definition, we are allowed to have
recursive types.

Let’s explore the *cons list*, a data type common in functional programming
languages, to illustrate this concept. The cons list type we’re going to define
is straightforward except for the recursion, so the concepts in this example
will be useful any time you get into more complex situations involving
recursive types.

A cons list is a list where each item in the list contains two things: the
value of the current item and the next item. The last item in the list contains
only a value called `Nil` without a next item.

#### More Information About the Cons List

A *cons list* is a data structure that comes from the Lisp programming language
and its dialects. In Lisp, the `cons` function (short for “construct function”)
constructs a new list from its two arguments, which usually are a single value
and another list.

The cons function concept has made its way into more general functional
programming jargon; “to cons x onto y” informally means to construct a new
container instance by putting the element x at the start of this new container,
followed by the container y.

A cons list is produced by recursively calling the `cons` function. The
canonical name to denote the base case of the recursion is `Nil`, which
announces the end of the list. Note that this is not the same as the “null” or
“nil” concept from Chapter 6, which is an invalid or absent value.

Note that while functional programming languages use cons lists frequently,
this isn’t a commonly used data structure in Rust. Most of the time when you
have a list of items in Rust, `Vec<T>` is a better choice. Other, more complex
recursive data types *are* useful in various situations in Rust, but by
starting with the cons list, we can explore how boxes let us define a recursive
data type without much distraction.

Listing 15-2 contains an enum definition for a cons list. Note that this
won’t compile quite yet because this type doesn’t have a known size, which
we’ll demonstrate:

Filename: src/main.rs

```
enum List {
    Cons(i32, List),
    Nil,
}
```

Listing 15-2: The first attempt of defining an enum to represent a cons list
data structure of `i32` values

> Note: We’re choosing to implement a cons list that only holds `i32` values
> for the purposes of this example. We could have implemented it using
> generics, as we discussed in Chapter 10, in order to define a cons list type
> that could store values of any type.

Using our cons list type to store the list `1, 2, 3` would look like the code
in Listing 15-3:

Filename: src/main.rs

```
use List::{Cons, Nil};

fn main() {
    let list = Cons(1, Cons(2, Cons(3, Nil)));
}
```

Listing 15-3: Using the `List` enum to store the list `1, 2, 3`

The first `Cons` value holds `1` and another `List` value. This `List`
value is another `Cons` value that holds `2` and another `List` value. This
is one more `Cons` value that holds `3` and a `List` value, which is finally
`Nil`, the non-recursive variant that signals the end of the list.

If we try to compile the above code, we get the error shown in Listing 15-4:

```
error[E0072]: recursive type `List` has infinite size
 -->
  |
1 | enum List {
  | ^^^^^^^^^ recursive type has infinite size
2 |     Cons(i32, List),
  |               ----- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to
  make `List` representable
```

Listing 15-4: The error we get when attempting to define a recursive enum

The error says this type ‘has infinite size’. The reason is the way we’ve
defined `List` is with a variant that is recursive: it holds another value of
itself directly. This means Rust can’t figure out how much space it needs in
order to store a `List` value. Let’s break this down a bit: first let’s look at
how Rust decides how much space it needs to store a value of a non-recursive
type.

### Computing the Size of a Non-Recursive Type

Recall the `Message` enum we defined in Listing 6-2 when we discussed enum
definitions in Chapter 6:

```
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

To determine how much space to allocate for a `Message` value, Rust goes
through each of the variants to see which variant needs the most space. Rust
sees that `Message::Quit` doesn’t need any space, `Message::Move` needs enough
space to store two `i32` values, and so forth. Since only one variant will end
up being used, the most space a `Message` value will need is the space it would
take to store the largest of its variants.

Contrast this to what happens when Rust tries to determine how much space a
recursive type like the `List` enum in Listing 15-2 needs. The compiler starts
by looking at the `Cons` variant, which holds a value of type `i32` and a value
of type `List`. Therefore, `Cons` needs an amount of space equal to the size of
an `i32` plus the size of a `List`. To figure out how much memory the `List`
type needs, the compiler looks at the variants, starting with the `Cons`
variant. The `Cons` variant holds a value of type `i32` and a value of type
`List`, and this continues infinitely, as shown in Figure 15-1.

<img alt="An infinite Cons list" src="img/trpl15-01.svg" class="center" style="width: 50%;" />

Figure 15-1: An infinite `List` consisting of infinite `Cons` variants

### Using `Box<T>` to Get a Recursive Type with a Known Size

Rust can’t figure out how much space to allocate for recursively defined types,
so the compiler gives the error in Listing 15-4. The error does include this
helpful suggestion:

```
= help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to
        make `List` representable
```

In this suggestion, “indirection” means that instead of storing a value
directly, we’re going to store the value indirectly by storing a pointer to
the value instead.

Because a `Box<T>` is a pointer, Rust always knows how much space a `Box<T>`
needs: a pointer’s size doesn’t change based on the amount of data it’s
pointing to.

So we can put a `Box` inside the `Cons` variant instead of another `List` value
directly. The `Box` will point to the next `List` value that will be on the
heap, rather than inside the `Cons` variant. Conceptually, we still have a list
created by lists “holding” other lists, but the way this concept is implemented
is now more like the items being next to one another rather than inside one
another.

We can change the definition of the `List` enum from Listing 15-2 and the usage
of the `List` from Listing 15-3 to the code in Listing 15-5, which will compile:

Filename: src/main.rs

```
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let list = Cons(1,
        Box::new(Cons(2,
            Box::new(Cons(3,
                Box::new(Nil))))));
}
```

Listing 15-5: Definition of `List` that uses `Box<T>` in order to have a known
size

The `Cons` variant will need the size of an `i32` plus the space to store the
box’s pointer data. The `Nil` variant stores no values, so it needs less space
than the `Cons` variant. We now know that any `List` value will take up the
size of an `i32` plus the size of a box’s pointer data. By using a box, we’ve
broken the infinite, recursive chain so the compiler is able to figure out the
size it needs to store a `List` value. Figure 15-2 shows what the `Cons`
variant looks like now:

<img alt="A finite Cons list" src="img/trpl15-02.svg" class="center" />

Figure 15-2: A `List` that is not infinitely sized since `Cons` holds a `Box`

Boxes only provide the indirection and heap allocation; they don’t have any
other special abilities like those we’ll see with the other smart pointer
types. They also don’t have any performance overhead that these special
abilities incur, so they can be useful in cases like the cons list where the
indirection is the only feature we need. We’ll look at more use cases for boxes
in Chapter 17, too.

The `Box<T>` type is a smart pointer because it implements the `Deref` trait,
which allows `Box<T>` values to be treated like references. When a `Box<T>`
value goes out of scope, the heap data that the box is pointing to is cleaned
up as well because of the `Box<T>` type’s `Drop` trait implementation. Let’s
explore these two traits in more detail; these traits are going to be even more
important to the functionality provided by the other smart pointer types we’ll
be discussing in the rest of this chapter.

## Treating Smart Pointers like Regular References with the `Deref` Trait

Implementing `Deref` trait allows us to customize the behavior of the
*dereference operator* `*`(as opposed to the multiplication or glob operator).
By implementing `Deref` in such a way that a smart pointer can be treated like
a regular reference, we can write code that operates on references and use that
code with smart pointers too.

Let’s first take a look at how `*` works with regular references, then try and
define our own type like `Box<T>` and see why `*` doesn’t work like a
reference. We’ll explore how implementing the `Deref` trait makes it possible
for smart pointers to work in a similar way as references. Finally, we’ll look
at the *deref coercion* feature of Rust and how that lets us work with either
references or smart pointers.

### Following the Pointer to the Value with `*`

A regular reference is a type of pointer, and one way to think of a pointer is
that it’s an arrow to a value stored somewhere else. In Listing 15-6, let’s
create a reference to an `i32` value then use the dereference operator to
follow the reference to the data:

Filename: src/main.rs

```
fn main() {
    let x = 5;
    let y = &x;

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

Listing 15-6: Using the dereference operator to follow a reference to an `i32`
value

The variable `x` holds an `i32` value, `5`. We set `y` equal to a reference to
`x`. We can assert that `x` is equal to `5`. However, if we want to make an
assertion about the value in `y`, we have to use `*y` to follow the reference
to the value that the reference is pointing to (hence *de-reference*). Once we
de-reference `y`, we have access to the integer value `y` is pointing to that
we can compare with `5`.

If we try to write `assert_eq!(5, y);` instead, we’ll get this compilation
error:

```
error[E0277]: the trait bound `{integer}: std::cmp::PartialEq<&{integer}>` is
not satisfied
 --> src/main.rs:6:5
  |
6 |     assert_eq!(5, y);
  |     ^^^^^^^^^^^^^^^^^ can't compare `{integer}` with `&{integer}`
  |
  = help: the trait `std::cmp::PartialEq<&{integer}>` is not implemented for
  `{integer}`
```

Comparing a reference to a number with a number isn’t allowed because they’re
different types. We have to use `*` to follow the reference to the value it’s
pointing to.

### Using `Box<T>` Like a Reference

We can rewrite the code in Listing 15-6 to use a `Box<T>` instead of a
reference, and the de-reference operator will work the same way as shown in
Listing 15-7:

Filename: src/main.rs

```
fn main() {
    let x = 5;
    let y = Box::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

Listing 15-7: Using the dereference operator on a `Box<i32>`

The only part of Listing 15-6 that we changed was to set `y` to be an instance
of a box pointing to the value in `x` rather than a reference pointing to the
value of `x`. In the last assertion, we can use the dereference operator to
follow the box’s pointer in the same way that we did when `y` was a reference.
Let’s explore what is special about `Box<T>` that enables us to do this by
defining our own box type.

### Defining Our Own Smart Pointer

Let’s build a smart pointer similar to the `Box<T>` type that the standard
library has provided for us, in order to experience that smart pointers don’t
behave like references by default. Then we’ll learn about how to add the
ability to use the dereference operator.

`Box<T>` is ultimately defined as a tuple struct with one element, so Listing
15-8 defines a `MyBox<T>` type in the same way. We’ll also define a `new`
function to match the `new` function defined on `Box<T>`:

Filename: src/main.rs

```
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}
```

Listing 15-8: Defining a `MyBox<T>` type

We define a struct named `MyBox` and declare a generic parameter `T`, since we
want our type to be able to hold values of any type. `MyBox` is a tuple struct
with one element of type `T`. The `MyBox::new` function takes one parameter of
type `T` and returns a `MyBox` instance that holds the value passed in.

Let’s try adding the code from Listing 15-7 to the code in Listing 15-8 and
changing `main` to use the `MyBox<T>` type we’ve defined instead of `Box<T>`.
The code in Listing 15-9 won’t compile because Rust doesn’t know how to
dereference `MyBox`:

Filename: src/main.rs

```
fn main() {
    let x = 5;
    let y = MyBox::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

Listing 15-9: Attempting to use `MyBox<T>` in the same way we were able to use
references and `Box<T>`

The compilation error we get is:

```
error: type `MyBox<{integer}>` cannot be dereferenced
  --> src/main.rs:14:19
   |
14 |     assert_eq!(5, *y);
   |                   ^^
```

Our `MyBox<T>` type can’t be dereferenced because we haven’t implemented that
ability on our type. To enable dereferencing with the `*` operator, we can
implement the `Deref` trait.

### Implementing the `Deref` Trait Defines How To Treat a Type Like a Reference

As we discussed in Chapter 10, in order to implement a trait, we need to
provide implementations for the trait’s required methods. The `Deref` trait,
provided by the standard library, requires implementing one method named
`deref` that borrows `self` and returns a reference to the inner data. Listing
15-10 contains an implementation of `Deref` to add to the definition of `MyBox`:

Filename: src/main.rs

```
use std::ops::Deref;

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0
    }
}
```

Listing 15-10: Implementing `Deref` on `MyBox<T>`

The `type Target = T;` syntax defines an associated type for this trait to use.
Associated types are a slightly different way of declaring a generic parameter
that you don’t need to worry about too much for now; we’ll cover it in more
detail in Chapter 19.

We filled in the body of the `deref` method with `&self.0` so that `deref`
returns a reference to the value we want to access with the `*` operator. The
`main` function from Listing 15-9 that calls `*` on the `MyBox<T>` value now
compiles and the assertions pass!

Without the `Deref` trait, the compiler can only dereference `&` references.
The `Deref` trait’s `deref` method gives the compiler the ability to take a
value of any type that implements `Deref` and call the `deref` method in order
to get a `&` reference that it knows how to dereference.

When we typed `*y` in Listing 15-9, what Rust actually ran behind the scenes
was this code:

```
*(y.deref())
```

Rust substitutes the `*` operator with a call to the `deref` method and then a
plain dereference so that we don’t have to think about when we have to call the
`deref` method or not. This feature of Rust lets us write code that functions
identically whether we have a regular reference or a type that implements
`Deref`.

The reason the `deref` method returns a reference to a value, and why the plain
dereference outside the parentheses in `*(y.deref())` is still necessary, is
because of ownership. If the `deref` method returned the value directly instead
of a reference to the value, the value would be moved out of `self`. We don’t
want to take ownership of the inner value inside `MyBox<T>` in this case and in
most cases where we use the dereference operator.

Note that replacing `*` with a call to the `deref` method and then a call to
`*` happens once, each time we type a `*` in our code. The substitution of `*`
does not recurse infinitely. That’s how we end up with data of type `i32`,
which matches the `5` in the `assert_eq!` in Listing 15-9.

### Implicit Deref Coercions with Functions and Methods

*Deref coercion* is a convenience that Rust performs on arguments to functions
and methods. Deref coercion converts a reference to a type that implements
`Deref` into a reference to a type that `Deref` can convert the original type
into. Deref coercion happens automatically when we pass a reference to a value
of a particular type as an argument to a function or method that doesn’t match
the type of the parameter in the function or method definition, and there’s a
sequence of calls to the `deref` method that will convert the type we provided
into the type that the parameter needs.

Deref coercion was added to Rust so that programmers writing function and
method calls don’t need to add as many explicit references and dereferences
with `&` and `*`. This feature also lets us write more code that can work for
either references or smart pointers.

To illustrate deref coercion in action, let’s use the `MyBox<T>` type we
defined in Listing 15-8 as well as the implementation of `Deref` that we added
in Listing 15-10. Listing 15-11 shows the definition of a function that has a
string slice parameter:

Filename: src/main.rs

```
fn hello(name: &str) {
    println!("Hello, {}!", name);
}
```

Listing 15-11: A `hello` function that has the parameter `name` of type `&str`

We can call the `hello` function with a string slice as an argument, like
`hello("Rust");` for example. Deref coercion makes it possible for us to call
`hello` with a reference to a value of type `MyBox<String>`, as shown in
Listing 15-12:

Filename: src/main.rs

```
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&m);
}
```

Listing 15-12: Calling `hello` with a reference to a `MyBox<String>`, which
works because of deref coercion

Here we’re calling the `hello` function with the argument `&m`, which is a
reference to a `MyBox<String>` value. Because we implemented the `Deref` trait
on `MyBox<T>` in Listing 15-10, Rust can turn `&MyBox<String>` into `&String`
by calling `deref`. The standard library provides an implementation of `Deref`
on `String` that returns a string slice, which we can see in the API
documentation for `Deref`. Rust calls `deref` again to turn the `&String` into
`&str`, which matches the `hello` function’s definition.

If Rust didn’t implement deref coercion, in order to call `hello` with a value
of type `&MyBox<String>`, we’d have to write the code in Listing 15-13 instead
of the code in Listing 15-12:

Filename: src/main.rs

```
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&(*m)[..]);
}
```

Listing 15-13: The code we’d have to write if Rust didn’t have deref coercion

The `(*m)` is dereferencing the `MyBox<String>` into a `String`. Then the `&`
and `[..]` are taking a string slice of the `String` that is equal to the whole
string to match the signature of `hello`. The code without deref coercions is
harder to read, write, and understand with all of these symbols involved. Deref
coercion makes it so that Rust takes care of these conversions for us
automatically.

When the `Deref` trait is defined for the types involved, Rust will analyze the
types and use `Deref::deref` as many times as it needs in order to get a
reference to match the parameter’s type. This is resolved at compile time, so
there is no run-time penalty for taking advantage of deref coercion!

### How Deref Coercion Interacts with Mutability

Similar to how we use the `Deref` trait to override `*` on immutable
references, Rust provides a `DerefMut` trait for overriding `*` on mutable
references.

Rust does deref coercion when it finds types and trait implementations in three
cases:

* From `&T` to `&U` when `T: Deref<Target=U>`.
* From `&mut T` to `&mut U` when `T: DerefMut<Target=U>`.
* From `&mut T` to `&U` when `T: Deref<Target=U>`.

The first two cases are the same except for mutability. The first case says
that if you have a `&T`, and `T` implements `Deref` to some type `U`, you can
get a `&U` transparently. The second case states that the same deref coercion
happens for mutable references.

The last case is trickier: Rust will also coerce a mutable reference to an
immutable one. The reverse is *not* possible though: immutable references will
never coerce to mutable ones. Because of the borrowing rules, if you have a
mutable reference, that mutable reference must be the only reference to that
data (otherwise, the program wouldn’t compile). Converting one mutable
reference to one immutable reference will never break the borrowing rules.
Converting an immutable reference to a mutable reference would require that
there was only one immutable reference to that data, and the borrowing rules
don’t guarantee that. Therefore, Rust can’t make the assumption that converting
an immutable reference to a mutable reference is possible.

## The `Drop` Trait Runs Code on Cleanup

The second trait important to the smart pointer pattern is `Drop`, which lets
us customize what happens when a value is about to go out of scope. We can
provide an implementation for the `Drop` trait on any type, and the code we
specify can be used to release resources like files or network connections.
We’re introducing `Drop` in the context of smart pointers because the
functionality of the `Drop` trait is almost always used when implementing a
smart pointer. For example, `Box<T>` customizes `Drop` in order to deallocate
the space on the heap that the box points to.

In some languages, the programmer must call code to free memory or resources
every time they finish using an instance of a smart pointer. If they forget,
the system might become overloaded and crash. In Rust, we can specify that a
particular bit of code should be run whenever a value goes out of scope, and
the compiler will insert this code automatically.

This means we don’t need to be careful about placing clean up code everywhere
in a program that an instance of a particular type is finished with, but we
still won’t leak resources!

We specify the code to run when a value goes out of scope by implementing the
`Drop` trait. The `Drop` trait requires us to implement one method named `drop`
that takes a mutable reference to `self`. In order to be able to see when Rust
calls `drop`, let’s implement `drop` with `println!` statements for now.

Listing 15-14 shows a `CustomSmartPointer` struct whose only custom
functionality is that it will print out `Dropping CustomSmartPointer!` when the
instance goes out of scope. This will demonstrate when Rust runs the `drop`
function:

Filename: src/main.rs

```
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("my stuff") };
    let d = CustomSmartPointer { data: String::from("other stuff") };
    println!("CustomSmartPointers created.");
}
```

Listing 15-14: A `CustomSmartPointer` struct that implements the `Drop` trait,
where we would put our clean up code.

The `Drop` trait is included in the prelude, so we don’t need to import it. We
implement the `Drop` trait on `CustomSmartPointer`, and provide an
implementation for the `drop` method that calls `println!`. The body of the
`drop` function is where you’d put any logic that you wanted to run when an
instance of your type goes out of scope. We’re choosing to print out some text
here in order to demonstrate when Rust will call `drop`.

In `main`, we create a new instance of `CustomSmartPointer` and then print out
`CustomSmartPointer created.`. At the end of `main`, our instance of
`CustomSmartPointer` will go out of scope, and Rust will call the code we put
in the `drop` method, printing our final message. Note that we didn’t need to
call the `drop` method explicitly.

When we run this program, we’ll see the following output:

```
CustomSmartPointers created.
Dropping CustomSmartPointer with data `other stuff`!
Dropping CustomSmartPointer with data `my stuff`!
```

Rust automatically called `drop` for us when our instance went out of scope,
calling the code we specified. Variables are dropped in the reverse order of
the order in which they were created, so `d` was dropped before `c`. This is
just to give you a visual guide to how the drop method works, but usually you
would specify the cleanup code that your type needs to run rather than a print
message.

#### Dropping a Value Early with `std::mem::drop`

Rust inserts the call to `drop` automatically when a value goes out of scope,
and it’s not straightforward to disable this functionality. Disabling `drop`
isn’t usually necessary; the whole point of the `Drop` trait is that it’s taken
care of automatically for us. Occasionally you may find that you want to clean
up a value early. One example is when using smart pointers that manage locks;
you may want to force the `drop` method that releases the lock to run so that
other code in the same scope can acquire the lock. First, let’s see what
happens if we try to call the `Drop` trait’s `drop` method ourselves by
modifying the `main` function from Listing 15-14 as shown in Listing 15-15:

Filename: src/main.rs

```
fn main() {
    let c = CustomSmartPointer { data: String::from("some data") };
    println!("CustomSmartPointer created.");
    c.drop();
    println!("CustomSmartPointer dropped before the end of main.");
}
```

Listing 15-15: Attempting to call the `drop` method from the `Drop` trait
manually to clean up early

If we try to compile this, we’ll get this error:

```
error[E0040]: explicit use of destructor method
  --> src/main.rs:15:7
   |
15 |     c.drop();
   |       ^^^^ explicit destructor calls not allowed
```

This error message says we’re not allowed to explicitly call `drop`. The error
message uses the term *destructor*, which is the general programming term for a
function that cleans up an instance. A *destructor* is analogous to a
*constructor* that creates an instance. The `drop` function in Rust is one
particular destructor.

Rust doesn’t let us call `drop` explicitly because Rust would still
automatically call `drop` on the value at the end of `main`, and this would be
a *double free* error since Rust would be trying to clean up the same value
twice.

Because we can’t disable the automatic insertion of `drop` when a value goes
out of scope, and we can’t call the `drop` method explicitly, if we need to
force a value to be cleaned up early, we can use the `std::mem::drop` function.

The `std::mem::drop` function is different than the `drop` method in the `Drop`
trait. We call it by passing the value we want to force to be dropped early as
an argument. `std::mem::drop` is in the prelude, so we can modify `main` from
Listing 15-14 to call the `drop` function as shown in Listing 15-16:

Filename: src/main.rs

```
fn main() {
    let c = CustomSmartPointer { data: String::from("some data") };
    println!("CustomSmartPointer created.");
    drop(c);
    println!("CustomSmartPointer dropped before the end of main.");
}
```

Listing 15-16: Calling `std::mem::drop` to explicitly drop a value before it
goes out of scope

Running this code will print the following:

```
CustomSmartPointer created.
Dropping CustomSmartPointer with data `some data`!
CustomSmartPointer dropped before the end of main.
```

The ```Dropping CustomSmartPointer with data `some data`!``` is printed between `CustomSmartPointer
created.` and `CustomSmartPointer dropped before the end of main.`, showing
that the `drop` method code is called to drop `c` at that point.

Code specified in a `Drop` trait implementation can be used in many ways to
make cleanup convenient and safe: we could use it to create our own memory
allocator, for instance! With the `Drop` trait and Rust’s ownership system, you
don’t have to remember to clean up after yourself, Rust takes care of it
automatically.

We also don’t have to worry about accidentally cleaning up values still in use
because that would cause a compiler error: the ownership system that makes sure
references are always valid will also make sure that `drop` only gets called
once when the value is no longer being used.

Now that we’ve gone over `Box<T>` and some of the characteristics of smart
pointers, let’s talk about a few other smart pointers defined in the standard
library.

## `Rc<T>`, the Reference Counted Smart Pointer

In the majority of cases, ownership is clear: you know exactly which variable
owns a given value. However, there are cases when a single value may have
multiple owners. For example, in graph data structures, multiple edges may
point to the same node, and that node is conceptually owned by all of the edges
that point to it. A node shouldn’t be cleaned up unless it doesn’t have any
edges pointing to it.

In order to enable multiple ownership, Rust has a type called `Rc<T>`. Its name
is an abbreviation for reference counting. *Reference counting* means keeping
track of the number of references to a value in order to know if a value is
still in use or not. If there are zero references to a value, the value can be
cleaned up without any references becoming invalid.

Imagine it like a TV in a family room. When one person enters to watch TV, they
turn it on. Others can come into the room and watch the TV. When the last
person leaves the room, they turn the TV off because it’s no longer being used.
If someone turns the TV off while others are still watching it, there’d be
uproar from the remaining TV watchers!

`Rc<T>` is used when we want to allocate some data on the heap for multiple
parts of our program to read, and we can’t determine at compile time which part
will finish using the data last. If we did know which part would finish last,
we could just make that the owner of the data and the normal ownership rules
enforced at compile time would kick in.

Note that `Rc<T>` is only for use in single-threaded scenarios; Chapter 16 on
concurrency will cover how to do reference counting in multithreaded programs.

### Using `Rc<T>` to Share Data

Let’s return to our cons list example from Listing 15-5, as we defined it using
`Box<T>`. This time, we want to create two lists that both share ownership of a
third list, which conceptually will look something like Figure 15-3:

<img alt="Two lists that share ownership of a third list" src="img/trpl15-03.svg" class="center" />

Figure 15-3: Two lists, `b` and `c`, sharing ownership of a third list, `a`

We’ll create list `a` that contains 5 and then 10, then make two more lists:
`b` that starts with 3 and `c` that starts with 4. Both `b` and `c` lists will
then continue on to the first `a` list containing 5 and 10. In other words,
both lists will try to share the first list containing 5 and 10.

Trying to implement this using our definition of `List` with `Box<T>` won’t
work, as shown in Listing 15-17:

Filename: src/main.rs

```
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let a = Cons(5,
        Box::new(Cons(10,
            Box::new(Nil))));
    let b = Cons(3, Box::new(a));
    let c = Cons(4, Box::new(a));
}
```

Listing 15-17: Demonstrating we’re not allowed to have two lists using `Box<T>`
that try to share ownership of a third list

If we compile this, we get this error:

```
error[E0382]: use of moved value: `a`
  --> src/main.rs:13:30
   |
12 |     let b = Cons(3, Box::new(a));
   |                              - value moved here
13 |     let c = Cons(4, Box::new(a));
   |                              ^ value used here after move
   |
   = note: move occurs because `a` has type `List`, which does not
   implement the `Copy` trait
```

The `Cons` variants own the data they hold, so when we create the `b` list, `a`
is moved into `b` and `b` owns `a`. Then, when we try to use `a` again when
creating `c`, we’re not allowed to because `a` has been moved.

We could change the definition of `Cons` to hold references instead, but then
we’d have to specify lifetime parameters. By specifying lifetime parameters,
we’d be specifying that every element in the list will live at least as long as
the list itself. The borrow checker wouldn’t let us compile `let a = Cons(10,
&Nil);` for example, since the temporary `Nil` value would be dropped before
`a` could take a reference to it.

Instead, we’ll change our definition of `List` to use `Rc<T>` in place of
`Box<T>` as shown here in Listing 15-18. Each `Cons` variant now holds a value
and an `Rc` pointing to a `List`. When we create `b`, instead of taking
ownership of `a`, we clone the `Rc` that `a` is holding, which increases the
number of references from 1 to 2 and lets `a` and `b` share ownership of the
data in that `Rc`. We also clone `a` when creating `c`, which increases the
number of references from 2 to 3. Every time we call `Rc::clone`, the reference
count to the data within the `Rc` is increased, and the data won’t be cleaned
up unless there are zero references to it:

Filename: src/main.rs

```
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
}
```

Listing 15-18: A definition of `List` that uses `Rc<T>`

We need to add a `use` statement to bring `Rc` into scope because it’s not in
the prelude. In `main`, we create the list holding 5 and 10 and store it in a
new `Rc` in `a`. Then when we create `b` and `c`, we call the `Rc::clone`
function and pass a reference to the `Rc` in `a` as an argument.

We could have called `a.clone()` rather than `Rc::clone(&a)`, but Rust
convention is to use `Rc::clone` in this case. The implementation of `Rc::clone`
doesn’t make a deep copy of all the data like most types’ implementations of
`clone` do. `Rc::clone` only increments the reference count, which doesn’t take
very much time. Deep copies of data can take a lot of time, so by using
`Rc::clone` for reference counting, we can visually distinguish between the
deep copy kinds of clones that might have a large impact on runtime performance
and memory usage and the types of clones that increase the reference count that
have a comparatively small impact on runtime performance and don’t allocate new
memory.

### Cloning an `Rc<T>` Increases the Reference Count

Let’s change our working example from Listing 15-18 so that we can see the
reference counts changing as we create and drop references to the `Rc` in `a`.

In Listing 15-19, we’ll change `main` so that it has an inner scope around list
`c`, so that we can see how the reference count changes when `c` goes out of
scope. At each point in the program where the reference count changes, we’ll
print out the reference count, which we can get by calling the
`Rc::strong_count` function. We’ll talk about why this function is named
`strong_count` rather than `count` in the section later in this chapter about
preventing reference cycles.

Filename: src/main.rs

```
fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    println!("count after creating a = {}", Rc::strong_count(&a));
    let b = Cons(3, Rc::clone(&a));
    println!("count after creating b = {}", Rc::strong_count(&a));
    {
        let c = Cons(4, Rc::clone(&a));
        println!("count after creating c = {}", Rc::strong_count(&a));
    }
    println!("count after c goes out of scope = {}", Rc::strong_count(&a));
}
```

Listing 15-19: Printing out the reference count

This will print out:

```
count after creating a = 1
count after creating b = 2
count after creating c = 3
count after c goes out of scope = 2
```

We’re able to see that the `Rc` in `a` has an initial reference count of one,
then each time we call `clone`, the count goes up by one. When `c` goes out of
scope, the count goes down by one. We don’t have to call a function to decrease
the reference count like we have to call `Rc::clone` to increase the reference
count; the implementation of the `Drop` trait decreases the reference count
automatically when an `Rc` value goes out of scope.

What we can’t see from this example is that when `b` and then `a` go out of
scope at the end of `main`, the count is then 0, and the `Rc` is cleaned up
completely at that point. Using `Rc` allows a single value to have multiple
owners, and the count will ensure that the value remains valid as long as any
of the owners still exist.

`Rc<T>` allows us to share data between multiple parts of our program for
reading only, via immutable references. If `Rc<T>` allowed us to have multiple
mutable references too, we’d be able to violate one of the borrowing rules
that we discussed in Chapter 4: multiple mutable borrows to the same place can
cause data races and inconsistencies. But being able to mutate data is very
useful! In the next section, we’ll discuss the interior mutability pattern and
the `RefCell<T>` type that we can use in conjunction with an `Rc<T>` to work
with this restriction on immutability.

## `RefCell<T>` and the Interior Mutability Pattern

*Interior mutability* is a design pattern in Rust for allowing you to mutate
data even when there are immutable references to that data, normally disallowed
by the borrowing rules. To do so, the pattern uses `unsafe` code inside a data
structure to bend Rust’s usual rules around mutation and borrowing. We haven’t
yet covered unsafe code; we will in Chapter 19. We can choose to use types that
make use of the interior mutability pattern when we can ensure that the
borrowing rules will be followed at runtime, even though the compiler can’t
ensure that. The `unsafe` code involved is then wrapped in a safe API, and the
outer type is still immutable.

Let’s explore this by looking at the `RefCell<T>` type that follows the
interior mutability pattern.

### Enforcing Borrowing Rules at Runtime with `RefCell<T>`

Unlike `Rc<T>`, the `RefCell<T>` type represents single ownership over the data
it holds. So, what makes `RefCell<T>` different than a type like `Box<T>`?
Let’s recall the borrowing rules we learned in Chapter 4:

1. At any given time, you can have *either* but not both of:
  * One mutable reference.
  * Any number of immutable references.
2. References must always be valid.

With references and `Box<T>`, the borrowing rules’ invariants are enforced at
compile time. With `RefCell<T>`, these invariants are enforced *at runtime*.
With references, if you break these rules, you’ll get a compiler error. With
`RefCell<T>`, if you break these rules, you’ll get a `panic!`.

The advantages to checking the borrowing rules at compile time are that errors
will be caught sooner in the development process and there is no impact on
runtime performance since all the analysis is completed beforehand. For those
reasons, checking the borrowing rules at compile time is the best choice for
the majority of cases, which is why this is Rust’s default.

The advantage to checking the borrowing rules at runtime instead is that
certain memory safe scenarios are then allowed, whereas they are disallowed by
the compile time checks. Static analysis, like the Rust compiler, is inherently
conservative. Some properties of code are impossible to detect by analyzing the
code: the most famous example is the Halting Problem, which is out of scope of
this book but an interesting topic to research if you’re interested.

Because some analysis is impossible, if the Rust compiler can’t be sure the
code complies with the ownership rules, it may reject a correct program; in
this way, it is conservative. If Rust were to accept an incorrect program,
users would not be able to trust in the guarantees Rust makes. However, if Rust
rejects a correct program, the programmer will be inconvenienced, but nothing
catastrophic can occur. `RefCell<T>` is useful when you yourself are sure that
your code follows the borrowing rules, but the compiler is not able to
understand and guarantee that.

Similarly to `Rc<T>`, `RefCell<T>` is only for use in single-threaded scenarios
and will give you a compile time error if you try in a multithreaded context.
We’ll talk about how to get the functionality of `RefCell<T>` in a
multithreaded program in Chapter 16.

To recap the reasons to choose `Box<T>`, `Rc<T>`, or `RefCell<T>`:

- `Rc<T>` enables multiple owners of the same data; `Box<T>` and `RefCell<T>`
  have single owners.
- `Box<T>` allows immutable or mutable borrows checked at compile time; `Rc<T>`
  only allows immutable borrows checked at compile time; `RefCell<T>` allows
  immutable or mutable borrows checked at runtime.
- Because `RefCell<T>` allows mutable borrows checked at runtime, we can mutate
  the value inside the `RefCell<T>` even when the `RefCell<T>` is itself
  immutable.

The last reason is the *interior mutability* pattern. Let’s look at a case when
interior mutability is useful and discuss how this is possible.

### Interior Mutability: A Mutable Borrow to an Immutable Value

A consequence of the borrowing rules is that when we have an immutable value,
we can’t borrow it mutably. For example, this code won’t compile:

```
fn main() {
    let x = 5;
    let y = &mut x;
}
```

If we try to compile this, we’ll get this error:

```
error[E0596]: cannot borrow immutable local variable `x` as mutable
 --> src/main.rs:3:18
  |
2 |     let x = 5;
  |         - consider changing this to `mut x`
3 |     let y = &mut x;
  |                  ^ cannot borrow mutably
```

However, there are situations where it would be useful for a value to be able
to mutate itself in its methods, but to other code, the value would appear to
be immutable. Code outside the value’s methods would not be able to mutate the
value. `RefCell<T>` is one way to get the ability to have interior mutability.
`RefCell<T>` isn’t getting around the borrowing rules completely, but the
borrow checker in the compiler allows this interior mutability and the
borrowing rules are checked at runtime instead. If we violate the rules, we’ll
get a `panic!` instead of a compiler error.

Let’s work through a practical example where we can use `RefCell<T>` to make it
possible to mutate an immutable value and see why that’s useful.

#### A Use Case for Interior Mutability: Mock Objects

A *test double* is the general programming concept for a type that stands in
the place of another type during testing. *Mock objects* are specific types of
test doubles that record what happens during a test so that we can assert that
the correct actions took place.

While Rust doesn’t have objects in the exact same sense that other languages
have objects, and Rust doesn’t have mock object functionality built into the
standard library like some other languages do, we can definitely create a
struct that will serve the same purposes as a mock object.

Here’s the scenario we’d like to test: we’re creating a library that tracks a
value against a maximum value, and sends messages based on how close to the
maximum value the current value is. This could be used for keeping track of a
user’s quota for the number of API calls they’re allowed to make, for example.

Our library is only going to provide the functionality of tracking how close to
the maximum a value is, and what the messages should be at what times.
Applications that use our library will be expected to provide the actual
mechanism for sending the messages: the application could choose to put a
message in the application, send an email, send a text message, or something
else. Our library doesn’t need to know about that detail; all it needs is
something that implements a trait we’ll provide called `Messenger`. Listing
15-20 shows our library code:

Filename: src/lib.rs

```
pub trait Messenger {
    fn send(&self, msg: &str);
}

pub struct LimitTracker<'a, T: 'a + Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T>
    where T: Messenger {
    pub fn new(messenger: &T, max: usize) -> LimitTracker<T> {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }

    pub fn set_value(&mut self, value: usize) {
        self.value = value;

        let percentage_of_max = self.value as f64 / self.max as f64;

        if percentage_of_max >= 0.75 && percentage_of_max < 0.9 {
            self.messenger.send("Warning: You've used up over 75% of your quota!");
        } else if percentage_of_max >= 0.9 && percentage_of_max < 1.0 {
            self.messenger.send("Urgent warning: You've used up over 90% of your quota!");
        } else if percentage_of_max >= 1.0 {
            self.messenger.send("Error: You are over your quota!");
        }
    }
}
```

Listing 15-20: A library to keep track of how close to a maximum value a value
is, and warn when the value is at certain levels

One important part of this code is that the `Messenger` trait has one method,
`send`, that takes an immutable reference to `self` and text of the message.
This is the interface our mock object will need to have. The other important
part is that we want to test the behavior of the `set_value` method on the
`LimitTracker`. We can change what we pass in for the `value` parameter, but
`set_value` doesn’t return anything for us to make assertions on. What we want
to be able to say is that if we create a `LimitTracker` with something that
implements the `Messenger` trait and a particular value for `max`, when we pass
different numbers for `value`, the messenger gets told to send the appropriate
messages.

What we need is a mock object that, instead of actually sending an email or
text message when we call `send`, will only keep track of the messages it’s
told to send. We can create a new instance of the mock object, create a
`LimitTracker` that uses the mock object, call the `set_value` method on
`LimitTracker`, then check that the mock object has the messages we expect.
Listing 15-21 shows an attempt of implementing a mock object to do just that,
but that the borrow checker won’t allow:

Filename: src/lib.rs

```
#[cfg(test)]
mod tests {
    use super::*;

    struct MockMessenger {
        sent_messages: Vec<String>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger { sent_messages: vec![] }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);

        limit_tracker.set_value(80);

        assert_eq!(mock_messenger.sent_messages.len(), 1);
    }
}
```

Listing 15-21: An attempt to implement a `MockMessenger` that isn’t allowed by
the borrow checker

This test code defines a `MockMessenger` struct that has a `sent_messages`
field with a `Vec` of `String` values to keep track of the messages it’s told
to send. We also defined an associated function `new` to make it convenient to
create new `MockMessenger` values that start with an empty list of messages. We
then implement the `Messenger` trait for `MockMessenger` so that we can give a
`MockMessenger` to a `LimitTracker`. In the definition of the `send` method, we
take the message passed in as a parameter and store it in the `MockMessenger`
list of `sent_messages`.

In the test, we’re testing what happens when the `LimitTracker` is told to set
`value` to something that’s over 75% of the `max` value. First, we create a new
`MockMessenger`, which will start with an empty list of messages. Then we
create a new `LimitTracker` and give it a reference to the new `MockMessenger`
and a `max` value of 100. We call the `set_value` method on the `LimitTracker`
with a value of 80, which is more than 75% of 100. Then we assert that the list
of messages that the `MockMessenger` is keeping track of should now have one
message in it.

There’s one problem with this test, however:

```
error[E0596]: cannot borrow immutable field `self.sent_messages` as mutable
  --> src/lib.rs:46:13
   |
45 |         fn send(&self, message: &str) {
   |                 ----- use `&mut self` here to make mutable
46 |             self.sent_messages.push(String::from(message));
   |             ^^^^^^^^^^^^^^^^^^ cannot mutably borrow immutable field
```

We can’t modify the `MockMessenger` to keep track of the messages because the
`send` method takes an immutable reference to `self`. We also can’t take the
suggestion from the error text to use `&mut self` instead because then the
signature of `send` wouldn’t match the signature in the `Messenger` trait
definition (feel free to try and see what error message you get).

This is where interior mutability can help! We’re going to store the
`sent_messages` within a `RefCell`, and then the `send` message will be able to
modify `sent_messages` to store the messages we’ve seen. Listing 15-22 shows
what that looks like:

Filename: src/lib.rs

```
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger { sent_messages: RefCell::new(vec![]) }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.borrow_mut().push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        // --snip--

        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```

Listing 15-22: Using `RefCell<T>` to be able to mutate an inner value while the
outer value is considered immutable

The `sent_messages` field is now of type `RefCell<Vec<String>>` instead of
`Vec<String>`. In the `new` function, we create a new `RefCell` instance around
the empty vector.

For the implementation of the `send` method, the first parameter is still an
immutable borrow of `self`, which matches the trait definition. We call
`borrow_mut` on the `RefCell` in `self.sent_messages` to get a mutable
reference to the value inside the `RefCell`, which is the vector. Then we can
call `push` on the mutable reference to the vector in order to keep track of
the messages seen during the test.

The last change we have to make is in the assertion: in order to see how many
items are in the inner vector, we call `borrow` on the `RefCell` to get an
immutable reference to the vector.

Now that we’ve seen how to use `RefCell<T>`, let’s dig into how it works!

#### `RefCell<T>` Keeps Track of Borrows at Runtime

When creating immutable and mutable references we use the `&` and `&mut`
syntax, respectively. With `RefCell<T>`, we use the `borrow` and `borrow_mut`
methods, which are part of the safe API that belongs to `RefCell<T>`. The
`borrow` method returns the smart pointer type `Ref`, and `borrow_mut` returns
the smart pointer type `RefMut`. Both types implement `Deref` so we can treat
them like regular references.

The `RefCell<T>` keeps track of how many `Ref` and `RefMut` smart pointers are
currently active. Every time we call `borrow`, the `RefCell<T>` increases its
count of how many immutable borrows are active. When a `Ref` value goes out of
scope, the count of immutable borrows goes down by one. Just like the compile
time borrowing rules, `RefCell<T>` lets us have many immutable borrows or one
mutable borrow at any point in time.

If we try to violate these rules, rather than getting a compiler error like we
would with references, the implementation of `RefCell<T>` will `panic!` at
runtime. Listing 15-23 shows a modification to the implementation of `send`
from Listing 15-22 where we’re deliberately trying to create two mutable
borrows active for the same scope in order to illustrate that `RefCell<T>`
prevents us from doing this at runtime:

Filename: src/lib.rs

```
impl Messenger for MockMessenger {
    fn send(&self, message: &str) {
        let mut one_borrow = self.sent_messages.borrow_mut();
        let mut two_borrow = self.sent_messages.borrow_mut();

        one_borrow.push(String::from(message));
        two_borrow.push(String::from(message));
    }
}
```

Listing 15-23: Creating two mutable references in the same scope to see that
`RefCell<T>` will panic

We create a variable `one_borrow` for the `RefMut` smart pointer returned from
`borrow_mut`. Then we create another mutable borrow in the same way in the
variable `two_borrow`. This makes two mutable references in the same scope,
which isn’t allowed. If we run the tests for our library, this code will
compile without any errors, but the test will fail:

```
---- tests::it_sends_an_over_75_percent_warning_message stdout ----
	thread 'tests::it_sends_an_over_75_percent_warning_message' panicked at
    'already borrowed: BorrowMutError', src/libcore/result.rs:906:4
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

We can see that the code panicked with the message `already borrowed:
BorrowMutError`. This is how `RefCell<T>` handles violations of the borrowing
rules at runtime.

Catching borrowing errors at runtime rather than compile time means that we’d
find out that we made a mistake in our code later in the development process--
and possibly not even until our code was deployed to production. There’s also a
small runtime performance penalty our code will incur as a result of keeping
track of the borrows at runtime rather than compile time. However, using
`RefCell` made it possible for us to write a mock object that can modify itself
to keep track of the messages it has seen while we’re using it in a context
where only immutable values are allowed. We can choose to use `RefCell<T>`
despite its tradeoffs to get more abilities than regular references give us.

### Having Multiple Owners of Mutable Data by Combining `Rc<T>` and `RefCell<T>`

A common way to use `RefCell<T>` is in combination with `Rc<T>`. Recall that
`Rc<T>` lets us have multiple owners of some data, but it only gives us
immutable access to that data. If we have an `Rc<T>` that holds a `RefCell<T>`,
then we can get a value that can have multiple owners *and* that we can mutate!

For example, recall the cons list example from Listing 15-18 where we used
`Rc<T>` to let us have multiple lists share ownership of another list. Because
`Rc<T>` only holds immutable values, we aren’t able to change any of the values
in the list once we’ve created them. Let’s add in `RefCell<T>` to get the
ability to change the values in the lists. Listing 15-24 shows that by using a
`RefCell<T>` in the `Cons` definition, we’re allowed to modify the value stored
in all the lists:

Filename: src/main.rs

```
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(10)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```

Listing 15-24: Using `Rc<RefCell<i32>>` to create a `List` that we can mutate

We create a value that’s an instance of `Rc<RefCell<i32>` and store it in a
variable named `value` so we can access it directly later. Then we create a
`List` in `a` with a `Cons` variant that holds `value`. We need to clone
`value` so that both `a` and `value` have ownership of the inner `5` value,
rather than transferring ownership from `value` to `a` or having `a` borrow
from `value`.

We wrap the list `a` in an `Rc<T>` so that when we create lists `b` and
`c`, they can both refer to `a`, the same as we did in Listing 15-18.

Once we have the lists in `a`, `b`, and `c` created, we add 10 to the value in
`value`. We do this by calling `borrow_mut` on `value`, which uses the
automatic dereferencing feature we discussed in Chapter 5 (“Where’s the `->`
Operator?”) to dereference the `Rc<T>` to the inner `RefCell<T>` value. The
`borrow_mut` method returns a `RefMut<T>` smart pointer, and we use the
dereference operator on it and change the inner value.

When we print out `a`, `b`, and `c`, we can see that they all have the modified
value of 15 rather than 5:

```
a after = Cons(RefCell { value: 15 }, Nil)
b after = Cons(RefCell { value: 6 }, Cons(RefCell { value: 15 }, Nil))
c after = Cons(RefCell { value: 10 }, Cons(RefCell { value: 15 }, Nil))
```

This is pretty neat! By using `RefCell<T>`, we have an outwardly immutable
`List`, but we can use the methods on `RefCell<T>` that provide access to its
interior mutability so we can modify our data when we need to. The runtime
checks of the borrowing rules protect us from data races, and it’s sometimes
worth trading a bit of speed for this flexibility in our data structures.

The standard library has other types that provide interior mutability, too,
like `Cell<T>`, which is similar except that instead of giving references to
the inner value, the value is copied in and out of the `Cell<T>`. There’s also
`Mutex<T>`, which offers interior mutability that’s safe to use across threads,
and we’ll be discussing its use in the next chapter on concurrency. Check out
the standard library docs for more details on the differences between these
types.

## Reference Cycles Can Leak Memory

Rust’s memory safety guarantees make it *difficult* to accidentally create
memory that’s never cleaned up, known as a *memory leak*, but not impossible.
Entirely preventing memory leaks is not one of Rust’s guarantees in the same
way that disallowing data races at compile time is, meaning memory leaks are
memory safe in Rust. We can see this with `Rc<T>` and `RefCell<T>`: it’s
possible to create references where items refer to each other in a cycle. This
creates memory leaks because the reference count of each item in the cycle will
never reach 0, and the values will never be dropped.

### Creating a Reference Cycle

Let’s take a look at how a reference cycle might happen and how to prevent it,
starting with the definition of the `List` enum and a `tail` method in Listing
15-25:

Filename: src/main.rs

```
use std::rc::Rc;
use std::cell::RefCell;
use List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match *self {
            Cons(_, ref item) => Some(item),
            Nil => None,
        }
    }
}
```

Listing 15-25: A cons list definition that holds a `RefCell` so that we can
modify what a `Cons` variant is referring to

We’re using another variation of the `List` definition from Listing 15-5. The
second element in the `Cons` variant is now `RefCell<Rc<List>>`, meaning that
instead of having the ability to modify the `i32` value like we did in Listing
15-24, we want to be able to modify which `List` a `Cons` variant is pointing
to. We’ve also added a `tail` method to make it convenient for us to access the
second item, if we have a `Cons` variant.

In listing 15-26, we’re adding a `main` function that uses the definitions from
Listing 15-25. This code creates a list in `a`, a list in `b` that points to
the list in `a`, and then modifies the list in `a` to point to `b`, which
creates a reference cycle. There are `println!` statements along the way to
show what the reference counts are at various points in this process.

Filename: src/main.rs

```
fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));

    println!("a initial rc count = {}", Rc::strong_count(&a));
    println!("a next item = {:?}", a.tail());

    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));

    println!("a rc count after b creation = {}", Rc::strong_count(&a));
    println!("b initial rc count = {}", Rc::strong_count(&b));
    println!("b next item = {:?}", b.tail());

    if let Some(ref link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }

    println!("b rc count after changing a = {}", Rc::strong_count(&b));
    println!("a rc count after changing a = {}", Rc::strong_count(&a));

    // Uncomment the next line to see that we have a cycle; it will
    // overflow the stack
    // println!("a next item = {:?}", a.tail());
}
```

Listing 15-26: Creating a reference cycle of two `List` values pointing to each
other

We create an `Rc` instance holding a `List` value in the variable `a` with an
initial list of `5, Nil`. We then create an `Rc` instance holding another
`List` value in the variable `b` that contains the value 10, then points to the
list in `a`.

Finally, we modify `a` so that it points to `b` instead of `Nil`, which creates
a cycle. We do that by using the `tail` method to get a reference to the
`RefCell` in `a`, which we put in the variable `link`. Then we use the
`borrow_mut` method on the `RefCell` to change the value inside from an `Rc`
that holds a `Nil` value to the `Rc` in `b`.

If we run this code, keeping the last `println!` commented out for the moment,
we’ll get this output:

```
a initial rc count = 1
a next item = Some(RefCell { value: Nil })
a rc count after b creation = 2
b initial rc count = 1
b next item = Some(RefCell { value: Cons(5, RefCell { value: Nil }) })
b rc count after changing a = 2
a rc count after changing a = 2
```

We can see that the reference count of the `Rc` instances in both `a` and `b`
are 2 after we change the list in `a` to point to `b`. At the end of `main`,
Rust will try and drop `b` first, which will decrease the count in each of the
`Rc` instances in `a` and `b` by one.

However, because `a` is still referencing the `Rc` that was in `b`, that `Rc`
has a count of 1 rather than 0, so the memory the `Rc` has on the heap won’t be
dropped. The memory will just sit there with a count of one, forever.

To visualize this, we’ve created a reference cycle that looks like Figure 15-4:

<img alt="Reference cycle of lists" src="img/trpl15-04.svg" class="center" />

Figure 15-4: A reference cycle of lists `a` and `b` pointing to each other

If you uncomment the last `println!` and run the program, Rust will try and
print this cycle out with `a` pointing to `b` pointing to `a` and so forth
until it overflows the stack.

In this specific case, right after we create the reference cycle, the program
ends. The consequences of this cycle aren’t so dire. If a more complex program
allocates lots of memory in a cycle and holds onto it for a long time, the
program would be using more memory than it needs, and might overwhelm the
system and cause it to run out of available memory.

Creating reference cycles is not easily done, but it’s not impossible either.
If you have `RefCell<T>` values that contain `Rc<T>` values or similar nested
combinations of types with interior mutability and reference counting, be aware
that you have to ensure you don’t create cycles yourself; you can’t rely on
Rust to catch them. Creating a reference cycle would be a logic bug in your
program that you should use automated tests, code reviews, and other software
development practices to minimize.

Another solution is reorganizing your data structures so that some references
express ownership and some references don’t. In this way, we can have cycles
made up of some ownership relationships and some non-ownership relationships,
and only the ownership relationships affect whether a value may be dropped or
not. In Listing 15-25, we always want `Cons` variants to own their list, so
reorganizing the data structure isn’t possible. Let’s look at an example using
graphs made up of parent nodes and child nodes to see when non-ownership
relationships are an appropriate way to prevent reference cycles.

### Preventing Reference Cycles: Turn an `Rc<T>` into a `Weak<T>`

So far, we’ve shown how calling `Rc::clone` increases the `strong_count` of an
`Rc` instance, and that an `Rc` instance is only cleaned up if its
`strong_count` is 0. We can also create a *weak reference* to the value within
an `Rc` instance by calling `Rc::downgrade` and passing a reference to the
`Rc`. When we call `Rc::downgrade`, we get a smart pointer of type `Weak<T>`.
Instead of increasing the `strong_count` in the `Rc` instance by one, calling
`Rc::downgrade` increases the `weak_count` by one. The `Rc` type uses
`weak_count` to keep track of how many `Weak<T>` references exist, similarly to
`strong_count`. The difference is the `weak_count` does not need to be 0 in
order for the `Rc` instance to be cleaned up.

Strong references are how we can share ownership of an `Rc` instance. Weak
references don’t express an ownership relationship. They won’t cause a
reference cycle since any cycle involving some weak references will be broken
once the strong reference count of values involved is 0.

Because the value that `Weak<T>` references might have been dropped, in order
to do anything with the value that a `Weak<T>` is pointing to, we have to check
to make sure the value is still around. We do this by calling the `upgrade`
method on a `Weak<T>` instance, which will return an `Option<Rc<T>>`. We’ll get
a result of `Some` if the `Rc` value has not been dropped yet, and `None` if
the `Rc` value has been dropped. Because `upgrade` returns an `Option`, we can
be sure that Rust will handle both the `Some` case and the `None` case, and
there won’t be an invalid pointer.

As an example, rather than using a list whose items know only about the next
item, we’ll create a tree whose items know about their children items *and*
their parent items.

#### Creating a Tree Data Structure: a `Node` with Child Nodes

To start building this tree, we’ll create a struct named `Node` that holds its
own `i32` value as well as references to its children `Node` values:

Filename: src/main.rs

```
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    children: RefCell<Vec<Rc<Node>>>,
}
```

We want a `Node` to own its children, and we want to be able to share that
ownership with variables so we can access each `Node` in the tree directly. To
do this, we define the `Vec` items to be values of type `Rc<Node>`. We also
want to be able to modify which nodes are children of another node, so we have
a `RefCell` in `children` around the `Vec`.

Next, let’s use our struct definition and create one `Node` instance named
`leaf` with the value 3 and no children, and another instance named `branch`
with the value 5 and `leaf` as one of its children, as shown in Listing 15-27:

Filename: src/main.rs

```
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        children: RefCell::new(vec![]),
    });

    let branch = Rc::new(Node {
        value: 5,
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });
}
```

Listing 15-27: Creating a `leaf` node with no children and a `branch` node with
`leaf` as one of its children

We clone the `Rc` in `leaf` and store that in `branch`, meaning the `Node` in
`leaf` now has two owners: `leaf` and `branch`. We can get from `branch` to
`leaf` through `branch.children`, but there’s no way to get from `leaf` to
`branch`. `leaf` has no reference to `branch` and doesn’t know they are
related. We’d like `leaf` to know that `branch` is its parent.

#### Adding a Reference from a Child to its Parent

To make the child node aware of its parent, we need to add a `parent` field to
our `Node` struct definition. The trouble is in deciding what the type of
`parent` should be. We know it can’t contain an `Rc<T>` because that would
create a reference cycle, with `leaf.parent` pointing to `branch` and
`branch.children` pointing to `leaf`, which would cause their `strong_count`
values to never be zero.

Thinking about the relationships another way, a parent node should own its
children: if a parent node is dropped, its child nodes should be dropped as
well. However, a child should not own its parent: if we drop a child node, the
parent should still exist. This is a case for weak references!

So instead of `Rc`, we’ll make the type of `parent` use `Weak<T>`, specifically
a `RefCell<Weak<Node>>`. Now our `Node` struct definition looks like this:

Filename: src/main.rs

```
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}
```

This way, a node will be able to refer to its parent node, but does not own its
parent. In Listing 15-28, let’s update `main` to use this new definition so
that the `leaf` node will have a way to refer to its parent, `branch`:

Filename: src/main.rs

```
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());

    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });

    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
}
```

Listing 15-28: A `leaf` node with a `Weak` reference to its parent node,
`branch`

Creating the `leaf` node looks similar to how creating the `leaf` node looked
in Listing 15-27, with the exception of the `parent` field: `leaf` starts out
without a parent, so we create a new, empty `Weak` reference instance.

At this point, when we try to get a reference to the parent of `leaf` by using
the `upgrade` method, we get a `None` value. We see this in the output from the
first `println!`:

```
leaf parent = None
```

When we create the `branch` node, it will also have a new `Weak` reference,
since `branch` does not have a parent node. We still have `leaf` as one of the
children of `branch`. Once we have the `Node` instance in `branch`, we can
modify `leaf` to give it a `Weak` reference to its parent. We use the
`borrow_mut` method on the `RefCell` in the `parent` field of `leaf`, then we
use the `Rc::downgrade` function to create a `Weak` reference to `branch` from
the `Rc` in `branch.`

When we print out the parent of `leaf` again, this time we’ll get a `Some`
variant holding `branch`: `leaf` can now access its parent! When we print out
`leaf`, we also avoid the cycle that eventually ended in a stack overflow like
we had in Listing 15-26: the `Weak` references are printed as `(Weak)`:

```
leaf parent = Some(Node { value: 5, parent: RefCell { value: (Weak) },
children: RefCell { value: [Node { value: 3, parent: RefCell { value: (Weak) },
children: RefCell { value: [] } }] } })
```

The lack of infinite output indicates that this code didn’t create a reference
cycle. We can also tell this by looking at the values we get from calling
`Rc::strong_count` and `Rc::weak_count`.

#### Visualizing Changes to `strong_count` and `weak_count`

Let’s look at how the `strong_count` and `weak_count` values of the `Rc`
instances change by creating a new inner scope and moving the creation of
`branch` into that scope. This will let us see what happens when `branch` is
created and then dropped when it goes out of scope. The modifications are shown
in Listing 15-29:

Filename: src/main.rs

```
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf),
    );

    {
        let branch = Rc::new(Node {
            value: 5,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(vec![Rc::clone(&leaf)]),
        });
        *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

        println!(
            "branch strong = {}, weak = {}",
            Rc::strong_count(&branch),
            Rc::weak_count(&branch),
        );

        println!(
            "leaf strong = {}, weak = {}",
            Rc::strong_count(&leaf),
            Rc::weak_count(&leaf),
        );
    }

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf),
    );
}
```

Listing 15-29: Creating `branch` in an inner scope and examining strong and
weak reference counts

Once `leaf` is created, its `Rc` has a strong count of 1 and a weak count of 0.
In the inner scope we create `branch` and associate it with `leaf`, at which
point the `Rc` in `branch` will have a strong count of 1 and a weak count of 1
(for `leaf.parent` pointing to `branch` with a `Weak<T>`). Here `leaf` will
have a strong count of 2, because `branch` now has a clone of the `Rc` of
`leaf` stored in `branch.children`, but will still have a weak count of 0.

When the inner scope ends, `branch` goes out of scope and the strong count of
the `Rc` decreases to 0, so its `Node` gets dropped. The weak count of 1 from
`leaf.parent` has no bearing on whether `Node` is dropped or not, so we don’t
get any memory leaks!

If we try to access the parent of `leaf` after the end of the scope, we’ll get
`None` again. At the end of the program, the `Rc` in `leaf` has a strong count
of 1 and a weak count of 0, because the variable `leaf` is now the only
reference to the `Rc` again.

All of the logic that manages the counts and value dropping is built in to
`Rc` and `Weak` and their implementations of the `Drop` trait. By specifying
that the relationship from a child to its parent should be a `Weak<T>`
reference in the definition of `Node`, we’re able to have parent nodes point to
child nodes and vice versa without creating a reference cycle and memory leaks.

## Summary

This chapter covered how you can use smart pointers to make different
guarantees and tradeoffs than those Rust makes by default with regular
references. `Box<T>` has a known size and points to data allocated on the heap.
`Rc<T>` keeps track of the number of references to data on the heap so that
data can have multiple owners. `RefCell<T>` with its interior mutability gives
us a type that can be used when we need an immutable type but need the ability
to change an inner value of that type, and enforces the borrowing rules at
runtime instead of at compile time.

We also discussed the `Deref` and `Drop` traits that enable a lot of the
functionality of smart pointers. We explored reference cycles that can cause
memory leaks, and how to prevent them using `Weak<T>`.

If this chapter has piqued your interest and you want to implement your own
smart pointers, check out “The Nomicon” at
*https://doc.rust-lang.org/stable/nomicon/* for even more useful information.

Next, let’s talk about concurrency in Rust. We’ll even learn about a few new
smart pointers.
