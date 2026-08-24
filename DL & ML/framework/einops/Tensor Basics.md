---
created: 2026-08-23T23:29
updated: 2026-08-23T23:33
---
## tensor 
- goal: to store the input data & weights and bias of NNET
### core concept
- def: <span style="color:rgb(255, 0, 0)">pointers</span> to some allocated memory <span style="color:rgb(255, 0, 0)">+ metadata</span> to get values & do operations
- metadata includes:
	- **stride**
	- shape
	- dtype (link to another doc call: numeric dtypes)
	- device (cpu, gpu, mps)
#### ⭐ stride
- BG: logically reshape tensor to k-dim metrics & still use the **same 1D physical storage**
	- how: use metadata to record no. of ele to skip to **turn 1D into conceptually k-dim tensor**
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
