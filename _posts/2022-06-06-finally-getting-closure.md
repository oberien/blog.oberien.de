---
layout: post
title: "Comparison of Closures in different Languages"
---

**Alternative Title: Finally getting Closure(s in rebo)**

---

Closures, also called lambdas, inline functions, or anonymous functions, are functions
that use variables from the outer scope, which they "close over".
Recently, I've wanted to implement closures in [rebo](https://github.com/oberien/rebo),
a statically type-checked rust-inspired scripting language.
Thus, I've been looking into how different languages implement and handle closures
to compare and find a solution for rebo.

## JavaScript

JS seems straightforward at first.
Every function can close over every variable in outer scopes.

```js
var x = 5;
function foo() {
    console.log(x); // 5
}
```

### Hoisting

However, this becomes more interesting with hoisting.
Conceptionally, whenever you declare a variable (in whichever way),
its declaration is hoisted to the very beginning of its scope.

Named functions like `function foo() {}` are hoisted to the top of the function /
module scope they are defined in, and their value is initialised to the function itself.

```js
foo(); // hi

function foo() {
    console.log("hi");
}
```

Variables like `var a = 42;` are hoisted to the top of the function /
module scope they are defined in, just above functions, and their value
is initialised to `undefined`.

The following code
```js
foo(); // undefined
function foo() {
    console.log(x);
}
var x = 5;
foo(); // 5
```
is interpreted as
```js
var x = undefined;
function foo() {
    console.log(x);
}

foo();
x = 5;
foo();
```

`let a = 42;` and `const a = 42` are hoisted to the top of the block scope they
are defined in, and their value is left uninitialized.
Thus, accessing their value before it's initialized results in
`ReferenceError: Cannot access 'x' before initialization`.
If we were to replace `var` with `let` in the above code, we'd get
`ReferenceError: Cannot access 'x' before initialization` when `foo()` is executed
before `x` is initialized.
The `foo()` after the initialization of `x` would run as expected with the output `5`.

And we're just gonna ignore undeclared global variables for sanity's sake[^undeclared].

&nbsp; | Hoisted Scope | Hoisted Value
--- | --- | ---
`var` | function / module | `undefined`
`function` | function / module | normal function value
`let` / `const` | block | uninitialized

### Capturing

Function values in JS contain not only their to-be-executed code, but
also a list of all variables they capture.
To be precise, they capture the bindings themselves.
They can't capture *references* as you can't take references to primitive types.
They also don't capture values, because you can reassign captured variables by
changing the outer value.

Any change of the binding will be reflected everywhere that binding is used.

```js
var a = 0;
function foo() {
    console.log(a);
    a = 1;
}
console.log(a); // 0
foo(); // 0
console.log(a); // 1
a = 2;
foo(); // 2
console.log(a); // 1
```

Changes of properties of an object are also visible globally.
```js
var a = { x: 0 };
function foo() {
    console.log(a.x);
    a = { y: 1 };
}
console.log(a); // { x: 0 }
foo(); // 0
console.log(a); // { y: 1 }
foo(); // undefined
a.x = 2;
console.log(a); // { y: 1, x: 2 }
foo(); // 2
```

### Quiz

Code 1:
```js
let x = 5;
(function() {
    console.log(x);
    let x = 7;
    console.log(x);
})();
```

Code 2 (`let` replaced with `var`):
```js
var x = 5;
(function() {
    console.log(x);
    var x = 7;
    console.log(x);
})();
```

Solution 1:
```js
ReferenceError: Cannot access 'x' before initialization
```
Solution 2:
```js
undefined
7
```

Those results are possibly unexpected even for experienced JS developers,
but given hoisting, they make sense.

## C#

In C# lambda expressions capture bindings semantically similar to JS.
However, while JS code is interpreted directly (i.e., JITed by V8),
C# is first compiled to MSIL bytecode which is executed by the language's VM called CLR.
The MSIL bytecode doesn't know about closures.
Instead, the C# compiler performs a transformation that converts closures into
regular C# classes.
For every function that lambda expressions are defined in, the C# compiler
creates an anonymous class.
That class has one field per captured variable.
All accesses within the function to captured variables are rewritten to accesses
into fields of that class.
The `this` variable of the outer class is also stored as a field to allow access
to outer fields.

Take the following code:

