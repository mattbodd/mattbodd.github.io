## Overview
The `update_C_array` function is responsible for updating a subset of values within a specified `yarray`. Subset, in this case, is used broadly to mean at least 1 element and at most the total number of elements in the entire `yarray`. A subset can span multiple dimensions and disparate ranges of values within various dimensions. The behavior being mimicked with `update_C_array` is expressed in Yorick syntax within the following style  
```
// NOTE: Shape is read left->right, outer-most dimension->inner-most dimension
// Initialize an array with shape (5,4,3) in which all values are initialized
// to 0
// NOTE: Yorick array initialization uses reversed shape specification
a = array(0, 3,4,5);
// Semtantic equivalent of `update_C_array`
// Read from right->left, outer-most dimension->inner-most dimension:
// Update all values in the innermost dimension of the second and third elements
// in the middle dimension which are located in the second element of the outermost dimension
a(:,2:3,2) = 1;
```
With C style indexing, the last statement in the above code snippet can be represented as:
```
// NOTE: C indexing is 0-based while Yorick indexing is 1-based
for (int m = 1; m <= 2; m++) {
  // `inner_width` is 3 for
  for (int i = 0; i < inner_width; i++) {
    a[1][m][i] = 1;
  }
}
```

## Definition
```
void update_C_array(yarr *y, double fill_val, int bounds_size, int *bounds)
```

### Input
`yarr *y`:         The `yarray` to update
`double fill_val`: The value to update specific elements to
`int bounds_size`: The size of the bounds array
`int *bounds`:     The bounds for each dimension (a two tuple of `lower` and `upper` bounds)

### Output
Side effect of an updated `y` `yarray`

## Discussion
At the heart of this function is a method for counting in terms of some arbitrary base. For an array with some number of dimensions, each dimension can be thought of as a 'place' within this unique number system. To draw the parallel between a familiar 10 based number system, any given place can only support 10 unique numbers (0..9) before repeating and incrementing the next highest place for one. As an example, `18, 19, 20, 21`. After 19, the ones place restarted its count at 0 but not before updating the tens place to one more than it was before. In the example of an array with shape `2x3x6`, think of the inner most dimension as the 'ones' place with each higher dimension being a more significant place. As we traverse the inner most dimension, after 6 steps, we start over at the 0th index but this time in the context of the next element in the middle dimension! This is another interpretation of the idea of stride. The `2x3x6` array has strides of `18x6x1` which intuitively tells us that, for every one step along the middle dimension, 6 steps are taken along the inner most dimension. For the outermost dimension, every one step is equivalent to taking 18 steps along the inner most dimension.

This unique form of counting is not as simple as reusing the strides for a given array as an operations may only want to traverse a subset of the elements of a given dimension. For this reason, before we can begin counting, we must determine the 'bounds' of each place.

The first computation involves computing widths. Width in this case is simply the number of elements being iterated over for a given dimension. Width is, intuitively, the difference between the `upper` and `lower` bounds for a given dimension. One quirk is that in Yorick, a dimension can be 'spectated' over. Consider the following Yorick snippet:
```
a(:,2:3)
```
Here, we are updating *all* elements in the 2nd and 3rd rows of this matrix. The `:` operator in this case is interpreted as bounds of `0,-1` where `-1` is later translated to the width of that dimension. It is also possible in Yorick to specify a lower bound with an omitted upper bound such as `2:`. This is translated as `2:-1` and the `-1`, again, is interpreted as the width of that dimension.

After `widths` have been calculated, a total number of updates is calculated as the product of widths. This is the final piece needed to begin updating elements within the specified array.

An array of counters, each starting at the lower bound for a given dimension, is maintained and as our counting proceeds, each count is updated after the dimension below it has exceeded its width. For example, with an array of shape `3x4` and an access pattern of `(:,2:)`, the accesses are interpreted as `(2,3 , 0,2)`. This update corresponds to updating every element in rows 2 and 3. Note again that Yorick computation is 1-based while the C translation is 0-based. Additionally, while Yorick notates inner dimensions on the left-hand-side, the C translation considers them to be on the right-hand side.

A simulated counter for the above `(2,3 , 0,2)` access can be seen as:
```
// Inner-most dimension is right-most index
counts[2][0]
counts[2][1]
counts[2][2]
counts[3][0]
counts[3][1]
counts[3][2]
```
As the underlying interpretation of `yarray` is a contiguous block of memory, accesses are computed using the formula:  
$$\Sigma_{i=0}^{n} (strides[i] * dims\_index[i])$$  
where `dims_index` holds the same meaning as counts which was the previously used terminology.
