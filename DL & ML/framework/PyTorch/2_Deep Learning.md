---
created: 2026-08-28T22:39
updated: 2026-09-20T21:55
---
Key mental model: [[0_Tensor Core Mental Model]]
Before: [[1_Create]]

### cpu ↔ gpu

```python
x = x.to("cuda")
x = x.to('cuda:0')  # specify if multiple gpu exist
x = x.to("cpu")

# Apple Silicon: unified mem so generally dont' need transfer
x = x.to("mps")  

# copy=True forces a new tensor
x = x.to("cuda", 
		 dtype=torch.float16,
         copy=False
		)

# Example script (auto-detect device)
if torch.cuda.is_available():
    device = torch.device("cuda")
elif torch.backends.mps.is_available():
    device = torch.device("mps")
else:
    device = torch.device("cpu")

# method 1. create at GPU
x = torch.ones(5, device=device)
# method 2. create 1st then move to GPU
y = y.to(device)
```

## autograd
- do backprop/gradient descent automatically
### Gradient
- Backprop details: [[Backpropagation#PyTorch practical guides]]
	- `.backward()` walk back to init param & accumulate `.grad` during all forward calc
- `with torch.no_grad():` avoid adding operation that is NOT a forward pass into backprop graph walk
	- eg. parameter update
- `.grad.zero_()`: let each `backward()` produce a fresh batch gradient
	- ref: [[Epoch_Batch]]- 
	- avoid param update do: grad1 + (grad1 + grad2) + (grad1+grad2+grad3)

```python
# track gradients
x = torch.ones(5, requires_grad=True) 
y = (x ** 2).sum()
y.backward()
x.grad              # ∂y / ∂x

# temp don't track gradient
with torch.no_grad():
	x -= learning_rate * x.grad
	x.grad
x.grad.zero_()  # empty gradient before nxt operation

# stop gradient tracking
x = x.detach() 
```
## Create a NNET
## `nn.Module`

**def:** a container for **learnable parameters + network computation**

- Base class for building neural networks and reusable model components
- Automatically tracks parameters and submodules
- Provides model-level utilities:
```python
model.parameters()  # get learnable parameters
model.to(device)    # move model
model.train()       # training mode
model.eval()        # evaluation mode
```

## `nn` vs `nn.functional` (`F`)

**Mental model:** 
- Use `nn` for layers that have <span style="color:rgb(255, 0, 0)">learnable parameters</span>, or need to be part of the model (eg. `Linear` , `Conv2d`: weights & bias & kernal values are <span style="color:rgb(255, 0, 0)">dynamic</span>)
	- reuse a module multiple times in a forward = share it's parameters -> do this when weight sharing is intentional, usually avoid (eg.`l1=nn.Linear(), l2=nn.Linear()`)
- Use `F` for stateless operations such as activations (eg. `relu`: calculation never changes)

|                                                                       | `nn`                  | `F`                        |
| --------------------------------------------------------------------- | --------------------- | -------------------------- |
| mental model                                                          | creates an OOP module | perform operation directly |
| Own/register parameters?                                              | Yes                   | No                         |
| Store state/configuration?                                            | Yes                   | No                         |
| Parameterized layer?                                                  | Yes                   | No                         |
| Stateless operation?                                                  | Sometimes             | Yes                        |
| Automatically changes behavior with `model.train()` / `model.eval()`? | Yes                   | No                         |



A reused `nn.Module` uses the **same parameters** each time. Create separate module instances when you want separate parameters.

### Common operations

|Operation|Common usage|Notes / setup|
|---|---|---|
|**ReLU**|`F.relu(x)`|Stateless activation|
|**GELU**|`F.gelu(x)`|Common in Transformers|
|**SiLU / Swish**|`F.silu(x)`|`x * sigmoid(x)`|
|**Softmax**|`F.softmax(x, dim=-1)`|`dim` = dimension to normalize|
|**Dropout**|`nn.Dropout(p=0.1)`|Usually `nn` because behavior changes between train/eval|
|**Linear**|`nn.Linear(128, 64)`|`128` = input features, `64` = output features|
|**Conv2d**|`nn.Conv2d(3, 64, 3)`|`3` = input channels, `64` = output channels, `3` = kernel size|
|**BatchNorm2d**|`nn.BatchNorm2d(64)`|`64` = number of channels|
|**LayerNorm**|`nn.LayerNorm(128)`|`128` = normalized dimension|
|**Embedding**|`nn.Embedding(10000, 256)`|`10000` = vocabulary size, `256` = embedding dimension|

#### Design Procedure
- mental model
	- __init__()  → what components does the model have?
	- forward()   → how does input flow through them?

1. Define the layers/components in `__init__()`
2. Define computation performed at every call in `forward()`
3. PyTorch automatically registers the layers and their parameters
4. `model.parameters()` can then be passed to an optimizer

#### Minimal example

```python
import torch.nn as nn

class NeuralNet(nn.Module):
    def __init__(self, input_size, hidden_size, num_classes):
        super().__init__()  # initialize the parent nn.Module
        self.l1 = nn.Linear(input_size, hidden_size)
        self.l2 = nn.Linear(hidden_size, num_classes)

    def forward(self, x):
        x = self.l1(x)
        x = F.relu(x)
        x = self.l2(x)
        return x
```


## Optimization

### Loss Function

> def: measure **how far prediction is from target**

| Task                      | Loss                           | PyTorch                  |                                           |
| ------------------------- | ------------------------------ | ------------------------ | ----------------------------------------- |
| Regression                | Mean Squared Error             | `nn.MSELoss()`           |                                           |
| Binary classification     | Binary Cross Entropy           | ~~`nn.BCELoss()`~~       | not preferred, requires extra `sigmoid()` |
| Binary classification     | Binary Cross Entropy + Sigmoid | `nn.BCEWithLogitsLoss()` |                                           |
| Multiclass classification | Cross Entropy                  | `nn.CrossEntropyLoss()`  | expect logit input, do NOT do `softmax`   |

- examples
```python
# binary
loss = nn.BCEWithLogitsLoss()
y_true = torch.tensor([1, 0, 1])
y_pred = torch.tensor([2.0, -1.0, 0.5])
loss_val = loss(y_pred, y_true)
loss_val.item()                 # get BCE

# multiclass:
loss = nn.CrossEntropyLoss()
y_true = torch.tensor([2, 0, 1])
y_pred = torch.tensor([
    [0.3, 1.0, 2.2],
    [2.0, 1.0, 0.2],
    [1.0, 2.5, 0.2]
])
# > no. of sample * no. of classes
# > NOT normalize with softmax 
loss_val = loss(y_pred, y_true)
loss_val.item()                 # get CrossEntropy
```

### Optimizer

> def: hold  current state and use **gradient to update all parameters**

```python
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)

loss.backward()
optimizer.step() 
# > update all parameters, don't need to update w-=, B-= ... by hand
optimizer.zero_grad()
```

### Training flow
- model → forward → predict → loss → backward (get gradient) → optimizer.step(update params)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch import optim

# Data
x = torch.tensor([
    [1, 1], [1, 2], [2, 2]
], dtype=torch.float32)

y = torch.tensor([
    [2],[4],[5]
], dtype=torch.float32)

# Model we created above
input_size = 2 ; hidden_size = 4 ; num_classes = 1
model = NeuralNet(input_size, hidden_size, num_classes)

# Loss + optimizer
loss_fn = nn.MSELoss()
optimizer = optim.SGD(model.parameters(), lr=0.01)


# Training
n_iters = 2000
for epoch in range(n_iters):
    # 1. Forward
    y_pred = model(x)
    # 2. Calculate loss
    loss = loss_fn(y_pred, y)
    # 3. Calculate gradients
    optimizer.zero_grad()
    loss.backward()
    # 4. Update parameters
    optimizer.step()
    # logs:
    if epoch % 100 == 0:
        print(f"epoch {epoch}: loss = {loss.item():.4f}")

```