```csharp
class Foo
{
    static void Main(string[] args)
    {
        new Foo().MyMain();
    }

    void MyMain() {
        var a = 0;
        Action foo = () => {
            Console.WriteLine(a);
            a = 1;
        };
        Action bar = () => {
            Console.WriteLine(a);
        };
        Console.WriteLine(a); // 0
        foo(); // 0
        bar(); // 1
        a = 2;
        bar(); // 2
        foo(); // 2
        Console.WriteLine(a); // 1
    }
}
```

It'll be converted into something akin to:
```csharp
class Foo
{
    class AnonymousMyMain
    {
        internal int A;
        internal Foo OuterThis;

        internal void Foo()
        {
            Console.WriteLine(this.A);
            this.A = 1;
        }
        internal void Bar()
        {
            Console.WriteLine(this.A);
        }
    }

    static void Main(string[] args)
    {
        new Foo().MyMain();
    }

    void MyMain() {
        var anonMain = new AnonymousMyMain();
        anonMain.OuterThis = this;
        anonMain.A = 0;

        Console.WriteLine(anonMain.A); // 0
        anonMain.Foo(); // 0
        anonMain.Bar(); // 1
        anonMain.A = 2;
        anonMain.Bar(); // 2
        anonMain.Foo(); // 2
        Console.WriteLine(anonMain.A); // 1
    }
}
```

At first sight, that solution seemed crazy to me.
On second thought, the solution is pretty clever as closures are only a
construct in the compiler.
The interpreter / VM handles them automatically without any further implementation.

## Java

Similar to C#, Java first compiles code into bytecode, which is executed by the JVM.
Unlike C#, in Java lambda expressions capture values instead of bindings.
It's the same behaviour as if all captured variables were to be passed as
function arguments (which in Java is pass-by-value with references).
Because that's exactly what's happening.

Let's try running the following code:
```java
public class Foo {
    public static void main(String[] args) {
        int a = 0;
        Runnable foo = () -> System.out.println(a);
        foo.run();
        a = 1;
        foo.run();
    }
}
```
This results in a compiler error:
```
error: local variables referenced from a lambda expression
       must be final or effectively final
```
All variables are passed by value.
Assigning a different value to a variable will not be reflected in the respective other bindings.
Java protects us from this possible pitfall by disallowing reassigning of captured variables.
We'll need to mark `int a` as `final` and remove the line `a = 1` (or just remove `a = 1`,
making `a` an "effectively final" variable as it's only assigned to a single time).

```java
public class Foo {
    public static void main(String[] args) {
        final int a = 0;
        Runnable foo = () -> System.out.println(a);
        foo.run(); // 0
    }
}
```
Traditionally, when you wanted to accept a function as parameter,
you had to define an interface.
The caller had to define a class implementing that interface and pass an instance
of that class (explicitly or anonymously).
This kind of sucked.  
To make this easier, Java 1.8 introduced lambda expressions which can be used instead.
Any interface with a single function can since be written as a lambda expression.

Java converts lambda expressions into static functions of the surrounding class.
The arguments of the generated function are one argument per captured variable
followed by the lambda's arguments.
Additionally, it generates an anonymous class implementing the interface.
That class has one field per captured variable.
Those captures are passed as arguments to the constructor.
The interface method's implementation is a call to the static function.[^java-lambdas]

As such, the above code gets translated into the following:
```java
public class Foo {
    public static void main(String[] args) {
        final int a = 0;
        Runnable foo = new Lambda1(a);
        foo.run(); // 0
    }
    static void foo(final int a) {
        System.out.println(a);
    }
    class Lambda1 implements Runnable {
        int a;
        Lambda1(int a) {
            this.a = a;
        }

        void run() {
            Foo.foo(this.a);
        }
    }
}
```

To exchange data between the surrounding code and the lambda, we'll need to use
a reference type, storing shared values in its fields.

```java
public class Foo {
    static class Data {
        int a;
    }
    public static void main(String[] args) {
        final Data data = new Data();
        data.a = 0;
        Runnable foo = () -> {
            System.out.println(data.a);
            data.a = 1;
        };
        System.out.println(data.a); // 0
        foo.run(); // 0
        System.out.println(data.a); // 1
        data.a = 2;
        foo.run(); // 2
        System.out.println(data.a); // 1
    }
}
```
Translation:
```java
public class Foo {
    static class Data {
        int a;
    }
    public static void main(String[] args) {
        final Data data = new Data();
        data.a = 0;
        Runnable foo = new Lambda1(data);
        System.out.println(data.a); // 0
        foo.run(); // 0
        System.out.println(data.a); // 1
        data.a = 2;
        foo.run(); // 2
        System.out.println(data.a); // 1
    }
    static void foo(final Data data) {
        System.out.println(data.a);
        data.a = 1;
    }
    static class Lambda1 implements Runnable {
        Data data;
        Lambda1(Data data) {
            this.data = data;
        }
        public void run() {
            Foo.foo(data);
        }
    }
}
```

