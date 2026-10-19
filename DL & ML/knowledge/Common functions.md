---
created: 2026-10-12T23:11
updated: 2026-10-12T23:21
---



## Activation
### ReLu
- def: keeps positive numbers and turns negatives into 0
$$ReLU(x) = max(0, x)$$
- most commonly used activation function, why?
	- give FFNET <span style="color:rgb(255, 0, 0)">non-linearity</span> $\rightarrow$ can approximate any shapes
	- cheap calculation (sparsity: many 0) compare to sigmoid/tanh
	- avoid vanishing gradient

### Leaky ReLu
- why: to avoid dying ReLU (all input < 0) by adding a tiny non-zero gradient for negative input
$$LReLU(x) = max(\alpha x, x) ~~~~\text{where } \alpha = 0.01$$
- ensure continuous learning