---
title: "Rust const-fn compile-time SUBLEQ interpreter"
---
With the [minimal subset of `const fn`](https://github.com/rust-lang/rust/issues/53555) becoming stable soon (in the second next Rust version), I wanted to give const fns a try and test what is possible with them.  
We implemented a compile-time SUBLEQ interpreter which only uses const-fns, which you can find [on the playground](https://play.rust-lang.org/?version=nightly&mode=release&edition=2015&gist=82cc33ea9cb221812a4794bb47e6bf50).

Let's walk through the process of building this abomination :)

## What is SUBLEQ?

SUBLEQ is a one-instruction set computer, named by its only instruction **SU**btract and **B**ranch if **L**ess than or **EQ**ual to zero. The instruction `subleq a, b, c` performs `mem[b] = mem[b] - mem[a]; if mem[b] <= 0 { goto c; }`. This single instruction is Turing complete. You can find out more about it [here](https://rosettacode.org/wiki/Subleq).

The SUBLEQ interpreter works as follows:
* Read the three values `a`, `b`, and `c` from the instruction pointer and advance it by 3.
* If the instruction pointer is negative, stop execution and print the output buffer.
* If `a` is `-1`, then one byte is read from the input and stored at address `b`.
* If `b` is `-1`, then the byte stored at address `a` is written to the output.
* Otherwise, perform the `subleq` instruction.

## Minimal subset of const fns

The soon-to-be stabilized minimal subset of const fns doesn't actually offer that much. For example, you're only allowed very primitive expression-like constructs including arithmetic operators, struct / tuple / array creation and access. You *aren't* allowed any sorts of control flow constructs like `if`, `match`, `while`, `for` or `loop`. In fact using the latter results in an ICE. You can find an exhaustive list in [its tracking issue](https://github.com/rust-lang/rust/issues/53555).

## Branching

"*But how do you branch without `if`*" I hear you ask? Let's copy an idea of [the movfuscator](https://www.youtube.com/watch?v=HlUe0TUHOIc). We precompute the results from either path and put them in an array. Then we index into the array based on the condition.

A trivial example is a function `either` taking a boolean and two values, returning the first value on `true` and the second on `false`. The generic solution would be this:

    const fn either<T>(cond: bool, a: T, b: T) -> T {
        [a, b][!cond as usize]
    }

The concept works, but there is a problem: `T` might implement `Drop`, which is not allowed in `const fns`. Currently only lifetime-bounds and the trait `Sized` are allowed as type bounds, so we can't specify `T: Copy` either. Thus, we need to monomorphise the function manually. Luckily, we can get away with only `i32`:

    const fn either(cond: bool, a: i32, b: i32) -> i32 {
        [a, b][!cond as usize]
    }

This solution means that we can just have an array of functions, select the needed function and call it. Or does it…? Sadly, we **can not** use the code below, as `function pointers in const fn are unstable`.

    const fn call_function(index: usize, arg: Arg) -> Ret {
        [output, read, write, subleq][index](arg)
    }

## Assignments

Unfortunately, assignments aren't supported yet. Neither let-bindings nor reassigning mutable function arguments works.  
Instead, we borrow from functional languages and always pass the entire state to functions, which create and return a new, modified state. For example, we can use this (simplified) helper function to set a value of an array at the given index:

    const fn set(array: [i32; 4], idx: usize, val: i32) -> [i32; 4] {
        [
            either(idx == 0, val, array[0]),
            either(idx == 1, val, array[1]),
            either(idx == 2, val, array[2]),
            either(idx == 3, val, array[3]),
        ]
    }

## Looping

Looping can be achieved by recursively calling const fns. For example, an infinite, counting loop could be achieved like this:

    const fn count(i: i32) -> i32 {
        count(i+1)
    }

Unfortunately, we can't use an exit condition, because we don't have actual branching. We could fall back to our branching concept. Unfortunately, precomputing all results means that we'll call ourselves recursively before "branching", resulting in an infinite loop. Consider the following code:

    const fn subleq(data: Data) -> Data {
        [output(data), read(data), write(data), subleq(data)][index]
    }

In this example all four possible values are evaluated before choosing the result. This means `subleq` will always call itself recursively. So we cannot "cheat" control flow with function pointers. We can use either branching or looping, but not loop conditionally. 

## Actual "Solution"

The final solution was to unroll the loop manually. When invoking the function, we actually do `subleq(subleq(subleq(…)))` *n* times, hoping it'll be enough to get the final result.

Because all four possible branches are evaluated, some functions (read, write, subleq) might be called with invalid input. For example `subleq(-1, _, _)` could be invoked, which results in the use of address `-1`. These cases must be handled correctly to not cause a panic. What works in our favour is that these cases can produce garbage. Their result is discarded anyways.

Another problem arises during index calculation, where we might end up calculating `(-1 as usize) + 1`. This will cause a panic due to an overflow. We could use the `wrapping_*` function family, but they aren't `const fn`s yet. We just turn on release mode, as that won't panic on overflows.

## Conclusion

Is the minimal subset of const fns turing complete? The answer is a clear yes, no, and maybe.

* Yes, we can interpret SUBLEQ, which is Turing complete. 
* No, we can only execute a finite number of iterations.
  Generally, these kind of practical limitations tend to be overlooked, though.
* Maybe, if we find a different way of aborting early from recursive calls or a different way of looping. 
  (Even if we'd find a way to abort, we'd still only have a maxiumum of 96 iterations, as the const fn evaluator won't allow more than 96 stack frames)

If you have an idea how to improve it, please tell us :)

## Fun Fact

The `subleq` function can be considered a fixpoint iterator. If the function converges (e.g. if there is a result), each iteration will bring you closer to that result. Once the result has been found, `subleq` will just return the result itself, so any future invocation is functionally a NOP. This property is what allows us to unroll the loop in the first place.

---

[Discussion on reddit](https://www.reddit.com/r/rust/comments/9o6vzo/constfn_compiletime_subleq_interpreter/)
