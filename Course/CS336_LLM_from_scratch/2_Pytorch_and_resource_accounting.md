---
created: 2026-03-12T06:24
updated: 2026-09-29T22:47
---
### Data types
- almost everything are stored as float (weights, bias, )
- details: [[Numeric dtypes _ memory usage]]
	- reduce precision (BF16) or magnitude (FP16) -> <span style="color:rgb(255, 0, 0)">save time & memory</span>
- mixed precision training:
	- fp32 for key parts (eg. optimizer states)
		- note: max required precision for DL is fp32
	- bf16/fp16/fp8 for parameters/activation/gradients
		- less stable (over/under-flow etc)
		- rec to just touch FP32 & BF16
	- PyTorch has built-in to cast things when safe (AMP)
### Pytorch
#### tensors
- [[0_Tensor Core Mental Model]], [[1_Create]] for tensor basics
- [[2_Deep Learning]], [[Backpropagation]], [[Epoch_Batch]] for basics DL knowlege & code

### einops
- [[einops]] basics syntax
- `einsum`
> Multiply dimensions that have same name, then **sum** over dimensions that appear in the inputs but disappear from the output
- can use `...` for broadcasting over several number of dimensions


**Motivation: Feed forward Net**
- Sequence of **linear transformations + activation functions**.

$$\begin{gather}

Y = a(XW + b) \\ 
X:[B,I],W:[I,H],b:[H],Y:[B,H]
\end{gather}$$


- a = activation
- X = input (batch, input dim)
- W = weight (input, hidden dim)
- b = bias (hidden dim)
```python
# original
Z = X @ W + b

# einops
Z = einsum(
	X, W, 
	"batch input, input hidden -> batch hidden"
	) + b

```

**Motivation 2: Pairwise dot product**
- Similar to the query-key dot product in attention
- strength: describe which dimensions interact directly, without manually transposing/rearranging tensors first
```python
# torch
for b in batch:
    z[b] = x[b] @ y[b].T

# einops
z = einsum(
    x, y,
    "batch seq1 hidden, batch seq2 hidden -> batch seq1 seq2"
)
# > dot product over hidden
```