---
created: 2026-06-23T09:46
updated: 2026-06-23T09:48
---

## tensor 
### core concept
- def: <span style="color:rgb(255, 0, 0)">pointers</span> to some allocated memory <span style="color:rgb(255, 0, 0)">+ metadata</span> to get values & do operations
- metadata includes:
	- **stride**
	- shape
	- dtype (link to another doc call: numeric dtypes)
	- device (cpu, gpu, mps)
#### ⭐ stride
- BG: logically reshape tensor to k-dim metrics & still use the **same 1D physical storage**
	- how: use metadata to record no. of ele to skip to locate conceptually different dim 
- def: <span style="color:rgb(255, 0, 0)">metadata</span> tuple of **step size** (in the underlying 1D storage) for each dim of a k-dim tensor
- <span style="color:rgb(255, 0, 0)">Last stride is always 1</span> (known as: row-major, C-order)
	- elements along the **last dim are physically adjacent in memory**
	- eg. for index `[0, 0]` ->  `[0, 1]` is closer comparing to `[1, 0]` 
```markdown
( dim0   dim1   dim2 )
   │      │        └─ stride = 1          ← smallest = CLOSEST in memory = fast
   │      └────────── stride = size(dim2)
   └───────────────── stride = size(dim2)*size(dim1)  ← biggest = FARTHEST = slow
```

- Code
```python
# stride
x = torch.tensor([[0, 1., 2.], [3., 4., 5.]])
x.stride()  # (3, 1)
x[1,2] # 1*3 + 2*1 = 5 → physical_memory position 5

y = torch.tensor([[[0., 1.], [2, 3]], [[4., 5.], [6., 7.]]])
y.stride()  # (4, 2, 1)
```

### tensor operations
- two main types

| operation type | description                                                                                      |
| -------------- | ------------------------------------------------------------------------------------------------ |
| View           | create a <span style="color:rgb(255, 0, 0)">new view</span> of existing tensor                   |
| Copy           | create a <span style="color:rgb(255, 0, 0)">new memory block</span>  with additional **compute** |

##### View
- def: create a **new view** of same tensor, did <span style="color:rgb(255, 0, 0)">NOT make copy</span>
	- mutation will affect both pointers since it's diff view of the same tensor
