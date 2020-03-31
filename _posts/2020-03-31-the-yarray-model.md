---
layout: post
title: "The Yarray Model"
date: 2020-03-31
---

## The Object
The `yarray` object is, at its core, a robust tagged union (also known as a variant type). It maintains the state of a particular array and preserves the costly interpretation of multiple dimensions.
```
typedef enum {INT, FLOAT, LONG, DOUBLE} dataType;
typedef struct yarr yarr;

// Complex array type implemented in C
// Modeled after `ndarray` from Numpy
struct yarr {
  int dims;
  // Widths hold the number of elements in the next dimension
  int *widths;
  // Strides are organized from outer dimensions to inner dimensions
  // eg: arr[5][4][3] -> strides{12, 3, 1}
  // where each stride denotes how many elements are in the nested dimensions
  // To access arr[2][1][0] ->
  //   *arr + ((2*strides[0] + 1*strides[1] + 0*strides[2]) * sizeof(int))
  int *strides;
  dataType tag;

  union data {
    int    *idata;
    float  *fdata;
    long   *ldata;
    double *ddata;
  } data;
};
```

In typical C programs, one might see the allocation of a two-dimensional array of integers as follows:
```
int rows = ...;
int cols = ...;

int **matrix = malloc(rows * sizeof(int *);
for (int r = 0; r < cols; r++)
{
    matrix[i] = malloc(cols * sizeof(int));
}
```
Under this model, underlying memory for `matrix` may be scattered across the available address space of the program. There is no guarantee that a given row, `matrix[j]` is physically adjacent in memory to its subsequent row, `matrix[j+1]`. The advantage of this potentially fragmented setup is that the maximum amount of contiguous memory needed is lower, as for a large matrix, `rows * cols` contiguous blocks of memory may not be always be available. The downside, however, is that significant operation optimizations can be realized when the underlying representation of the multi-dimensional array is one-dimensional.

To illustrate this point, consider a program which tries to reform a two-dimensional array from having the shape of `50x50` to `25x100`. If the underlying memory of the matrix is not contiguous, this would require resizing both dimensions even though the total size of the matrix remains unchanged. Utilizing a model which maintains a contiguous representation, the one-dimensional array of size `2500` can be reinterpreted from being split into two dimensions of size `50` and `50` to two dimensions of size `25` and `100`.

### Stride

Within the `yarray` model, the `strides` array is responsible for managing this interpretation. A stride is simply the number of elements that must be skipped over in order to get to the adjacent member in that same dimension. For example, with a three-dimensional array of shape `100x50x10` (dimensions read left -> right, outermost -> innermost), to get from the first outermost element to the second, `50 * 10` elements will have to be skipped. Pictorally, with a smaller three dimensional array with shape `2x3x4` seen as:
```
[
  [
    a, b, c, d
  ],
  [
    e, f, g, h
  ],
  [
    i, j, k, l
  ]
s],
[
  [
    m, n, o, p
  ],
  [
    q, r, s, t
  ],
  [
    u, v, w, x
  ],
]
```
with strides represented as `[12][4][1]`.  
Moving from element `a` to element `m`, `3 * 4` elements need to be skipped as there are three middle-dimensions between `a` and `m` (including the middle-dimensions containing `a`) and each of these middle dimensions contains `4` elements. Thus, it can be seen that `three_dim_array[0][0][0]` and `three_dim_matrix[1][0][0]` are `12` elements apart - this is reflected in our representation of strides!

Going back to our discussion about ease of reforming under a contiguous memory model, if our initial `matrix` is of shape `50x50`, its strides can be represented as `[50][1]`. Reforming `matrix` to be of shape `25x100` would then be written as `[100][1]`. Initially, moving between outermost elements would require skipping over `50` elements but the reformed array requires skipping over `100` elements to get to subsequent outermost dimensions.

At this point, the observant reader may notice that the inner most dimension will always have a stride of 1 as there are no intermediate dimensions separating these elements! Generally, stride can be calculated by taking the product of subsequent dimensions' widths. Mathematically speaking for a given dimension, `j`, in an `n` dimensional array where $$j \in 0..n$$),  the stride of dimension `j` is: $$\prod_{k=j+1}^{n} width[k]$$.

### Widths
This brings us to our next field of the `yarray` model, `widths`. `widths` may be slightly more intuitive than strides as it simply describes, for a given dimension, the number of elements *of* the subsequent dimension it contains. To give an example, our `25x100` matrix has widths which can be written as `[25][100]`. The outermost dimension holds 25 elements each of which represent the start of the innermost dimension. Another way to visualize widths is as the number of strides that can be taken such that each stride accesses an element without advancing a higher dimension. In our `2x3x4` array starting from `three_dim_array[0][0][0]`, we can take **2** steps of size `12` along the outermost dimension before exceeding the bounds of the array, **3** steps along the middle dimension before moving from `three_dim_array[0][0][0]` to `three_dim_array[1][0][0]`, and **4** steps along the innermost dimensino before moving from `three_dim_array[0][0][0]` to `three_dim_array[0][1[0]`. This description of widths is also seen when describing the *shape* of an array.

### Tag and Data
As was stated at the start of this post, the `yarray` model is, at its core, a tagged union. For those new to tagged unions or variant types, it is simply a data-structure which can hold *one* of a finite number of values at a time, and is aware of which value it is holding onto at any given time. In the C language, a union is used to encapsulate a choice between possibly complex types. In the `yarray` model, the `data` field is able to hold a pointer to `int`s, `float`s, `long`s, and `double`s. These pointers all represent contiguous blocks of memory which hold the actual elements our `yarray` is responsible for. Once a choice is made, however, the `data` field alone does not tell us much about which type of pointer was selected. This is where the 'tag' in tagged union comes in. Whenever a choice of what kind of `data` is to be stored, the `tag` field is set accordingly. Thus, we are able to use a single `yarray` object to hold multiple types of data! It is important to note that a struct (a sum type) cannot hold more than one choice at a time. It is not valid for a single instance of `yarray` to hold a pointer to both integers and doubles.

### Overhead
With the concept of `strides`, `widths`, and `data`, we are nearly complete with our array type. The only remaining elements in the initial definition of `yarray` are trivial or peripheral. We keep track of the total number of dimensions in `dims`, and use an `enum` to represent our tags which we call `tag`.

### Wrapup
With this introduction the `yarray` model, we are one step closer to making C suitable for supporting array programming conventions. The next step is to add support for common [array operations]({{ site.baseurl }}{% link _posts/2020-03-31-working-with-yarray.md %}).
