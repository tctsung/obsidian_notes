---
created: 2026-08-23T16:44
updated: 2026-09-03T11:42
---


### Create
- from scratch
```python
x = torch.zeros(2, 2)          # 2×2, filled with 0
x = torch.ones(2, 2, 3)        # 2×2×3, filled with 1
x = torch.empty(2, 3)          # uninitialized values (NOT zeros)
x = torch.rand(2, 2)            # uniform random [0, 1)
x = torch.randn(2, 2)           # standard normal N(0, 1)
x = torch.arange(10)            # [0, 1, ..., 9]
x = torch.arange(0, 10, 2)      # [0, 2, 4, 6, 8]
x = torch.linspace(0, 1, 5)     # [0, 0.25, 0.5, 0.75, 1]

# Identity / diagonal
x = torch.eye(3)                # 3×3 identity matrix
x = torch.diag(torch.ones(3))   # diagonal matrix

# same shape as y
x = torch.zeros_like(y)            
x = torch.ones_like(y)
x = torch.rand_like(y)

# same shape + value
x_new = x.clone().detach()
```

- from existing data
	- check [[Numeric dtypes _ memory usage]] for dtype details
```python
# list → tensor
x = torch.tensor([[1, -1], [-1, 1]])    

# numpy → tensor
x = torch.from_numpy(arr)  # share memory (val change at same time)
x = torch.tensor(arr)      # copies data (val don't change together)

# tensor → numpy
arr = x.numpy()                  # shares memory

# detach if requires_grad=True (GPU → CPU → NumPy)
arr = x.detach().cpu().numpy()   

# switch dtype
x = x.to(torch.float16)
x = x.to(torch.long)
```

### Basic Calculation

> For matrix multiplication and complex operations, see [[einops]] for more intuitive notation.

- Element-Wise

```python
x = torch.tensor([3, 1]) ; y = torch.tensor([6,0])
x + y                         # add
x - y                         # subtract
x * y                         # element-wise multiply
x / y                         # divide

x.pow(2)                      # x²
x ** 2                        # x²
x.sqrt()                      # √x
x ** 0.5                      # √x
x.exp()                       # eˣ
x.log()                       # ln(x)
x.abs()                       # |x|

## in-place calculation
y.add_(x) # y = y + x
y.sub_(x)
y.mul_(x)
y.div_(x) # y = y / x
```
- Reduction Operations (Dimensions collapse)
	- `dim=k`: <span style="color:rgb(255, 0, 0)">removes axis k from the output shape</span>.
```python
x = torch.tensor([[3.0, 1.0], [6.0,0.0], [2.0,0.0]]) 
# collapse all dim into single scalar
x.sum()            # 3+1+6+2           
x.mean()                      
x.max()                       
x.min()     

# collapse
x.mean(dim=1)       # Collapses dim 1 (columns) -> 2,3,1
x.max(dim=0)        # [6,1],[1,0]  -> returns (values, indices) 
x.min(dim=-1)       # [1,0,0],[1,1,1]  -> returns (values, indices)           
```


- Common Matrix operation
```python
x=x.squeeze(0)    # remove dim=0; eg. 1*2*3 -> 2*3
x=x.unsqueeze(1)  # add a dimension; eg. 2*3 -> 2*1*3

# matrix multiplication
x @ y                         
torch.matmul(x, y)        
```

### Subsetting
- use `.item()` to get the ele itself, otherwise, len=1 is still a tensor

```python
x[2,:] # single dim
x[1,1] # still a tensor
x[1,1].item() # get the actual ele
```