The implementation of lambdas in Java could have been similar to C# such that
the JVM doesn't need to know about the existence of lambda expressions.
The compilation to bytecode could generate the anonymous classes implementing the interface.
However, Java decided to only perform a primitive transformation during compilation
and leave the final interface implementation up to the runtime.
This allows substituting the current anonymous class approach with a better
implementation in the future, should new language concepts allow for such.

## Python

Python captures bindings.
```python
x = 5
c = lambda: x
print(c()) // 5
x = 7
print(c()) // 7
```

As lambdas are advertised as inline functions and are usually used within a line,
the lifetime of bindings can have unexpected results.
Take the following code, which creates a list of lambda functions:
```python
l = [(lambda: x) for x in range(10)]
[f() for f in l]
```
The expectation would be to get a list from 0 through 9.
However, we get an array with ten times the number 9 (`[9, 9, …, 9]`).
This is because for-loops in python reuse the binding between all iterations
instead of creating a new binding every iteration.

We can force a copy of the value by creating a function that takes the value
as the first argument and returns a function capturing the argument.
As arguments are passed by-value in python, a copy of the iteration variable is made.
```python
l = [(lambda x: lambda: x)(x) for x in range(10)]
[f() for f in l]
```

That code results in `[0, 1, 2, …, 9]`.

