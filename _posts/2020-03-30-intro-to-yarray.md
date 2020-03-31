---
layout: post
title: "Introduction to Yarray"
date: 2020-03-30
---

## Introduction and motivation
Array programming languages differentiate themselves from other languages paradigms by natively supporting array operations for an intrinsic array type. Most languages support primitive scalar types such as floats and integers, and array languages simply extend this set of primitives to include arrays. In addition, array programming languages often optimize operations using this array type in such a way that resulting programs are both more ergonomic and efficient than their equivalents written in other language paradigms. For this reason, common applications of array programming languages involve data heavy computation such as scientific programming.

Operations on arrays are characterized by their need to update many elements simultaneously and in the same way. For example, consider the following addition of two single-dimension arrays in an array language format:
```
// Initialize an array, A, with 50 evenly spaced values from 0 to 100(inclusive)
A = span(0, 100, 50);
// Initialize an array, B, with 50 evenly spaced values from 150 to to 199(inclusive)
B = span(50, 200, 50);
// Initialize an array, C, which contains a pairwise sum of both A amd B
C = A + B;
```
under a non-array language paradigm, the same addition would require explicit looping over members over the `A` and `B` arrays:
```
// Initialize arrays A and B
A = ...;
B = ...;
// Declare array C
C = ...;
// Perform addition
for (i = 0; i < 50; i++)
{
  C[i] = A[i] + B[i];
}
```
As the array grows in size and dimensions, the distinct points needed to iterate over grows quickly. To be more exact, iterating over an `n` dimensional array with each dimension of size `e` has a time-complexity of `e^n` - exponential in the number of dimensions! These operations can quickly become the bottleneck for a program manipulating large, higher-dimensional arrays. One effective strategy for mitigating this bottleneck is to perform these operations in parallel.

This project leverages a specialty compiler, PPCG, which is able to transform a sequential C program into cooperating C and CUDA implementations. The PPCG compiler relies on compiler directives to identify loops which are amenable to being parallelized on a GPU. The goal of this project is to leverage the power of PPCG with Yorick programs which are natively run sequentially. A direct translation between the Yorick language and C is necessary to provide PPCG with supported input, and the resulting translation is then split into sequential C and parallel CUDA implementations and run on a heterogeneous system. Ultimately, a performance improvement is expected from the cooperating C and CUDA implementation over the initial Yorick implementation.

## The Model
The *Yorick Array* model, named `yarray`, is an optimized C implementation of an array type based on the [NumPy `ndarray`](https://arxiv.org/pdf/1102.1523.pdf). The `yarray` model strives to facilitate array programming conventions in C with the secondary goal of doing so in an optimized way. As is true for the `ndarray`, operations supported by `yarray` are optimized for common array manipulations but the `yarray` model supports additional, unique optimizations with the goal of GPU parallelization in mind. The PPCG compiler, as will be discussed further in a separate post, relies on a particular type of analysis that is restricted in its ability to reason about a program. Certain programming semantics (eg: indirect array indexing) can prevent optimization under the PPCG model. The `yarray` model takes these restrictions into consideration and attempts to translate Yorick programs in such a way that compiled PPCG programs are unhindered by valid semantics in both Yorick and C.

An in-depth discussion of the `yarray` model can be found [here]({{ site.baseurl }}{% link _posts/2020-03-31-the-yarray-model.md %}).
