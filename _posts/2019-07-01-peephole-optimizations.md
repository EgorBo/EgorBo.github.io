---
layout: post
date: 2019-06-25 11:32 PM
title: "Peephole optimizations in C++ and C#"
description: 
comments: true
category: 
-
tags:
- llvm
- C++
- C#
- optimizations
---

> "Performance gains due to improvements in compiler optimizations will double   
> the speed of a program every 18 years" © [Proebsting’s Law](http://proebsting.cs.arizona.edu/law.html)

When we solve equations, we try to simplify them first, e.g. `Y = -(5 - X)` can be simplified to just  `Y = X - 5`. In modern compilers it's called "Peephole Optimizations". Roughly speaking, compilers search for certain patterns and replace them with corresponding simplified expressions. In this blog post I'll list some of them which I found in LLVM, GCC and .NET Core (CoreCLR) sources.

Let's start with simple cases:

{% highlight csharp linenos %}
  X * 1          =>  X
 -X * -Y         =>  X * Y
-(X - Y)         =>  Y - X
  X * Z - Y * Z  =>  Z * (X - Y)
{% endhighlight %}

and check the 4th one in C++ and C# compilers:

{% highlight csharp linenos %}
int Test(int x, int y, int z) {
    return x * z - y * z;       //  =>  z * (x - y)
}
{% endhighlight %}

Now let's take a look at what the compilers output:
<figure class="alignleft">
{% highlight nasm linenos %}
Test(int, int, int):
  mov eax, edi
  sub eax, esi   ; -         
  imul eax, edx  ; *         
  ret
{% endhighlight %}
	<figcaption>C++ (Clang, GCC, MSVC)</figcaption>
</figure>

<figure class="alignleft">
{% highlight nasm linenos %}
C.Test(Int32, Int32, Int32)  
  mov eax, edx
  imul eax, r9d  ; *
  imul r8d, r9d  ; *
  sub eax, r8d   ; -
  ret
{% endhighlight %}
	<figcaption>C# (RyuJIT)</figcaption>
</figure>
<figure class="aligncenter">
</figure>

All three C++ compilers have just one `imul` instruction. C# (.NET Core) has two because it has a very limited set of available peephole optimizations and I'll list some of them later. Be sure to note, the entire InstCombine transformation implementation, where peephole optimizations live, in LLVM takes more than 30K lines of code (+20k LOC in DAGCombiner.cpp). By the way, [here is the piece of code in LLVM](https://github.com/llvm-mirror/llvm/blob/45adfa50b3fddb97d7fc512cec80e48c551f3280/lib/Transforms/InstCombine/InstCombineAddSub.cpp#L1329-L1332) responsible for the pattern we are inspecting now. GCC has a special DSL which describes all peephole optimizations, and [here is the piece of that DSL](https://github.com/gcc-mirror/gcc/blob/5882c51592109e2e228d3c675792f891a09b43d6/gcc/match.pd#L2185-L2220) for our case.  
  
I decided, just for this blog post, to try to implement this optimization for C# in JIT (hold my beer 😛):
<figure class="aligncenter">
	<img src="/images/instcombine/jit-1.png" />
</figure>
<br/>
Let's now test my JIT improvement (see [EgorBo/coreclr](https://github.com/EgorBo/coreclr/commit/3d0abaa2c9919a48110a66b3fe19c7abed2bf041) commit for more details) in VS2019 with Disasmo:
<br/><br/>
<figure class="aligncenter">
	<img src="/images/instcombine/jit-2.png" />
	<figcaption>lea + imul instead of imul + imul + add</figcaption>
</figure>

<br/><br/>

Let's go back to C++ and trace the optimization in Clang. We need to ask clang to emit LLVM IR for our C++ code via `-emit-llvm -g0` flags (see [godbolt.org](https://godbolt.org/z/RZQTDV)) and then give it to the LLVM optimizer **opt** via `-O2 -print-before-all -print-after-all` flags in order to find out what transformation actually removes that extra multiplication from the `-O2` set. (see [godbolt.org](https://godbolt.org/z/3f0TyT)):

<figure class="aligncenter">
{% highlight llvm linenos %}
; *** IR Dump Before Combine redundant instructions ***
define dso_local i32 @_Z5Case1iii(i32, i32, i32) {
  %4 = mul nsw i32 %0, %2
  %5 = mul nsw i32 %1, %2
  %6 = sub nsw i32 %4, %5
  ret i32 %6
}

