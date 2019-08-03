---
layout: post
date: 2019-06-25 11:32 PM
title: "Smart LLVM: Optimizing range checks"
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

Sometimes I explore LLVM sources and play with godbolt.org in order to find interesting optimizations (not only the peephole ones) so I think I'll post some here in my blog time to time. 

So let's say we have a function that checks if a char belongs to a list of reserved chars:  
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
Now let's compare RuyJIT and LLVM:
<!--more-->

<figure class="alignleft">
{% highlight nasm linenos %}
; C# RuyJIT
  movzx    rax, cx
  cmp      eax, 59
  je       SHORT G_M40423_IG04
  cmp      eax, 47
  je       SHORT G_M40423_IG04
  cmp      eax, 58
  je       SHORT G_M40423_IG04
  cmp      eax, 64
  je       SHORT G_M40423_IG04
  cmp      eax, 38
  je       SHORT G_M40423_IG04
  cmp      eax, 61
  je       SHORT G_M40423_IG04
  cmp      eax, 43
  je       SHORT G_M40423_IG04
  cmp      eax, 36
  je       SHORT G_M40423_IG04  
  cmp      eax, 44
  sete     al
  movzx    rax, al
G_M40423_IG03:
  ret      
G_M40423_IG04:
  mov      eax, 1
G_M40423_IG05:
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

As you can see C# generated a pretty simple set of 9 cmp + jumps. LLVM generated something strange with magic numbers and just two branches. Let's try to convert (disassemble) LLVM's output to C#:
{% highlight csharp linenos %}
bool IsReservedCharacter(char c)
{
    c = (char)(c - 36);
    if (c > 28) return false;
    return ((314575237 >> c) & 1) == 1;
}
{% endhighlight %}

so insted of 9 cmp we have `add, cmp, shr, and`
Let me explain the magic constants.  
First we need to convert chars to their ASCII numbers:
{% highlight csharp linenos %}
';' '/' ':' '@' '&' '=' '+' '$' ','
59  47  58  64  38  61  43  36  44
{% endhighlight %}

So the biggest is `@` (64) and the smallest is `$` (36). So the range starts from 36 and the length is `64 - 36 = 28`. Here is how I explained the first two magic numbers. Now it's `314575237`s turn:

Since the range is known and the length=28 easily fits into a 32/64bit CPU register we can encode it to a special bit-map (a set of 0 and 1) - a 32/64 bit integer (depending on a platform).
Here is how it's done:
{% highlight csharp linenos %}
long bitmap = 0;
foreach (char c in new [] { ';','/',':','@','&','=','+','$',',' })
    bitmap |= 1L << c - 36;
{% endhighlight %}
and our bitmap becomes:
{% highlight csharp linenos %}
  
00010010110000000000100110000101 = 314575237
   |  | ||          |  ||    | |
   @  = ;:          /  ,+    & $
  
{% endhighlight %}

Let's benchmark it! I have a random string and I need to calculate how many symbols are reserved:
{% highlight csharp linenos %}
string str = "123_)()&*(^^@@$%!*&*()@*(%(+)@_+*(&^%$?><LOP{end";
int count = 0;
foreach (char c in str)
    if (IsReservedCharacter(c))
        count++;
{% endhighlight %}

The results are:
{% highlight csharp linenos %}
|                      Method |      Mean |     Error |    StdDev | Ratio |
|---------------------------- |----------:|----------:|----------:|------:|
| CountReserverCharacters_old | 100.83 ns | 0.0422 ns | 0.0352 ns |  1.28 |
| CountReserverCharacters_new |  78.92 ns | 0.0735 ns | 0.0652 ns |  1.00 |
{% endhighlight %}

So the improved version is **28%** faster!  
  
Feature request for RuyJIT https://github.com/dotnet/coreclr/issues/12477

LLVM opt: https://godbolt.org/z/2B-00V (convert to switch)  
LLVM llc: https://godbolt.org/z/JSBhgh (DAG*)  
