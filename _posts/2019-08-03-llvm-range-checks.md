---
layout: post
date: 2019-08-03 10:00 AM
title: "Smart LLVM #1: Optimizing range checks"
description: 
comments: false
category: 
-
tags:
- llvm
- C++
- C#
- optimizations
---

Sometimes I explore LLVM sources and play with godbolt.org in order to find some interesting optimizations (not only the peephole ones) so I think I'll post some here in my blog from time to time. Also, if an optimization is simple enough I try to implement it in RuyJIT, e.g.:

* [dotnet/coreclr#25912](https://github.com/dotnet/coreclr/pull/25912) Remove bound checks when index is Byte and array.Length >= 256
* [dotnet/coreclr#25744](https://github.com/dotnet/coreclr/pull/25744) Transform X % C == 0 to X & (C-1) == 0
* [dotnet/coreclr#25856](https://github.com/dotnet/coreclr/pull/25856) Recognize FMA patterns (x*y+z) 
* [dotnet/coreclr#24584](https://github.com/dotnet/coreclr/pull/24584) Replace (val / 2) with (val * 0.5)
* [dotnet/coreclr#25458](https://github.com/dotnet/coreclr/pull/25458) Optimize u>=1 to u!=0 and u<1 to u==0  
As you can see the CoreCLR developers are quite friendly and help a lot to understand things better.

Today I am going to share a nice LLVM trick to optimize some common range checks.  
So, let's say we have a function that checks if a char belongs to a list of reserved chars:  
(I actually copy-pasted it from CoreFX)
{% highlight csharp linenos %}
bool IsReservedCharacter(char character) // uint16_t
{
    return character == ';'
        || character == '/'
        || character == ':'
        || character == '@'
        || character == '&'
        || character == '='
        || character == '+'
        || character == '$'
        || character == ',';
}
{% endhighlight %}
Now let's compare outputs for RuyJIT and LLVM:
<!--more-->

<figure class="alignleft">
{% highlight nasm linenos %}
; C# RuyJIT
  movzx    rax, cx
  cmp      eax, 59
  je       SHORT G_IG04
  cmp      eax, 47
  je       SHORT G_IG04
  cmp      eax, 58
  je       SHORT G_IG04
  cmp      eax, 64
  je       SHORT G_IG04
  cmp      eax, 38
  je       SHORT G_IG04
  cmp      eax, 61
  je       SHORT G_IG04
  cmp      eax, 43
  je       SHORT G_IG04
  cmp      eax, 36
  je       SHORT G_IG04 
  cmp      eax, 44
  sete     al
  movzx    rax, al
G_IG03:
  ret      
G_IG04:
  mov      eax, 1
G_IG05:
  ret
{% endhighlight %}
</figure>

<figure class="alignleft">
{% highlight nasm linenos %}
; LLVM
  add edi, -36
  cmp di, 28
  ja .LBB0_2
  mov al, 1
  movzx ecx, di
  mov edx, 314575237    
  bt rdx, rcx
  jae .LBB0_2
  ret
.LBB0_2:
  xor eax, eax
  ret
{% endhighlight %}
</figure>
<figure class="aligncenter">
</figure>

As you can see C# generated a pretty simple set of 9 cmp + jumps for each logical OR. LLVM, at the same time, generated something strange with magic numbers and just two branches. Let's try to convert (disassemble) LLVM's output to C#:
{% highlight csharp linenos %}
bool IsReservedCharacter(char c)
{
    c = (char)(c - 36);
    if (c > 28)
        return false;
    return ((314575237 >> c) & 1) == 1;
}
{% endhighlight %}

so insted of 9 cmp we have `add, cmp, shr, and`
Let me explain the magic constants.  
First, we need to convert chars to their ASCII numbers:
{% highlight csharp linenos %}
';' '/' ':' '@' '&' '=' '+' '$' ','
59  47  58  64  38  61  43  36  44
{% endhighlight %}

The biggest is `@` (64) and the smallest is `$` (36). So, the range starts from 36 and the length is `64 - 36 = 28`. Thus the first `if` simply ignores all values outside of `[36..64]` range. Here is how I explained the first two magic numbers. Now it's `314575237`s turn:

Since the range is known and the length is 28 which easily fits into a 32/64bit CPU register we can encode it to a special bit-map (a set of 0 and 1) - a 32/64 bit integer (depending on a platform).
Here is how it's done:
{% highlight csharp linenos %}
long bitmap = 0;
foreach (char c in new [] { ';', '/', ':', '@', '&', '=', '+', '$', ',' })
    bitmap |= 1L << c - 36;
{% endhighlight %}
So, for each char we push (shift) `1` to the left according to `c - 36` value (as you remember 36 stands for `$` so its index will be zero - on the right)  
and our bitmap becomes:
{% highlight csharp linenos %}
  
00010010110000000000100110000101 = 314575237
   |  | ||          |  ||    | |
   @  = ;:          /  ,+    & $
  
{% endhighlight %}

Now when we do `314575237 >> (c - 36)` we either get `1` (symbol is one of the reserved) or `0` (doesn't belong to the set)

Let's benchmark it! I have a random string here and I need to calculate how many symbols are reserved:
{% highlight csharp linenos %}
string str = "Some link https://github.com/dotnet/coreclr/issues/12477, some@mail.com.";
int count = 0;
foreach (char c in str)
    if (IsReservedCharacter(c))
        count++;
{% endhighlight %}

The results are:
{% highlight csharp linenos %}
|                      Method |      Mean | Ratio |
|---------------------------- |----------:|------:|
| CountReserverCharacters_old |  197.6 ns |  1.43 |
| CountReserverCharacters_new |  138.4 ns |  1.00 |
{% endhighlight %}

The improved version is **43%** faster! (Core i7 8700K)
  
Feature request for RuyJIT [dotnet/coreclr#12477](https://github.com/dotnet/coreclr/issues/12477)

LLVM opt: [godbolt.org](https://godbolt.org/z/2B-00V) (convert to switch)  
LLVM llc: [godbolt.org](https://godbolt.org/z/JSBhgh) (DAG*)  