- `.transpose(i, j)`: swaps strides without moving data 
	- make tensor non-contiguous連續
	- some operations requires tensor to be contiguous (eg. `.view()` 
```python
x = torch.tensor([[1., 2, 3], [4., 5, 6]])
x[0]         # [1, 2, 3]
x[:, 1]      # [2, 5]
x.view(3,2)  # view 2X3 as 3X2 matrix

# mutation will affect both VIEWS since it's the same tensor
y = x.transpose(0, 1)
x[0][1] = 100
y[1]                    # [100, 5]
```
##### Copy
- create the data into a <span style="color:rgb(255, 0, 0)">new memory block</span>  with additional compute
- `.is_contiguous()`: bool, tensor's <span style="color:rgb(255, 0, 0)">logical order matches physical order</span> in memory
```python
y = x.transpose(0, 1)   # non-contiguous, still points to x's memory
y.view(2,3)   # error

z = y.contiguous()  # copy to new memory: [1,4,2,5,3,6] laid out sequentially

y.is_contiguous()  # F
z.is_contiguous()  # T
```
- `.reshape()` ==  `.contiguous()` + `.view()`
```python
x.reshape(1, 6)  # [1, 2, 3, 4, 5, 6]
```
---
## einops

### rearrange
- def: **reorder / merge / split** axes — <span style="color:rgb(255, 0, 0)">element count stays the same</span>,<span style="color:rgb(255, 0, 0)"></span> no math on values
- syntax: **`rearrange(tensor, "in -> out", **axis_sizes)`**
#### transpose
```python
import einops
# simple example
x = torch.tensor([[0, 1., 2.], 
				  [3., 4., 5.]])
einops.rearrange(x, "row col -> col row")  # flip through diagnol ↘ line
# > tensor([[0., 3.],
# >         [1., 4.],
# >         [2., 5.]])
```
- Coordinate Convention
	- Origin = **top-left**. <span style="color:rgb(255, 0, 0)">Indices increase like reading English</span> (→ right, ↓ down)
	- Why: screens write pixels to memory in this order (row 0 = top, not bottom)
	- **important detail for** <span style="color:rgb(255, 0, 0)">images</span>
```python
rearrange(img1, "h w c -> w h c") # height width channel/color
# > diagonal reflect from ↘
```
#### merge
- merge adjacent dimension into 1
- **dim=0 represent batch**, check [[Training Basics#Training Loop Terminology]] for why
- length of newly composed axis is a product of components
	- eg. `"2 (3 5) -> 2 15"`
```python
x1 = einops.rearrange(x, "r c -> (r c)")
# > [0, 1., 2., 3., 4., 5.]


# imagine ims is an batch of 3 images: [x], [y], [z]
einops.rearrange(ims, "b h w c -> h (b w) c") # merge to one with same h
# > [x][y][z] -> [xyz]
 
einops.rearrange(ims, "b h w c -> (b h) w c") # combine imgs to one with same w
# [x][y][z] ->  [x
#			     y
#			     z]
```
#### split/decompose
- if splitting into k sub-axes → must specify k-1 sizes (einops infers the last from total)
	- e.g. `h=4`, split to `(h1 h2)`: provide `h1=2` → einops infers `h2 = 4/2 = 2
```python
x = torch.tensor([[0, 1., 2., 3.],
				  [4., 5., 6., 7.]])

# Q: group odd & even numbers:
y = rearrange(x, "r (c1 c2) -> r c2 c1", c1=2)
# > tensor([[[0., 2.],
#           [1., 3.]],
#
#          [[4., 6.],
#           [5., 7.]]])

# eg. Multi-head attention: split embedding into heads
rearrange(x, "batch token (head dim) -> batch head token dim", head=8)
```
- image example
```python
# ims = 6 images "e", "i", "n", "o", "p", "s"
rearrange(ims, "(b1 b2) h w c -> (b1 h) (b2 w) c ", b1=2)
# > e i n
# > o p s

# 1. b1=2: split batches to 2 * 3
# 2. b1 merge h: the vertical direction should have ori 2 images 'height'
# 3. b2 merge w; the horizontal direction should be 3 * ori image 'width'
```
#### image spatial reshape

- Ex. move part of width dimension to height -> shrink width & double the height
```python
# simple example
img = torch.tensor([[1,2,3,4],[5,6,7,8]])  # 2×4
rearrange(img, "h (w1 w2) -> (h w2) w1", w2=2)

# with batch & channel
rearrange(img, "b h (w1 w2) c -> b (h w2) w1 c", w2=2)

# pixel arrangement (look taller)
Before (2×4):         After (4×2):
┌──────────────┐     ┌───────┐
│ p1 p2 p3 p4  │     │ p1 p3 │
│ p5 p6 p7 p8  │     │ p2 p4 │
└──────────────┘     │ p5 p7 │
                     │ p6 p8 │
                     └───────┘
```
- Ex 2: shrink height & double width
```python
img = torch.tensor([[1,2],[3,4],[5,6],[7,8]]) # 4 * 2
einops.rearrange(img, "(h1 h2) w -> h1 (h2 w)", h2=2)

# pixel arrangement (look wider)
Before (4×2):        After (2×4):
┌───────┐           ┌──────────────┐
│ p1 p2 │           │ p1 p2 p3 p4  │
│ p3 p4 │           │ p5 p6 p7 p8  │
│ p5 p6 │           └──────────────┘
│ p7 p8 │
└───────┘
```
#### Concatenation
- can automatically transform list of same shape array

```python
from torch import tensor
x = [ 
    tensor([[0], [1], [2]]),
    tensor([[3], [4], [5]])
	]

rearrange(x, "b r c -> b r c").shape # 2, 3, 1
# > b: "list axis" becomes first

rearrange(x, "b r c -> r (b c)") # 3, 2 
# > tensor([[0, 3], [1, 4], [2, 5]])
# diff col then batch became cloest
# diff row became furtherest
```

### ⭐ Order of axes
- same rule as [[#stride]]: in a composition `(d0 d1 d2)`
	- <span style="color:rgb(255, 0, 0)">rightmost axis is closest in memory</span>, changes fastest (d2)
	- leftmost changes slowest (d0)
- swapping axis order inside `( )` changes how elements interleave
	- <span style="color:rgb(255, 0, 0)">right/inner: small stride -> closer to each other</span>
	- left/outer: large stride -> further
- nice intuition: last dim len = no. of ele of the lowest-level bracket
```python
# ims = 6 images "e", "i", "n", "o", "p", "s"

rearrange(ims, "b h w c -> h (b w) c")  # b slow -> full image, then next image
# > [e][i][n][o][p][s]   side by side
```
![[Pasted image 20260602213944.png|561]]

```python
rearrange(ims, "b h w c -> h (w b) c")  # b fast -> width look overlapping 
# > 6X wider ghosted image
```
![[Pasted image 20260602213901.png|515]]
- decompose then recompose: 
```python
# ims = 6 images "e", "i", "n", "o", "p", "s"

rearrange(ims, "(b1 b2) h w c -> h (b2 b1 w) c", b1=2)  
# 1. split to 2X3: r1- ein, r2- ops
# 2. b1 (row repr) became a smaller stride dim because it's inner/on the right
# -> "diff row same col" are now closer than "diff col same row"
# > eoipns (eo, ip, ns: diff row same col)
```

### reduce
- def: **collapse** axes by aggregating — <span style="color:rgb(255, 0, 0)">axes missing from the output get reduced</span>
- syntax: `reduce(tensor, "in -> out", operation, **axis_sizes)` 
- type of operations: `mean`/`max`/`sum`/`min`/`prod`
```python
from einops import reduce
# ims = 6 images "e", "i", "n", "o", "p", "s"

# mean:
reduce(ims, "b h w c -> h w c", "mean")  # drop batch
# > 1 img of 6 letters stacked (fainted)

# min
reduce(ims, "b h w c -> h w c", "min")   
# > i img of 6 letters stacked, darker

# turn colored img to grayscale
reduce(ims, "h w c -> h w", "mean")
```
#### ⭐ downsample (pool) vs fog
- same [[#⭐ Order of axes]] concept: <span style="color:rgb(255, 0, 0)">reduce the inner factor (stride 1), because only adjacent pixels are real neighbors to pool</span>
- syntax: split each axis to `(coarse fine)`, then **reduce `fine` (inner = the kernel), keep `coarse` (outer = the grid)**
- eg. local 3x3 pool (3 adjacent px on w & h -> turn to 1 pixel)
```python
# mean pooling
reduce(ims, "b (h1 h2) (w1 w2) c -> h1 (b w1) c", "mean", h2=3, w2=3) 
# > 3X smaller sharp einops img

# max pooling
reduce(ims, "b (h h2) (w w2) c -> h (b w) c", "max", h2=2, w2=2) 
# > less smooth than mean
```
- Incorrect example (reduce the **outer** (large stride) -> folds far-apart regions together)
	- not a meaningful reduction
	- ~~`reduce(ims, "b (h1 h2) (w1 w2) c -> h2 (b w2) c", "mean", h1=3, w1=3)`~~
	- eg. `h (96) -> h1(3) X h2(32) -> h2 (32)`
```markdown
h1=0  ->  rows  0..31   TOP third
h1=1  ->  rows 32..63   MIDDLE third
h1=2  ->  rows 64..95   BOTTOM third

**Output (32): avg(0, 32, 64), avg(1, 33, 65) ... avg(31, 63, 95)**
```

### add / remove axes
- def: insert/drop a <span style="color:rgb(255, 0, 0)">length-1 axis</span> — carries no data, only structure
- einops version of torch `unsqueeze`/`squeeze` 
- two ways to write a length-1 axis, <span style="color:rgb(255, 0, 0)">they're synonyms</span>:
	- `1`: a literal length-1 axis
	- `()`:  composition of **zero axes**, also unit length
	- eg. `"b -> b 1 1 1"` == `"b -> b () () ()"`
#### add (broadcasting)
- def: insert length-1 axes to make match shape for pair-wise product at proper dim
	- note: <span style="color:rgb(255, 0, 0)">broadcasting aligns shapes from the right</span>
- Eg: update brightness per-image (batch dimension)  
```python
imgs = torch.ones(2, 2, 2, 3)            # (b0 h w c)
brightness = torch.tensor([0.5, 2.0])    # (b1,) one factor per image

# broadcasting:
factor = rearrange(brightness, "b1 -> b1 1 1 1")  # (2, 1, 1, 1)
# b0 matches b1 ✓

# pair-wise product:
imgs * factor   
# > each img scaled by its own factor
# > img0 all 0.5 (dimmed), img1 all 2.0 (brightened)
```
- `reduce` with a length-1 axis = **keepdims**: reduce **and** leave the broadcast hole in one step (vs reduce then manual `1`s above)
	- eg. `b h w c -> b () () c`: collapse h,w but keep `(b, 1, 1, c)` so it broadcasts back
	- see [[#🎯 capstone: per-image normalize]]
#### remove (squeeze)
- def: drop a length-1 axis to clean up after an op or fix a shape mismatch
- eg. squeezing regression/classification model ending in `Linear(h, 1)` to match true label
```python
pred = model(x)                  # (32, 1)
loss = mse(pred, targets)        
# > (32,1) vs (32,) -> broadcasts to (32, 32);  silent bug

# with squeeze:
pred = rearrange(pred, "b 1 -> b")  # (32, 1) -> (32,)
loss = mse(pred, targets)           # (32,) vs (32,) ✓
```
- squeeze vs [[#reduce]]: 
	- squeeze only removes an **already length-1** axis (no data lost, reversible); 
	- reduce removes **any length** by aggregating (irreversible). 
	$\rightarrow$ should use `rearrange` to squeeze so it fails loudly instead of silently collapsing real data

### repeat
- def: duplicate values to grow an axis (or add one) — three variations:
	- **repeat**: copy along a *new* axis -> `[A] -> [A, A, A]`
	- **tile**: copy a whole block along an *existing* axis -> `ABC -> ABCABCABC`
	- **stretch**: copy each element in place -> `ABC -> AAABBBCCC`
- same [[#⭐ Order of axes]] guideline: number on the **outer/left** = tile, **inner/right** = stretch
#### repeat (new axis)
```python
from einops import repeat
x = torch.tensor([[0, 1.], [2., 3.]])

# verbose: name the new axis + pass its size
repeat(x, "r c -> r new_axis c", new_axis=3)
#  [
#   [[0., 1.], [0., 1.], [0., 1.]], 
#	[[2., 3.], [2., 3.], [2., 3.]]
#  ]

# shortcut: number directly in pattern (used from here on)
repeat(x, "r c -> 3 r c")
# [
#  [[0, 1.], [2., 3.]]
#  [[0, 1.], [2., 3.]]
#  [[0, 1.], [2., 3.]]
# ]
```

#### tile  (`ABC -> ABCABC`)
- number **outer/left** of an existing axis -> repeat the whole block
```python
repeat(x, "r c -> r (3 c)")    # tile each row
# [[0., 1., 0., 1., 0., 1.], 
#  [2., 3., 2., 3., 2., 3.]]

repeat(x, "r c -> (2 r) (2 c)")  # tile both axes
# [
#   [0., 1., 0., 1.], [2., 3., 2., 3.], 
#   [0., 1., 0., 1.], [2., 3., 2., 3.]
# ]
```
- image: `ims[0]` = one "e"
```python
repeat(ims[0], "h w c -> h (repeat w) c", repeat=3)
# > three "e"s side by side
```

#### stretch  (`ABC -> AAABBB`)
- number **inner/right** of an existing axis -> repeat each value in place
```python
repeat(x, "r c -> r (c 3)")
# [[0., 0., 0., 1., 1., 1.], 
#  [2., 2., 2., 3., 3., 3.]]
```
- image: `ims[0]` = one "e"
```python
repeat(ims[0], "h w c -> h (w repeat) c", repeat=3)
# > one "e" stretched 3× wider (fat "e")
```

### Capstone: per-image normalize
- **goal**: subtract background & normalize **each image individually**
- **solution** — the `reduce` gives one max per image per channel `(b, 1, 1, c)`; the `()` axes broadcast it back over every pixel of that same image:
	1. **flip** — `each_image_max - pixel`
		- brightest pixel -> `max - max = 0`
		- darker pixel -> `max - small = large`
	2. **normalize** — divide each pixel by its image's new max -> rescale to `[0, 1]`
		- per-image, so a dim image & a bright image end on the same scale (comparable)
	3. **layout** — lay the 6 cleaned images side by side along width
```python
# 1. flip
im2 = reduce(ims, "b h w c -> b () () c", "max") - ims

# 2. normalize
im2 = im2 / reduce(im2, "b h w c -> b () () c", "max")

# 3. layout
rearrange(im2, "b h w c -> h (b w) c")
```

Original
![[Pasted image 20260622164320.png]]
Normalized:
![[Pasted image 20260622164337.png]]