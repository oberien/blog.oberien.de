---
layout: post
title: "Compile Time Gauß on 3×4 Matrixes"
---

I had some fun with associated constants and implemented compile time Scalar, Matrix-Vector and Matrix-Matrix Multiplication. Then I decided to go ahead and implement a compile-time Gauß algorithm on 3×4 matrixes, which prints the intermediate right-upper matrix and the final solution of the linear equation.

On the stable channel, associated constant evaluation seems to be recursive, resulting in exponential compile times, which results in compilation to take longer thant he allowed 10 seconds on the playground. On nightly with miri this is not a problem anymore, in fact it complies faster than I expected.

You can find the implementation [in this playground](https://play.rust-lang.org/?gist=48e4f3fc0f6173f47b13c3fe23826a48&version=nightly).

---

[Discussion on reddit](https://www.reddit.com/r/rust/comments/858bta/compile_time_gau%C3%9F_on_34_matrixes/)
