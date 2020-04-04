---
layout: post
date: 2020-04-04 10:00 AM
title: "How RyuJIT inlines a function (heuristics)"
description: 
comments: false
category: 
-
tags:
- C#
- RyuJIT
- inlining
- optimizations
---

Inlining is one of the most important optimizations. It eliminates a `call` overhead and exposes more opportunities for other optimizations (e.g. constant folding) and sometimes even makes callers smaller. Most people I've asked think that the jit inlines only small methods under a certain IL size threshold, e.g. < 32 bytes of IL and simply gives up on bigger functions.
So, I decided to write this blog post and came up with a perfect example to cover several heuristics at once:
<figure>
	<img src="/images/inline/raw.png" />
</figure>
Take a guess - is this `Volume` constructor inlineable? Obviously, it's not, it's just too big. Especially because `throw new`
is quite expensive and emits a lot of native code we don't want to see in our callers. Let's check codegen via Disasmo:

<figure>
	<img src="/images/inline/codegen1.png" />
</figure>

It's inlined! And all the exceptions are completely eliminated! At this point you might think "Ah, ok, Jit is
smart enough to make a full analyze of all basic-blocks/branches/locals and calculate the accurate control flow for constant arguments" or
"Jit inlines EVERYTHING, runs the full cycle of optimizations and then decides if it's profitable to inline or not"

Well... no, it's not possible to do in a reasonable time. Jit only makes a few guesses (or observations) and estimates final native code size and performance impact. There are positive and negative observations, positive ones increase a special benefit multiplier, the bigger the multiplier - the more code we can inline. Negative observations might limit the benefit multiplier or just completely abort the whole optimization. So what observations did jit make for `Volume..ctor` inside `Test`?

<figure>
	<img src="/images/inline/heuristics.png" />
</figure>

<br/>

We can see it in Disasmo (JitDump log):

<figure>
	<img src="/images/inline/bm.png" />
</figure>

All these simple observations set our multiplier to 11.5 and helped us to satisfy the inliner. E.g. the fact that we end up testing (`==`) constant arg `'B'` (promotable struct) with another constant (e.g. `'C'`) gives us confidence that one of the branches will
be optimized out and the native size will be smaller than estimated. Also, the fact that the method (constructor) is called inside a loop tells tje jit that it should try harder, etc.


<br/>
The Jit also uses these and other observations to estimate the final codegen size and its performance impact via magic coefficients (ML?), see
[EstimateCodeSize()](https://github.com/dotnet/runtime/blob/a605729eee65344b4c63fb036a35405abcc1de31/src/coreclr/src/jit/inlinepolicy.cpp#L1681-L1737) and [EstimatePerformanceImpact()](https://github.com/dotnet/runtime/blob/a605729eee65344b4c63fb036a35405abcc1de31/src/coreclr/src/jit/inlinepolicy.cpp#L1749-L1766).

Btw, did you notice this trick?:
{% highlight csharp linenos %}
if ((value - 'A') > ('Z' - 'A'))
{% endhighlight %}

it's an optimized version of:
{% highlight csharp linenos %}
if (value < 'A' || value > 'Z')
{% endhighlight %}

Both are semantically the same but the latter consists of two expressions and the former is a single expression. It turns out the jit also has a limited amount of basic-blocks in a method it can inline and if it's > 5 (see [here](https://github.com/dotnet/runtime/blob/a605729eee65344b4c63fb036a35405abcc1de31/src/coreclr/src/jit/inlinepolicy.cpp#L492-L495)) -- no matter how big is the multiplier, it prints `too many basic-blocks` and aborts. That's why I had to apply this `'Z' - 'A'` trick. And I guess it'd be nice if both Roslyn and RyuJIT could automatically could do it for me:

**Roslyn** issue: [https://github.com/dotnet/runtime/issues/13347](https://github.com/dotnet/runtime/issues/13347)<br/>
**RyuJIT** PR (my poor attempt): [https://github.com/dotnet/coreclr/pull/27480](https://github.com/dotnet/coreclr/pull/27480)<br/><br/> 
And that's why I think it makes sense to do the optimization in Roslyn:

<figure>
	<img src="/images/inline/roslyn.png"/>
</figure>

#### Inlining and virtual methods
Obviously, we can't inline virtual methods so that's why RyuJIT needs more "devirtualization" optimizations (it already has some).

#### Inlining and "throw new"
If a method never returns - it's most likely just a throw helper and should not be inlined (and the call should be marked as 'rarely executed'). You can find a lot of ThrowHelpers in the BCL - it's one of the first things they do for hot methods.

#### Inlining and [AggressiveInlining]
You basically __strongly__ advice the jit to inline a method but it should be used carefully and most of the "I've added an AggressiveInlining here" PRs in BCL are simply rejected because of two reasons:
1) Inlining can negatively affect native code size (e.g. it optimizes for constant input and regresses other cases)
2) Inlining generates a lot of temp variables and the amount of these variables can easily hit the hard-coded limit of variables JIT can track (512) and you'll see a lot of very slow spills, a perfect example is this tweet: [https://twitter.com/damageboy/status/1238724089403097088](https://twitter.com/damageboy/status/1238724089403097088) or this issue [https://github.com/dotnet/runtime/issues/13423#issuecomment-531854959](https://github.com/dotnet/runtime/issues/13423#issuecomment-531854959)

#### Inlining and DynamicMethod
It's not currently supported, see this issue: [https://github.com/dotnet/runtime/issues/34500](https://github.com/dotnet/runtime/issues/34500)<br/>
But if you think this can significantly optimize your code leave a comment there.

#### My attempt to make a heuristic
I tried to extend the existing heuristics in order to help the following case:
<figure>
	<img src="/images/inline/strlen.png"/>
</figure>
A few months ago I [added](https://github.com/dotnet/runtime/pull/1378) an optimization to RyuJIT for `"const str".Length` to be replaced with a constant. So here ^ if we inline
`Validate` into `Test` we'll have `if ("hello".Length > 10)` and it will be optimized to just `if (5 > 10)` and the whole
branch including `throw new` will be eliminated. But unfortunately in this case JIT refuses to inline:
<figure>
	<img src="/images/inline/strlen2.png"/>
</figure>
And the main problem here is the fact that Jit doesn't know we are going optimize `get_Length` and the inliner should aslo 
have a sort of `constant string feeds get_Length, multiplier is increased to ..` observation. Here is my attempt to add it [https://github.com/EgorBo/runtime-1/commit/3810c2146f7db9deb9f75f486cd2ccb3cc50a620](https://github.com/EgorBo/runtime-1/commit/3810c2146f7db9deb9f75f486cd2ccb3cc50a620): The only problem here we don't have time to resolve all `callvirt` CIL instructions to find out if it's `System.String.get_Length` or not (see Andy's [comment](https://github.com/dotnet/runtime/issues/33338#issuecomment-596153086)).

There are a lot of other limitations, you can find some of them [here](https://github.com/dotnet/runtime/blob/master/src/coreclr/src/jit/inline.def). Also, I recommend you to read Andy Ayers's [thoughts](https://github.com/dotnet/runtime/issues/34286#issuecomment-606186300) about inliner's design in general and his ["Some Notes on Using Machine Learning to Develop Inlining Heuristics"](https://github.com/AndyAyersMS/PerformanceExplorer/blob/master/notes/notes-aug-2016.md) article.
