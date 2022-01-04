---
layout: post
title: "Thoughts on return, break and continue"
---

Commonly, the keywords `return`, `break` and `continue` are used to influence the
control flow of a program.
Designing my own language [rebo](https://github.com/oberien/rebo), I'm at a point
where I want to implement a form of control-flow-changing operations.
This post is primarily a braindump of ideas I had thinking about the implementation.
Let's embark on journey through those ideas in the form of thought experiments.

## Only `break`

The keywords `return`, `break` and `continue` are only valid in some contexts.

`return` can only be used within functions, where it returns from the function it's
used in, either with a value or with void / unit.
```rust
fn my_function() {
    // return implicit unit
    return;
    // return expression (evaluating to unit in this case)
    return ();
}
```

The `break` keyword can only be used within loops (`for`, `while`, `loop`),
where it aborts the loop and continues executing the code afterwards.
(There are also languages where `break` is used within `switch` statements to abort
execution of those and continue with the code after the switch, but we'll ignore
those here.)
```rust
// only "before 0" will be printed
for i in 0..10 {
    println!("before {}", i);
    break;
    println!("after {}", i);
}
```

Similarly, `continue` can be used within loops (`for`, `while`, `loop`) to
abort the current execution of the loop and continue with the next iteration.
```rust
// only "before 0", "before 1", ..., "before 9" will be printed
for i in 0..10 {
    println!("before {}", i);
    continue;
    println!("after {}", i);
}
```

Some languages employ labels to allow breaking out of an outer loop:
```rust
'loop1: while true {
    while true {
        break 'loop1;
    }
}
```

With labels, we can get creative and start to explore new possibilities.
We can "label" a function and use `break` to "break out of" that function,
i.e., return from the function without having a `return` keyword:
```rust
'fun1: fn my_function() {
    break 'fun1;
}
```

However, that wouldn't allow returning a value.
You'd need to allow breaking with a value.
Rust already has this feature, called [loop-break-value](https://rust-lang.github.io/rfcs/1624-loop-break-value.html):
```rust
let foo = loop {
    break 1337;
};
```

The same feature could allow you to "return" a value from a function via `break`:
```rust
'fun1: fn my_function() -> int {
    break 'fun1 1337;
}
```

You can go further and also allow "early-returning" from any block (which
[is being considered for rust](https://rust-lang.github.io/rfcs/2046-label-break-value.html)):
```rust
let foo = 'block: {
    if true {
        break 'block 1337;
    }
    42
};
```

The `loop` expression can be seen as the `loop` keyword being followed
by a block, which is repeated indefinitely.
Allowing breaking out of blocks, you don't need the `continue` keyword:
```rust
'loop: loop 'block: {
    // continue
    break 'block;
    // break
    break 'loop;
}
```

As a generalization, every block can have an optional label.
That way function-labels could also just be the normal block label of the
function body block:
```rust
fn foo() -> int 'label: {
    break 'label 1337;
}
```

The same is true for loops, which also don't need special labels:
```rust
'loop: {
    loop 'block: {
        // continue
        break 'block;
        // break
        break 'loop;
    }
}
```

Using the keyword `break` for all of `break`, `continue` and `return` may feel weird.
Instead, a different keyword like `quit` or `exit` could be used
(or something else as those two are also somewhat taken).

## Only `return`

Instead of using break-with-value, we could use the standard `return` for everything.
This works especially well with closures.
Loops don't need to be keywords but could become recursive functions
(given [TCO](https://en.wikipedia.org/wiki/Tail_call)).
Those looping functions take a function representing the loop-body as argument.
Loop-body-functions return a value indicating whether the loop should break or continue.
```rust
enum LoopResult {
    Break,
    Continue,
}
```

The simplest of these cases is `loop`:
```rust
// signature of the `loop` function
fn loop(body: impl Fn() -> LoopResult) { ... }

let mut i = 0;
loop(|| {
    if i > 3 {
        return LoopResult::Break;
    }
    println!("loop: {}", i);
    i += 1;
    LoopResult::Continue
});
```

`while` would look like this:
```rust
// signature of the `while` function
fn while(condition: impl Fn() -> bool, body: impl Fn() -> LoopResult) { ... }

let mut i = 0;
while(|| i <= 3, || {
    println!("while: {}", i);
    i += 1;
});
```

Finally, `for` could be used like this:
```rust
// signature of the `for` function
fn for<T>(iterator: impl IntoIterator<Item = T>, body: impl Fn(T) -> LoopResult) { ... }

for(0..=3, |i| { println!("for: {}", i); LoopResult::Continue });
```

This approach is somewhat similar to pure functional languages where loops are
often represented as reduce operators on iterators / lists.

## "Extreme" Continuation Passing

Another solution could be a variation of "only `return`", also taking ideas from
the continuation passing style.
Similarly to above, looping operations are represented as functions.
Functions never return.
Instead of writing code after a function / loop, that code is wrapped in a function
and passed as argument to the previous function as continuation.
Every function has an implicit additional argument, which is the continuation.
Loops themselves don't exist.
They are replaced with calling the current function again at the end of its body.

```rust
fn my_loop(i: i32, continuation) {
    if i > 3 {
        // will never return
        continuation();
    }
    println!("{}", i);
    my_loop(i + 1, continuation);
    // unreachable
}
```

…Never mind, I think I just invented a variation of functional languages with TCO…  
NB: This is somewhat akin to (in)direct threading in interpreters.

## Labels as Values / Scoped Labels

So far we've only looked at labels available in the current context.
You can't break out of a label which you haven't seen yet during execution.
For example you can't "break out of" a label that is defined after the current loop:
```rust
'loop1: loop {
    // error
    break 'loop2;
}

'loop2: loop {}
```

Having such functionality would allow `goto` with extra steps:
```rust
// we want to jump to the code below
// create loop just for breaking out of it
loop { break 'label }

// define empty labeled loop to use for the break above
'label loop {}
// code we want to jump to
```

The main problem with `goto` and labels in C in my opinion is that labels are effectively global.
This opens up to lots of gotchas and possibilities for unclean code.

But nothing stops us from making labels scoped.
We can define labels to be globally available within the block / scope they are
defined in (and all sub-blocks / sub-scopes).
Labels could be their own built-in type, be passed around as special values and
used as function arguments.
`return`, `break`, `continue` and `goto` can take labels just as before.
```rust
fn return_to_label('to_return_to: Label<()>) {
    return 'to_return_to;
}

let 'label = 'outer_label;
return_to_label('label)

'outer_label;
// code
```

We also need to allow returning values to labels.
For that, labels evaluate to a value.
If the value is not unit, a default must be specified which is used if
the label-expression is reached without being jumped to.
```rust
fn return_to_label('to_return_to: Label<i32>) {
    return 'to_return_to 1337;
}

return_to_label('outer_label);

let result = 'outer_label else 42;
assert_eq!(result, 1337);
```

Getting different values from labels based on how they were reached
is similar to the φ (phi) function in llvm.

The idea of label values is possible in C already via the `setjmp` and `longjmp` functions.

In the end I just went with the standard `return`, `break` and `continue`.

---

Discussion [on reddit](https://www.reddit.com/r/rust/comments/rvr9zx/thoughts_on_return_break_and_continue/).

