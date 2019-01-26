---
layout: post
title: GETTING STARTED WITH SIMD INTRINSICS
date: 2019-01-24 21:01:00
description: This post enumerates some of the considerations for writing SIMD intrinsics in C++.
---

## What is SIMD?
<br>
SIMD has a long and storied history in DSP. It stands for `single instruction, multiple data`
and started to be used in DSPs in the 1980s to speed up processor hungry operations like
convolution, the FFT, and FIR filtering. The basic idea is that a single specialized register
can perform an arithmetic operation on multiple data in a single cycle. The width of these 
special SIMD registers is hardware and instruction-set dependent. Common SIMD register widths
are 2-way, 4-way, and 8-way. (Ie, 2, 4, or 8 floats could be processed in a single cycle.)

In essence, SIMD creates another "tier" of parallelism in software design. While multithreading
can be used for parallelism at the *task level*, SIMD allows for parallelism at the *instruction level*. 
If each parallel "task" is also configured to use SIMD instructions, you'll really be playing with power!

Like writing code with parallel tasks, writing parallel-instruction friendly code requires 
special design considerations to be effective. Luckily, some of the significant design changes
used to make SIMD-friendly code also make your code more cache friendly.

## How do I use it?
<br>
# Lazy Approach: Auto-Vectorization
<br>
One can take advantage of SIMD in many different ways. Sometimes you even get it for free
when your compiler performs *auto-vectorization*. So if you write code that happens to be
"embarrassingly parallel" at the instruction level, your compiler may actually generate
SIMD instructions for you. This is a *really* nice concept when you aren't actually relying on 
SIMD. Small, seemingly inconsequential code changes can cause the compiler to bail on
trying to generate SIMD and there's really no way to tell unless you inspect the assembly
output each time you compile after a small change. And at that point, you might as well
be more intentional about writing your algo with SIMD.
<br>
# Usual Best Approach: Wrapper Libraries and Code Generation
<br>
Another approach to using SIMD is using SIMD wrapper libraries. This is great for more
consistent, broad strokes, intentional SIMD code. Wrapper libraries can range from 
math libraries that just use SIMD under-the-hood to libraries that swap out your primitive
data-types (ie, `float`) with SIMD data types (`vfloat4`) that you can use to write a 
"scalar-looking" algorithm that will consistently compile to SIMD instructions.
This is great for broad-strokes speedup in an application. Some efficiency may be lost
by the wrapper library doing expensive SIMD loads/stores behind your back more often than
it needs to or dealing with any unaligned data you throw at it, but overall this is a 
great practical approach to SIMD. 

Worth also mentioning are SIMD-code-generation tools such as `ispc`. These tools are pretty
neat -- you write C-like code and the tools output SIMD instructions from them. Could be 
useful, though I don't know much about them. 
<br>
# Most Flexible Approach: Intrinsics (or Inline Assembly)
<br>
Here is the one I plan to discuss. SIMD boils down to platform specific code. At the end of
the day, the most predictable and powerful way to write SIMD code is using those
platform specific instructions or compiler intrinsics (thin wrappers over inline assembly).
And since inline assembly is becoming deprecated in some compilers (*cough* MSVC), intrinsics 
seem to be the way to go.

With that said, intrinsics are imperfect in several ways:
- They obfuscate the heck out of your code (as you will see below)
- They make your code platform specific, so you'll likely need to reach for `#ifdef`s or template code
- They defeat many optimizations the compiler otherwise might do

And with regards to that last point... **ALWAYS** profile your code when writing SIMD intrinsics.
Since you are opting out of compiler optimizations, your SIMD code can actually be slower than
scalar code that "does the same thing" if you don't make the right design choices. (But if you
do make the correct decisions, it can be much faster.) Always be on the lookout for both performance
and output regressions. 

In short, intrinsics are the tool you should reach for when you may otherwise be considering 
inline assembly. In audio, this is usually the innermost loop of an expensive DSP algorithm in 
a code-base where SIMD wrappers were used everywhere else.
<br>
# If you decide to use SIMD intrinsics, where do you start?
<br>
Raw intrinsics at the application level make sense when you have something like this:

{% highlight c++ %}
static void convolve_naive(
                     float *inSig, size_t M,
                     float *inKernel, size_t N,
                     float *outSig)
{
    const size_t outLen = M + N - 1;
    for (size_t i = 0; i < outLen; i+=4)
    {
        float accumulator = 0;
        float prod;
        float sig;
        float kernel;
        for (size_t j = 0; j < N; ++j)
        {
            sig = inSig[i+j];
            kernel = inKernel[N - 1 - j];
            prod = sig * kernel;
            accumulator = accumulator + prod;
        }
        outSig[i] = accumulator;
    }
}
{% endhighlight %}

