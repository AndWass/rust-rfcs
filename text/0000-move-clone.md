- Feature Name: `move_clone`
- Start Date: 2024-08-23
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Provide a feature to simplify cloning values into closures or async blocks, while still keeping such cloning visible and explicit.

# Motivation
[motivation]: #motivation

A very common source of friction in asynchronous or multithreaded Rust
programming is having to clone various objects into
an async block or task. This is particularly common when spawning a closure as
a thread, or spawning an async block as a task. Common patterns for doing so
include:

```rust
// Use new names throughout the block
let new_x = x.clone();
let new_y = y.clone();
spawn(async move {
    func1(new_x).await;
    func2(new_y).await;
});

// Introduce a scope to perform the clones in
{
    let x = x.clone();
    let y = y.clone();
    spawn(async move {
        func1(x).await;
        func2(y).await;
    });
}

// Introduce a scope to perform the clones in, inside the call
spawn({
    let x = x.clone();
    let y = y.clone();
    async move {
        func1(x).await;
        func2(y).await;
    }
});
```

All of these patterns introduce noise every time the program wants to spawn a
thread or task, or otherwise clone an object into a closure or async block.
Feedback on Rust regularly brings up this friction, seeking a simpler solution.

This RFC proposes a solution that *decreases* the syntactic noise when cloning values
into a closure or async block. It does not strive to *minimize* the syntax, but to
strike a balance between those that want less syntax and those that value explict-ness.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When creating a closure or async block that captures objects that support cloning, such as `Rc`,
`Arc`, `String` among others, you can put the contextual keyword `clone` after `move`. This will
move clones of the referred objects into the closure or async block instead.

```rust
let obj: ClonableObject = new_object();
let map: HashMap<i32, String> = new_mapping();
std::thread::spawn(move clone || func(map, obj));
task::spawn(async move clone { op(map, obj).await });
another_func(obj, map);
```

Note that `clone` must appear after `move` to be completely unambigious:

```rust
let clone = false;
let x = clone || true; // Is x a closure or bool?
```

`clone` also supports a list of identifiers, called a capture list. This will limit which objects to be cloned,
all other objects to be moved.

```rust
let obj: ClonableObject = new_object();
let string: String = new_string();
let map: HashMap<i32, String> = new_mapping();
std::thread::spawn(move clone(obj, string) || func(map, obj, string)); // only obj and string are cloned and moved, map is moved into the closure.
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`clone` can appear as a contextual keyword following `move` when creating closures or async blocks. `clone`
can have an optional list of comma-separated identifiers enclosed in parenthesis, listing the captures that
will be cloned. All objects not cloned will be moved instead.

## Equivalences

All equivalnces are shown for closures, but are equally valid when creating async blocks as well.

### Implicit captures

```rust
move clone || op(a, b, c);
```

is equivalent to

```rust
{
    let a = a.clone();
    let b = b.clone();
    let c = c.clone();
    move || op(a, b, c)
};
```

### Explicit captures

```rust
move clone(a, b) || op(a, b, c);
```

is equivalent to

```rust
{
    let a = a.clone();
    let b = b.clone();
    move || op(a, b, c)
};
```

# Drawbacks
[drawbacks]: #drawbacks

TODO

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TODO

# Prior art
[prior-art]: #prior-art

C++ lambdas does not support implicit captures the way Rust closures does. Instead
the programmer must choose to either capture by value or by reference. There are
shorthands for capturing everything by value, and everything by reference as well.

```c++
auto no_capture = []() { std::print("No captures"); };
auto all_by_reference = [&]() { std::print("{} and {} captured by reference", a, b); };
auto all_by_value = [=]() { std::print("{} and {} captured by value", a, b); };
auto mix = [&a, b]() { std::print("{} by reference, {} by value", a, b); };
```

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How should `move clone || use(a, b)` handle non-clonable objects?
- Supporting renaming of cloned objects is considered outside the scope.
- Supporting capture list for `move` is considered outside the scope.

# Future possibilities
[future-possibilities]: #future-possibilities

Allowing a capture list for `move` keyword as well.

Allowing `move clone(mut a, b)` to mark that `a` should be mutable within the closure or async block.

Allowing `move clone(c = a, b)` to clone `a` to a new binding named `c`.