---
created: 2026-07-27T09:37
updated: 2026-09-03T09:53
---

## Big picture

| #   | Concept          | Type       | One line                                                   |
| --- | ---------------- | ---------- | ---------------------------------------------------------- |
| 1   | Gradient descent | the *goal* | nudge params downhill: `param -= lr * grad`, repeat        |
| 2   | Chain rule       | math fact  | nested funcs → **multiply** slopes; forked paths → **sum** |
| 3   | Backpropagation  | algorithm  | get **every** param's grad in **one** backward pass        |

- flow: descent wants gradients → **but where do gradients come from?** → backprop leans on the chain rule to compute gradients
- one training step always does **both**: `forward → loss → backward (grads) → descent (update)`

---
## 1. Gradient descent
- **goal**: find params that make the loss small. Loss = "how wrong are we", a single scalar
- **def**: repeatedly step each param <span style="color:rgb(255, 0, 0)">opposite its gradient</span> (the **slope** of loss w.r.t. that param)
- intuition: you're on a foggy hill, want the valley. Feel the slope under your feet, take a step **downhill**, repeat
- formula:
	- `lr` (learning rate) = $\eta$ = step size
	- near the valley the slope → 0, so steps shrink & it naturally settles
$$\theta \leftarrow \theta - \eta \cdot \frac{\partial L}{\partial \theta}$$
- see [[DL Training basics]] for epoch / batch / iteration (we run descent once per **batch**)

---
## 2. Chain rule
- **calculus** trick descent needs to actually get its gradients
- two cases, two operations:
#### Case 1 — stacked (composed) → **multiply**
- a wiggle propagates `Δx → Δy → Δz`, so multiply the local slopes

$$y = g(x), z = h(y) ~~~ \Rightarrow ~~~  \frac{dz}{dx} = \frac{dz}{dy}\,\frac{dy}{dx}$$

#### Case 2 — forked paths (one var → many routes) → **sum**
- `s` reaches `z` via both `x` and `y`, so sum the two routes

$$x = g(s),~y = h(s),~z = k(x, y) ~~~  \Rightarrow ~~~ \frac{dz}{ds} = \frac{dz}{dx}\,\frac{dx}{ds} + \frac{dz}{dy}\,\frac{dy}{ds}$$

#### How this maps to DL / backprop
- **Case 1 = depth**: stacked layers `L( aₙ( … a₁(w) ) )` → backprop multiplies slopes layer-by-layer down the net (also where vanishing/exploding grads come from)
- **Case 2 = fan-out**: any value **used in N places** → its grad = **sum of N contributions**. 
	- eg. **FFNet**: one neuron/weight feeds *every* neuron in the next layer → its update sums the grads flowing back from all of them
	- batch: loss = mean over samples → sum of per-sample grads
	- weight sharing (CNN filter, RNN step): same param reused → sum over patches/timesteps
- big picture: **forward fans out** (1 value → many uses); **backward fans in** (many grads → summed back onto that value)

---
## 3. Backpropagation
- **the gap**: gradient descent needs `grad` for *every* param. A big net has millions. Not possible to calc one param at a time
- def: compute the loss-gradient for <span style="color:rgb(255, 0, 0)">every param in ONE backward pass</span> by reusing the chain rule down the graph, very **efficient** algorithm
	- the reason backprop has the name & why deep learning is even feasible
#### how it works — record forward, replay backward
- **forward**: as you compute, autograd secretly **tapes** a graph. Every new tensor remembers the operation that create it (its **`grad_fn`**)
- **backward** (`.backward()`): walk that tape from loss → params, applying **chain rule (multiply)** along each path & **sum rule (fan-in)** where paths rejoin. Save result in <span style="color:rgb(255, 0, 0)">leaf params</span> `.grad`
	- intermediate tensors flow through (gradient calculated but NOT saved)
	- we only care about leaf params for update (w, b)
- <span style="color:rgb(255, 0, 0)">no need to label "this is loss function".</span> Autograd **differentiates whatever arithmetic you wrote** → the final differentiable expression must be <span style="color:rgb(255, 0, 0)">single number</span> (a scalar) as a valid loss
## PyTorch practical guides
```markdown
  loss            ← root  (grad_fn = MeanBackward)
   │
  pred            ← internal node (grad_fn = AddBackward)
 ╱    ╲
x @ w   b          ← internal (grad_fn = MulBackward)
│
w                 ← LEAF (grad_fn = None) — nothing created it
```
- **`requires_grad=True`** → tensor becomes a <span style="color:rgb(255, 0, 0)">leaf</span> that tracks all ops done on it
	- leaf has `grad_fn=None`
- grads **accumulate** → must `.grad.zero_()` each step
	- avoid accumulate gradient of one mini-batch to another
	- matches the fan-in "sum" behavior
- graph is rebuilt **fresh every iteration** (define-by-run) → can put `if`/loops in different iterations, forward & autograd just follows the code at run-time
- loss must be a **scalar** → that's why loss lines end in `.mean()` / `.sum()`
### Ex 1. Toy
- Goal: converge w to 4
```python
import torch

# --- smallest possible loop: learn w so that w = 4 (the target) ---
w = torch.tensor(0.0, requires_grad=True)    # param (leaf: tracks ops on w)
lr = 0.05                                    # step size
loss_fn = lambda w: (w - 4) ** 2             # MSE loss: how far w is from 4

for step in range(30):
    loss = loss_fn(w)                        # forward (builds graph)
    loss.backward()                          # backprop: calc w.grad
    print(f"Round {step}: w = {w.item()}, grad = {w.grad.item()}")
    # w.grad = d/dw (w-4)^2 = 2*(w-4)        <- chain rule (inner slope = 1)
    with torch.no_grad():
        w -= lr * w.grad                     # gradient descent
        w.grad.zero_()                       # reset grads to avoid accumulation
# > w:    0.0 -> 1.87 ... 3.82 (climbs to 4)
# > grad: -7.12  -> -6.45 -> -0.37 (flattens near the valley)
```

### Ex 2. Linear regression
- Goal: train linear model on line `y = 3x + 2`
```python
torch.manual_seed(0)                            # reproducible seed
x = torch.rand(100, 1)                          # 100 inputs, uniform in [0, 1)
y = 3 * x + 2 + 0.1 * torch.randn_like(x)       # noisy true line (target)

# params (leaves) + hyperparams + functions
w = torch.zeros(1, 1, requires_grad=True)       # param: slope
b = torch.zeros(1, requires_grad=True)          # param: intercept
lr = 0.1                                        # step size
forward = lambda x: x @ w + b           # forward: w fans out over all 100 rows
loss_fn = lambda pred, y: ((pred - y) ** 2).mean()  # mean = sum rule over batch

for step in range(200):
    y_pred = forward(x)                     # forward (builds graph)
    loss = loss_fn(y_pred, y)               # turn 100 errors -> 1 scalar
    loss.backward()                              # backprop: calc grad for leaf
    with torch.no_grad():
        w -= lr * w.grad                         # gradient descent
        b -= lr * b.grad
        w.grad.zero_(); b.grad.zero_()           # reset grads
# > converge to w≈3, b≈2   
```