Convolution was one of the original motivations for writing SIMD code and 
it remains a nice, clean example. Keep in mind that the actual code that you
see (a nested loop with arithmetic) is not the only part of the equation that
makes this a good candidate for SIMD. A key factor is the usual high-throughput
requirement of real-world convolution algorithms.

When you start off with scalar code like this, it helps to take an 
iterative approach.

<br>
# First Iteration: A "Naive SIMD" Approach
<br>
Now before you start hacking the intrinsics into the algorithm, make sure to
create a backup of your code. Making a branch in your VCS is a good idea,
and you also may find it convenient to `#ifdef` your old code above the new 
code to reference and do quick checks for output regressions.

Once you have that, it's time to just rework your inner loop to look “SIMD-like".
At this point the performance is not as important as coaxing out any caveats
you may have not noticed at first glance that give you compiler errors
or cause you to to reorganize buffers, etc.

{% highlight c++ %}
static void convolve_simd_unaligned( float *inSig, size_t M,
                                     float *inKernel, size_t N,
                                     float *outSig)
{
    const size_t outLen = M + N - 1;
    // Preprocess the kernel:
    // reverse the kernel pre-loop and repeat each value across a 4-vector
    alignas(16) __m128 inKernelSIMD[N];
    for(int i=0; i<N; i++)
    {
        inKernelSIMD[i] = _mm_set1_ps(inKernel[N - i - 1]);
    }
    
    // pre-loop vars
    alignas(16) __m128 simd_sig;
    alignas(16) __m128 prod;
    alignas(16) __m128 accumulator;
    for (size_t i = 0; i < outLen; i+=4)
    {
        accumulator = _mm_setzero_ps();
        for (size_t j = 0; j < N; ++j)
        {
            simd_sig = _mm_loadu_ps(&inSig[i + j]);
            prod = _mm_mul_ps(simd_sig, inKernelSIMD[j]);
            accumulator = _mm_add_ps(accumulator, prod);
        }
        _mm_storeu_ps(&outSig[i], accumulator);
    }
}
{% endhighlight %}

In this case, the inner loop is made to look as much like the original arithmetic
as possible. An unaligned load of the input is done on each inner loop, which is
a big inefficiency, but don't worry about that yet. Just try and do something that 
looks as similar to your original algo as possible and test for regressions!

This code also starts to do some of the re-jiggering of the buffers necessary when 
converting to SIMD. The filter kernel is reversed and stored in SIMD registers before 
the actual convolution takes place.
<br>
# Pass 2
<br>
Restructure your data to load aligned outside of the loop
Rework the inner loop to take advantage of your load restructuring
PROFILE

{% highlight c++ %}
static void convolve_simd( float *inSig, size_t M,
                           float *inKernel, size_t N,
                           float *outSig)
{
    const size_t outLen = M + N - 1;
    
    // preprocess the input
    // store offset versions of the input signal to allow for aligned loads
    for(int ii = 0; ii < 4; ii++)
    {
        int j = 0;
        for (size_t i = 0; i < M; i+=4)
        {
            inSignalSIMD[ii][j++] = _mm_set_ps(inSig[i+0+ii], 
                                               inSig[i+1+ii], 
                                               inSig[i+2+ii], 
                                               inSig[i+3+ii]);
        }
    }

    // Preprocess the kernel:
    // reverse the kernel pre-loop and repeat each value across a 4-vector
    alignas(16) __m128 inKernelSIMD[N];
    for(int i=0; i<N; i++)
    {
        inKernelSIMD[i] = _mm_set1_ps(inKernel[N - i - 1]);
    }
    
    // pre-loop vars
    alignas(16) __m128 accumulator;
    for (size_t i = 0; i < outLen; i+=4)
    {
        accumulator = _mm_setzero_ps();
        for (size_t j = 0; j < N; ++j)
        {
            // indexing gets real complicated!
            int index = i/4 + (int)(j*0.25); 
            accumulator = _mm_add_ps(accumulator, 
                                     _mm_mul_ps(inSignalSIMD[j%4][index], 
                                     inKernelSIMD[j]));
        }
        _mm_storeu_ps(&outSig[i], accumulator);
    }
}
{% endhighlight %}

<br>
## Other Design Considerations
<br>

<br>
# Operate on aligned data 
<br>

<br>
# Minimize loads/stores 
<br>
( call _mm_store and _mm_load the least number of times )

<br>
# Restructure "Array of Structures" to "Structure of Arrays"
<br>
- this won't feel natural in a c-like language
- it is the data oriented design approach
- makes code more cache friendly and more SIMD friendly
  - no longer "fighting" the simd registers