; *** IR Dump After Combine redundant instructions ***
define dso_local i32 @_Z5Case1iii(i32, i32, i32) {
  %4 = sub i32 %0, %1
  %5 = mul i32 %4, %2
  ret i32 %5
}
{% endhighlight %}

	<figcaption>It's InstCombine!</figcaption>
</figure>

So it's InstCombine indeed, we can even use it as the only optimization for our code for tests via `-instcombine` flag passed to `opt`:
<figure class="aligncenter">
	<img src="/images/instcombine/p2.png" />
</figure>

<br/>

Let's go back to the examples. Look what a cute optimization I found in GCC sources:
{% highlight cpp linenos %}
X == C - X  =>  false if C is odd
{% endhighlight %}
And that's true, e.g.: `4 == 8 - 4`. Any odd number for C (C usually means a constant/literal) will always be false for the expression:
<figure class="aligncenter">
	<img src="/images/instcombine/p3.png" />
	<figcaption>Foo2(int x) always returns false. LLVM doesn't have this optimization.</figcaption>
</figure>

### Optimizations vs IEEE754

Lots of this type of optimizations work for different data types, e.g. `byte`, `int`, `unsigned`, `float`. The latter is a bit tricky e.g. you can't simplify `A - B - A` to `-B` for floats/doubles, even `(A * B) * C` is not equal to `A * (B * C)` due to the [IEEE754 specification](https://en.wikipedia.org/wiki/IEEE_754). However, C++ compilers have a special flag to let the optimizers be less strict around IEEE754, NaN and other FP corner cases and just apply all of the optimizations - it's usually called "Fast Math" (`-ffast-math` for clang and gcc, `/fp:fast` for MSVC). Btw, here you can find my feature request for .NET Core to introduce the "Fast Math" mode there: [dotnet/coreclr#24784](https://github.com/dotnet/coreclr/issues/24784)).

As you can see, two `vsubss` were eliminated in the `-ffast-math` mode:
<figure class="aligncenter">
	<img src="/images/instcombine/p5.png" />
	<figcaption></figcaption>
</figure>

The C++ optimizers also support `math.h` functions, e.g.:
{% highlight cpp linenos %}
abs(X) * abs(X)  =>  X * X
{% endhighlight %}
The square root is always positive:
{% highlight cpp linenos %}
sqrt(X) < Y  => false, if Y is negative.
sqrt(X) < 0  => false
{% endhighlight %}
Why should we calculate sqrt(X) if we can just calculate C^2 in compile time instead?:
{% highlight cpp linenos %}
sqrt(X) > C  => X > C * C
{% endhighlight %}
<p/>
<figure class="aligncenter">
	<img src="/images/instcombine/sqrt.png" />
</figure>
<p/>
More sqrt optimizations:

{% highlight c linenos %}
sqrt(X) == sqrt(Y)  => X == Y
sqrt(X) * sqrt(X)   => X
sqrt(X) * sqrt(Y)   => sqrt(X * Y)
logN(sqrt(X))       => 0.5 * logN(X)
{% endhighlight %}
or `exp`:

{% highlight c linenos %}
exp(X) * exp(Y)  => exp(X + Y)
{% endhighlight %}

And my favorite one:

{% highlight c linenos %}
sin(X) / cos(X)  => tan(X)
{% endhighlight %}

<p/>

<figure class="aligncenter">
	<img src="/images/instcombine/p6.png" />
	<figcaption></figcaption>
</figure>

There are lots of boring bit/bool patterns:

{% highlight c linenos %}
((a ^ b) | a) -> (a | b)
(a & ~b) | (a ^ b)  -->  a ^ b
((a ^ b) | a) -> (a | b)
(X & ~Y) |^+ (~X & Y) -> X ^ Y
A - (A & B) into ~B & A
X <= Y - 1 equals to X < Y
A < B || A >= B -> true
... hundreds of them ...
{% endhighlight %}

### Machine-dependent optimizations

Some operations may be faster or slower on different CPUs, e.g.:

{% highlight c linenos %}
X / 2  =>  X * 0.5
{% endhighlight %}

<p/>

<figure class="aligncenter">
	<img src="/images/instcombine/p8.png" />
	<figcaption></figcaption>
</figure>
  
`mulss`/`mulsd` usually have better both latency and throughput than `divss`/`divsd` for example, here is the spec for my Intel Haswell CPU:
<figure class="aligncenter">
	<img src="/images/instcombine/p7.png" />
	<figcaption></figcaption>
</figure>

We can replace `/ C` with `* 1/C` even in the non-"Fast Math" mode if `C` is a power of two. Btw, here is my PR for .NET Core for this optimization: [dotnet/coreclr#24584](https://github.com/dotnet/coreclr/pull/24584).

The same rationale for:
{% highlight cpp linenos %}
X * 2  =>  X + X
{% endhighlight %}
<p/>
`test` is better than `cmp` (see my PR [dotnet/coreclr#25458](https://github.com/dotnet/coreclr/pull/25458) for more details):
{% highlight c linenos %}
X >= 1  =>  X > 0
X < 1   =>  X <= 0
X <= -1 =>  X >= 0
X > -1  =>  X >= 0
{% endhighlight %}
    
And what do you think about these ones?:
{% highlight cpp linenos %}
pow(X, 0.5)   =>  sqrt(x)
pow(X, 0.25)  =>  sqrt(sqrt(X))
pow(X, 2)     =>  X * X     ; 1 mul
pow(X, 3)     =>  X * X * X ; 2 mul
{% endhighlight %}

<p/>
<figure class="aligncenter">
	<img src="/images/instcombine/p9.png" />
</figure>

<br/>
How many `mul` are needed to perform `pow(X, 4)` or `X * X * X * X`?
<figure class="aligncenter">
	<img src="/images/instcombine/pow4.png" />
</figure>
Just 2! Just like for `pow(X, 3)` and unlike `pow(X, 3)` we don't even use the `xmm1` register.


<br/>
Modern CPUs support a special FMA instruction to perform `mul` and `add` in just one step without an intermediate rounding operation for `mul`:

{% highlight cpp linenos %}
X * Y + Z  =>  fmadd(X, Y, Z)
{% endhighlight %}
<p/>
<figure class="aligncenter">
	<img src="/images/instcombine/p11.png" />
</figure>

<br/>
Sometimes compilers are able to replace entire algorithms with just one CPU instruction, e.g.:
<figure class="aligncenter">
	<img src="/images/instcombine/p12.png" />
</figure>

### Traps for optimizations
We can't just find patterns & optimize them:
* There is a risk to break some code: there are always corner-cases, hidden side-effects. LLVM's bugzilla contains lots of InstCombine bugs.
* An expression or its parts we want to simplify might be used somewhere else. 

I borrowed a nice example for the second issue from ["Future Directions for Optimizing Compilers"](https://arxiv.org/pdf/1809.02161.pdf) article.  
Imagine we have a function:

{% highlight c linenos %}
int Foo1(int a, int b) {
   int na = -a;
   int nb = -b;
   return na + nb;
}
{% endhighlight %}

We need to perform 3 operations here: `0 - a`, `0 - b`, и `na + nb`. LLVM simplifies it to just two operations: `return -(a + b)` - what a smart move, here is the IR:

{% highlight llvm linenos %}
define dso_local i32 @_Z4Foo1ii(i32, i32) {
  %3 = add i32 %0, %1 ; a + b
  %4 = sub i32 0, %3  ; 0 - %3
  ret i32 %4
}
{% endhighlight %}

Now imagine that we need to store values of `na` and `nb` in some global variables, e.g. `x` and `y`:

{% highlight c linenos %}
int x, y;

int Foo2(int a, int b) {
   int na = -a;
   int nb = -b;
   x = na;
   y = nb;
   return na + nb;
}
{% endhighlight %}

The optimizer still recognizes the pattern and simplifies it by removing redundant (from its point of view) `0 - a` and `0 - b` operations. But we do need them! We save them to the global variables! Thus, it leads to this:
  
{% highlight llvm linenos %}
define dso_local i32 @_Z4Foo2ii(i32, i32) {
  %3 = sub nsw i32 0, %0                    ; 0 - a 
  %4 = sub nsw i32 0, %1                    ; 0 - b
  store i32 %3, i32* @x, align 4, !tbaa !2  
  store i32 %4, i32* @y, align 4, !tbaa !2
  %5 = add i32 %0, %1                       ; a + b
  %6 = sub i32 0, %5                        ; 0 - %5
  ret i32 %6
}
{% endhighlight %}

4 math operations instead of 3! The optimizer has just made our code a bit slower.
Now let's see what C# RuyJIT generates for this case:

<figure class="aligncenter">
	<img src="/images/instcombine/p10.png" />
</figure>

RuyJIT doesn't have this optimization so the code contains only 3 operations :-) C# is faster than C++! :p

### Do we really need these optimizations?
Well, you never know what the final code will look like after inlining, constant folding, copy propagation, CSE, etc.  
Also, both LLVM IR and .NET IL are not tied to a specific programming language and can't rely on quality of the IR it generates. And you can just run your app/lib with `InstCombine` pass on and off to measure the performance impact ;-).

### What about C#?
As I said earlier, peephole optimizations are very limited in C# at the moment. However, when I say "C#" I mean the most popular C# runtime - CoreCLR with RuyJIT. But there are more, including those, using LLVM as a backend: Mono (see my [tweet](https://twitter.com/EgorBo/status/1063468884257316865)), Unity Burst and LILLC - these runtimes basically use exactly the same optimizations as clang does. Unity guys are even considering [replacing C++ with C#](https://lucasmeijer.com/posts/cpp_unity/) in their internal parts. By the way, since .NET 5 will include Mono as an optional built-in runtime - you will be able to use LLVM power for such cases.
  
Back to CoreCLR - here are the peephole optimizations I managed to find in comments in `morph.cpp` (I am sure there are more):
{% highlight cpp linenos %}
*(&X)  =>  X
X % 1  => 0
X / 1  =>  X
X % Y  => X - (X / Y) * Y
X ^ -1  =>  ~x
X >= 1  =>  X > 0 
X <  1  =>  X <= 0
X + С1 == C2  =>  X == C2 - C1
((X + C1) + C2)  =>  (X + (C1 + C2))
((X + C1) + (Y + C2))  =>  ((X + Y) + (C1 + C2))
{% endhighlight %}
There are also some in `lowering.cpp` (machine-dependent ones) but in general RyuJIT obviously loses to С++ compilers here. RyuJIT just focuses more on different things and has a lot of requirements. The main one is - it should compile fast! it's called JIT after all. And it does it very well (unlike the C++ compilers - see ["Modern" C++ Lamentations](https://aras-p.info/blog/2018/12/28/Modern-C-Lamentations/)). It's also more important to de-virtualize calls, optimize out boxings, heap allocations (e.g. [Object Stack Allocation](https://github.com/dotnet/coreclr/issues/20253)). However, since RyuJIT is now supporting tiers, who knows maybe there will be a place for peephole optimizations in future in the tier1 or even a separate tier2 ;-). Maybe with some sort of DSL to declare them, just read [this](https://medium.com/@prathamesh1615/adding-peephole-optimization-to-gcc-89c329dd27b3) article where Prathamesh Kulkarni managed to declare an optimization for GCC in just a few lines of DSL:
{% highlight cpp linenos %}
(simplify
 (plus (mult (SIN @0) (SIN @0))
       (mult (COS @0) (COS @0)))
 (if (flag_unsafe_math_optimizations)
  { build_one_cst (TREE_TYPE (@0)); }))
{% endhighlight %}

for the following pattern:

{% highlight cpp linenos %}
cos^2(X) + sin^2(X) equals to 1 
{% endhighlight %}
  

### Links
* ["Future Directions for Optimizing Compilers"](https://arxiv.org/pdf/1809.02161.pdf) by Nuno P. Lopes and John Regehr  
* ["How LLVM Optimizes a Function"](https://blog.regehr.org/archives/1603) by John Regehr  
* ["The surprising cleverness of modern compilers"](https://lemire.me/blog/2016/05/23/the-surprising-cleverness-of-modern-compilers/) by Daniel Lemire
* ["Adding peephole optimization to GCC"](https://medium.com/@prathamesh1615/adding-peephole-optimization-to-gcc-89c329dd27b3) by Prathamesh Kulkarni
* [C++, C# and Unity](https://lucasmeijer.com/posts/cpp_unity/) by Lucas Meijer
* ["Modern" C++ Lamentations](https://aras-p.info/blog/2018/12/28/Modern-C-Lamentations/) by Aras Pranckevičius