This happens so often that this behaviour of lambdas in for-loops even has its own
[FAQ entry](https://docs.python.org/3/faq/programming.html#why-do-lambdas-defined-in-a-loop-with-different-values-all-return-the-same-result).

## Rust

Rust, similar to C#, transforms closures into structs which have one field per captured variable.
These structs implement function traits that allow calling the structs as if
they were functions.
Usually, when you implement a trait, you have to call its methods through the dot
operator (`foo.bar()`).
With the function traits, you can instead directly call the name (`foo()`).

There are three different function traits which express different guarantees
in accordance with rust's ownership model.
The trait `Fn` states that the closure only borrows values and doesn't modify them.
They can be executed any number of times and only take `&self` as argument.  
`FnMut` is implemented for all closures that capture variables mutably,
i.e., modify captured variables during execution.
Thus, they take `&mut self` as argument and can also be executed any number of times.
It is also implemented for all `Fn` closures.  
The final trait is `FnOnce`.
In addition to all `FnMut` implementers, `FnOnce` is implemented for closures that
consume themselves while executing, taking `self` as argument.
For example, if values are moved into a closure and dropped in its body,
the closure can't run a second time.

Let's take this code:
```rust
let a = 5;
let closure = || println!("{}", a + 2);
closure(); // 7
```
This will be translated into:
```rust
struct Closure<'a> {
    a: &'a i32,
}
impl<'a> FnOnce<()> for Closure<'a> {
    type Output = ();
    extern "rust-call" fn call_once(self, _args: ()) -> () {
        println!("{}", *self.a + 2);
    }
}
impl<'a> FnMut<()> for Closure<'a> {
    extern "rust-call" fn call_mut(&mut self, _args: ()) -> () {
        println!("{}", *self.a + 2);
    }
}
impl<'a> Fn<()> for Closure<'a> {
    extern "rust-call" fn call(&self, _args: ()) -> () {
        println!("{}", *self.a + 2);
    }
}

let a = 5;
let closure = Closure { a: &a };
closure(); // 7
```

Rust gives the programmer a lot of control over how variables are captured in closures.
But rust is also very good at figuring out how to capture variables correctly automatically.
Usually, when writing closures everything is captured in a way to make it *just work*&#8482;.
It makes using closures feel natural and expected.  
What is unexpected, though, is how complex the internals of capturing are and how
deep the rabbit hole goes.
In fact, rust has four different capture modes: pre-edition-2021 `move` closures,
post-edition-2021 `move`-closures, pre-edition-2021 non-`move` closures, and
post-edition-2021 non-`move` closures.

### Pre-edition-2021 `move` Closures

`move` closures move all bindings / values into the closure.
The bindings can't be used outside of the closure anymore.
As usual, `Copy` types aren't moved but copied into the closure so their bindings
can still be used outside, but their value has been duplicated.
As (mutable) references are just regular types, bindings can also be references.
In that case, the binding and the binding's value is moved into the closure,
but the binding's value is the reference.
This allows moving mutable references pointing to data outside the closure into
the closure, being able to modify them from within the closure.
However, this also means that the borrowed value is borrowed until the last
usage of the closure.
If a field of a struct is accessed, the whole struct is moved into the closure.
This means that the outer function can't access any fields of that struct even
if the fields accessed are disjoined.

```rust
struct Foo {
    a: String,
    b: String,
}

let mut foo = Foo { a: "abc".to_string(), b: "xyz".to_string() };
let mut a = 5;
let mut closure = move || {
    println!("{}", foo.a);
    foo.a = "def".to_string();
    println!("{}", a);
};

closure(); // "abc", 5
// error: borrow of moved value: `foo`
//println!("{}", foo.b);
a = 7;
closure(); // "def", 5
// error: borrow of moved value: `foo`
//println!("{}", foo.b);
```

### Post-edition-2021 `move` Closures

The behaviour of a whole struct being moved into the closure even though only
a field is accessed was a pain point in a lot of real-world code.
Therefore, [RFC 2229](https://rust-lang.github.io/rfcs/2229-capture-disjoint-fields.html)
proposed to only capture the actual field.
This change was applied and stabilized with the 2021 edition and is the primary
difference between pre- and post-edition-2021 `move` closures.

```rust
struct Foo {
    a: String,
    b: String,
}

let mut foo = Foo { a: "abc".to_string(), b: "xyz".to_string() };
let mut closure = move || {
    println!("{}", foo.a);
    foo.a = "def".to_string();
};

closure(); // "abc"
println!("{}", foo.b); // "xyz"
closure(); // "def"
println!("{}", foo.b); // "xyz"
```

### Pre-edition-2021

If a closure is not a `move` closure, rust captures the bindings.
It achieves this by analysing the closure's code to find out how captured variables are used.
The compiler captures each variable using the least invasive capture mode required.
It tries in order to capture each respective variable as shared reference,
unique immutable reference (yes, such a thing exists), mutable reference, or
by moving the value into the closure.
For example `|| drop(x)` must move `x` into the closure while `|| println!("{x}")`
and `|| *x += 5` only require a shared and mutable reference respectively.
If bindings are captured by creating a new reference to them, those references behave
transparent by automatically being dereferenced on access (similar to C++ references).

Unique immutable references are an implementation detail of the rust compiler
for handling closure captures.
A user can't explicitly declare a unique immutable reference.
You can read about them in the [reference](https://doc.rust-lang.org/reference/types/closure.html#unique-immutable-borrows-in-captures),
the [rustc-dev-guide](https://rustc-dev-guide.rust-lang.org/closure.html),
and the [rustc-middle API docs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.BorrowKind.html#variant.UniqueImmBorrow).
It is required in cases where a non-mutable binding is a mutable
reference, which is modified within the closure.
Take the code from the example in the reference:

```rust
let mut b = false;
let x = &mut b;
let mut c = || { *x = true; };
// this produces an error
// println!("{x}");
c();
println!("{b}");
```

Let's try the three other capture modes in order, leaving out the unique immutable
reference capture mode.
Rust can't borrow `x` as a shared immutable reference because it is modified
within the closure.
It can't capture a mutable reference to `x` because the binding `x`
isn't declared as mutable.
Moving `x` into the closure doesn't make semantic sense as that would remove the
binding from the outer scope when it doesn't need to be.
To capture `x`, we need to take an immutable reference to it, through which it's
allowed to mutate `x`.
And that's exactly what the unique immutable reference does.

We can see this behaviour in the produced MIR (comments inserted):
```rust
_1 = const false;                 // let mut b = false;
_2 = &mut _1;                     // let x = &mut b;
_4 = &mut _2;                     // let temp_x = &uniq x;
// &uniq is displayed as &mut in the MIR output
(_3.0: &mut &mut bool) = move _4; // let mut temp2_x = move temp_x;
_6 = &mut _3;                     // let captured_x = &mut temp2_x;
// create closure capturing `captured_x` as `&mut &uniq &mut b`
_5 = <[closure@src/main.rs:4:13]>::call_mut(move _6, move _7) -> bb1;
```

If we uncomment the `println!("{x}")` line, we get an error explaining that
`x` is captured uniquely:
```rust
error[E0501]: cannot borrow `x` as immutable because previous closure requires unique access
 --> src/main.rs:5:12
  |
4 | let mut c = || { *x = true; };
  |             --   -- first borrow occurs due to use of `x` in closure
  |             |
  |             closure construction occurs here
5 | println!("{x}");
  |            ^ second borrow occurs here
6 | c();
  | - first borrow later used here
  |
  = note: this error originates in the macro `$crate::format_args_nl` (in Nightly builds, run with -Z macro-backtrace for more info)
```


### Post-edition-2021

As stated before, in edition 2021 rust changed how closure capturing works.
To be more precise, rust now captures places instead of bindings.
Previously, the whole binding was captured and thus unmodifiable while
the closure exists.
With the changes, rust captures the places where the values are stored instead
of the bindings.

If we compile the above example with edition 2021, `x` isn't captured as
unique immutable reference anymore.
Instead, it is mutably reborrowed as seen in the MIR output (comments inserted).
Notice the difference of `let temp_x = &mut *x` compared to the previous `let temp_x = &uniq x`:

```rust
_1 = const false;            // let mut b = false;
_2 = &mut _1;                // let x = &mut b;
_4 = &mut (*_2);             // let temp_x = &mut *x;
(_3.0: &mut bool) = move _4; // let mut temp2_x = move temp_x;
_6 = &mut _3;                // let captured_x = &mut temp2_x;
// create closure capturing `captured_x` as `&mut &mut b`
_5 = <[closure@src/main.rs:4:17: 4:34]>::call_mut(move _6, move _7) -> bb1;
```

The new behaviour allows bindings to be modified even when their pointed-to
value is captured.
This is demonstrated by the below (hopefully never to be seen in the wild) code:
```rust
let mut x = 42;
let mut a = &mut 0i32;
let mut closure = || { *a = 5; };
a = &mut x;
closure();
println!("{a}"); // 42
```

Pre-edition-2021 the code results in an error:
```rust
error[E0506]: cannot assign to `a` because it is borrowed
 --> src/main.rs:5:1
  |
4 | let mut closure = || { *a = 5; };
  |                   --   -- borrow occurs due to use in closure
  |                   |
  |                   borrow of `a` occurs here
5 | a = &mut x;
  | ^^^^^^^^^^ assignment to borrowed `a` occurs here
6 | closure();
  | ------- borrow later used here
```

Post-edition-2021 the code prints `42`.
Its MIR looks like this (comments inserted):
```rust
_1 = const 42_i32;               // let mut x = 42;
_3 = const 0_i32;                // let mut temp = 0;
_2 = &mut _3;                    // let mut a = &mut temp;
_5 = &mut (*_2);                 // let temp_a = &mut *a;
(_4.0: &mut i32) = move _5;      // let mut temp2_a = move temp_a;
_7 = &mut _1;                    // let temp2 = &mut temp;
_6 = &mut (*_7);                 // let temp3 = &mut *temp2;
_2 = move _6;                    // a = move temp3;
_9 = &mut _4;                    // let captured_a = &mut temp2_a;
// create closure capturing `captured_x` as `&mut &mut temp`
_8 = <[closure@src/main.rs:4:19: 4:33]>::call_mut(move _9, move _10) -> bb1;
```

Rust closure capturing gives the programmer the option to have full control.
However, it has lots of inherent and internal complexity.
Normal code (for some definition of normal) works intuitively.
But some more or less contrived code can result in incredibly unexpected behaviour.
I've also heard from some people that they always and only use `move` closures
to make capturing explicit instead of relying on the implicit inferred capture modes.

## Comparison

JS, C#, and Python semantically capture bindings (albeit with JS requiring
lots of other implicit knowledge to find out which variables are actually captured).
Java captures values.
Rust allows the programmer to decide how to capture variables and by default
infers if bindings or values are captured as needed.

The translation of C# is pretty clever in that the runtime doesn't need to be
changed to support lambdas.  
Java disallows modification of the variables holding captured values
to prevent unintuitive reassignment to copied primitive values.  
Rust has an awful lot of special rules and preforms a lot of internal trickery
to make the captures of closures as intuitive and expected as possible.

## Rebo

Previously, rebo didn't have closures.
It had named functions and unnamed / anonymous functions.
Named functions are global and thus exist before any other code is executed.
They can only use statics and aren't able to capture variables.  
Unnamed or anonymous functions are defined inline.
They are created only when the expression they're defined in is evaluated.

As such, anonymous function could be able to capture variables defined anywhere
in a parent scope.
The question is how to capture variables: by value or by binding.
Rebo currently acts similar to Java, C#, JS and python.
There isn't an explicit notion of references.
Instead, primitives (including strings) are value-types, while structs and enums
are reference-types.
Function arguments are passed by value (with references).

To keep thinks simple, anonymous functions / unnamed functions / closures will
capture variables by value.
This decision simplifies the implementation as it can reuse the way arguments
are passed to functions.
It doesn't require changing bindings to be able to be used from different places.
It's also not unexpected behaviour as Java handles it the same way.
If a programmer wishes to modify a value from both inside and outside the closure,
they can wrap it in a struct.

Initially, contrary to Java, rebo didn't require captured primitive bindings to
be declared as immutable.
Instead, they could be modified, even when the behaviour might be unexpected in some cases.
However, after porting code from functions using statics to closures using local
variables, I realised that exactly this unexpected behaviour was introduced
several times, resulting in hard to track-down bugs.
Therefore, I added a lint to disallow mutable primitives being captured, similar to Java.
This makes the compiler find possible bugs, while a workaround is easy to write
and signifies the intend better.

---

Discussion on [r/rust](https://www.reddit.com/r/rust/comments/v5zekj/).

---

[^undeclared]:
    Undeclared variables without a preceding keyword (`a = 42;`) are
    in the global scope but are not hoisted.
    They only exist once they are assigned to the first time and produce a
    `ReferenceError` otherwise.
    ```js
    function foo() {
        console.log(a);
    }
    foo(); // ReferenceError: a is not defined
    a = 10;
    foo(); // 10
    ```
    Trying to rewrite the code to how JS interprets it takes away from the
    topic of this post.
    Rewriting `a` to `global.a` (or `window.a` in browsers) doesn't work directly
    because it'll result in `undefined` instead of `ReferenceError` if `a`
    hasn't been assigned to yet.
    The rewrite of the above code would be
    ```js
    function foo() {
        if (!global.hasOwnProperty("a")) {
            throw new ReferenceError("a is not defined");
        }
        console.log(global.a);
    }
    foo(); // ReferenceError: a is not defined
    global.a = 10;
    foo(); // 10
    ```

[^java-lambdas]:
    In reality [it's a bit more complicated](https://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html).
    The java compiler itself produces the static function.
    At the creation site it generates [an `invokedynamic` call](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.invokedynamic).
    The instruction `invokedynamic` [is designed for dynamic languages](https://stackoverflow.com/a/61043989)
    where a different method call may be chosen depending on metadata
    like the receiver, number of arguments, argument types, etc.
    It invokes a function provided by the VM implementation, which returns an
    object representing the lambda.
    In typical Java fashion, the `invokedynamic` call site is called `lambdaFactory`
    while the VM-provided function returning the object is called `lambdaMetafactory`.
    The [`LambdaMetafactory`](https://github.com/openjdk/jdk/blob/a445b66e58a30577dee29cacb636d4c14f0574a2/src/java.base/share/classes/java/lang/invoke/LambdaMetafactory.java)
    currently (since JDK1.8 through JDK14) defers to the [`InnerClassLambdaMetafactory`](https://github.com/openjdk/jdk/blob/a445b66e58a30577dee29cacb636d4c14f0574a2/src/java.base/share/classes/java/lang/invoke/InnerClassLambdaMetafactory.java).
    That class during runtime dyanmically creates bytecode for the inner class.
    It creates the constructor taking each captured variable as argument and
    storing them in fields.
    Additionally, it generates the implementation of the interface by calling the static
    function generated by `javac` forwarding the captured-variable-fields and
    interface method arguments.
    Afterwards, it calls the constructor with the captured arguments which have
    been passed as part of the `invokedynamic` invocation.
    The resulting object is returned.  
    This dynamic approach was chosen to allow more efficient ways of handling
    and representing lambda expressions in the future.
