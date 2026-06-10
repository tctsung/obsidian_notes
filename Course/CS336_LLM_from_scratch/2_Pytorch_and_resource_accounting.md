---
created: 2026-03-12T06:24
updated: 2026-05-10T23:47
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
- 
