---
created: 2026-10-22T09:34
updated: 2026-10-22T13:29
---

## 1. Intro
- **def**: CNN (Convolutional Neural Network) = NNET that uses <span style="color:rgb(255, 0, 0)">convolution in at least one layer</span>
- common usage:
	- any data where <span style="color:rgb(255, 0, 0)">nearby values are related</span> (neighbor pixels, consecutive audio samples)
	- **image** (main one): classification, object detection, segmentation
	- speech/audio, time series (1D conv), board games (eg. AlphaGo)

### Difference to FFNET
- both are linear combination + activation $\sigma(\sum w_i x_i + b)$
- <span style="font-weight:bold; color:rgb(255, 0, 0)">conv = FFNET weight matrix with forced zeros & repeated values</span>
- CNN is less flexible than FFNET (larger bias), but avoids FFNET's overfitting (dense weights)
- benefits:
	- parameter sharing: can find pattern anywhere (FFNET must learn separately at every position, CNN learns it once)
	- FFNET doesn't know which pixels are **neighbors** (shuffle all pixels the same way → FFNET learns equally well, CNN wouldn't)

|                  | FFNET (fully connected)             | CNN (conv layer)                                               |
| ---------------- | ----------------------------------- | -------------------------------------------------------------- |
| each output sees | **all** inputs                      | a small **window** (eg. 3×3)                                   |
| weights          | own weight per (input, output) pair | **same** kernel reused at every window (**parameter sharing**) |
| data shape       | flattened 1D vector                 | stays 2D/3D `(c, h, w)` until the end                          |

- 1D toy example
```markdown
FFNET: 15 free params         conv, w = [w1 w2 w3]: 3 free params
[a b c d e]                   [w1 w2 w3 0  0 ]
[f g h i j]  @ x              [0  w1 w2 w3 0 ]  @ x
[k l m n o]                   [0  0  w1 w2 w3]
```

---
## 2. Basics
### Convolution
#### math
- **def**: slide a small weight grid (**kernel** $w$) over the input; at each position take the <span style="color:rgb(255, 0, 0)">dot product</span> of the kernel & the window under it, + bias → **one number**
$$z_{i,j} = w \cdot x_{[\,i:i+k,\ \ j:j+k\,]} + b$$
	- $x_{[\,i:i+k,\ j:j+k\,]}$ = the k×k **window** of x starting at (i, j) (Python slice) → moving (i, j) = sliding the window
		- using 2D as example
	- $w$ = the **same** k×k kernel at every position
	- dot product: element-wise product, then sum
- `nn.Conv2d` = dot product + bias only; activation (`nn.ReLU`) is a separate layer

#### math intuition
- **why dot product?** it's the same neuron as FFNET ($w \cdot x + b$), just applied to one window → CNN changes *what* each neuron looks at, not the math
- is it really similarity? yes, dot product = **un-normalized cosine similarity**
$$w \cdot x = \|w\|\,\|x\|\cos\theta$$
	- for windows of the same brightness ($\|x\|$ fixed): biggest when $\cos\theta = 1$ → window has the <span style="color:rgb(255, 0, 0)">same shape as the kernel</span>
	- ∴ the kernel itself is the pattern it detects best → each kernel = a <span style="color:rgb(255, 0, 0)">pattern detector</span> (template matching)
	- evidence: first-layer kernels of trained CNNs (eg. AlexNet) look like edges & color blobs
	- caveat: un-normalized → a very bright window can also score high → bias + ReLU act as a threshold (fire only if $w \cdot x > -b$)
		- toy k1 sums to 0 → flat windows (all 0 or all 1) score 0 → responds only to edges
- **cross-correlation** = the name of this sliding dot product (the formula above)
	- not different from dot product: cross-correlation = dot product at every position
	- math **convolution** = same, but flips the kernel 180° first: $x * w = x \star \text{rot180}(w)$ ($*$ = convolution, $\star$ = cross-correlation)
	- 1D eg. x = [x1 x2 x3], w = [a b c]
		- cross-correlation: a·x1 + b·x2 + c·x3
		- convolution: c·x1 + b·x2 + a·x3
	- why DL skips the flip: w is learned → the net just learns [c b a] instead → same model. Not about speed; flipping is just pointless
		- math flips for nice algebra properties (eg. commutative); DL doesn't need them but kept the name "convolution"


### Kernel, window & receptive field

| Term            | What                                                                                                     | Learned?                      |
| --------------- | -------------------------------------------------------------------------------------------------------- | ----------------------------- |
| Kernel / filter | the weights `w` (eg. 3×3 grid)                                                                           | yes, by backprop              |
| Window / patch  | piece of input currently under the kernel                                                                | no, it's data                 |
| Receptive field | region of the <span style="color:rgb(255, 0, 0)">original input</span> that one output number depends on | no, follows from architecture |

- analogy: kernel = stamp, window = what's under the stamp, receptive field = how much of the original photo one output number has "seen"
- receptive field test: which pixels of x, if changed, would change this number?
	- layer 1: receptive field = window (3×3)
	- grows with depth: 2 stacked 3×3 convs → 5×5 ; 3×3 conv + 2×2 pool → 4×4
	- why care: an output can only detect patterns that fit in its receptive field → deep layers see big patterns (edge → texture → part → object)
- parameter sharing, two details:
	- **relative** position (inside the window) matters → that's what the kernel encodes
	- **absolute** position doesn't matter for detection → a stroke detector fires anywhere; the feature map still records *where*
	- gradient: same w used at every window → its grad = **sum** over all windows (fan-in, see [[Backpropagation#2. Chain rule]])

### Hyperparameters

| Hyperparam             | Meaning                                                           | Common                                 |
| ---------------------- | ----------------------------------------------------------------- | -------------------------------------- |
| kernel size `k`        | window size                                                       | 3×3 (odd → has a center pixel)         |
| no. of kernels `C_out` | no. of feature maps produced                                      | 16, 32, 64… (more as maps shrink)      |
| stride `s`             | step size when sliding                                            | 1; 2 to halve the size                 |
| padding `p`            | zeros added around the border so kernel can center on edge pixels | `p = (k-1)/2` keeps the size ("same")  |

- output size:
$$n_{out} = \left\lfloor \frac{n + 2p - k}{s} \right\rfloor + 1$$
	- eg. n=6, k=3: p=0, s=1 → 4 ; p=1 → 6 ; s=2 → 2
	- why `(k-1)/2`: without padding each conv shrinks by k−1 → pad half of it per side
- kernel shape = `(C_in, k, k)`: one kernel spans <span style="color:rgb(255, 0, 0)">all input channels</span> (eg. RGB → 3×3×3) → still one number per window
	- all kernels in a layer have the same shape → their maps stack into `(C_out, H, W)`
	- layer weight = `(C_out, C_in, k, k)` + `C_out` biases
	- different layers can differ (eg. ResNet: 7×7 first layer, then 3×3)

---
## 3. Procedure
![[cnn-pipeline.png|700]]
- toy example in the figure: 6×6 grayscale image with a short vertical stroke (1 = bright), 2 kernels, b = 0
- general recipe: `image → [conv → ReLU → pool] × N → flatten (or GAP) → FFNET → softmax`
	- conv & pool often alternate, eg. conv → conv → pool → conv → conv → pool

### (1) Data preparation
- input shape: `(c, h, w)`, eg. RGB 100×100 → `(3, 100, 100)`; grayscale → c = 1
	- batch → `(b, c, h, w)`, see [[Epoch_Batch]]
- every input must be the <span style="color:rgb(255, 0, 0)">same size</span> (a batch stacks into one tensor; flatten → FFNET needs a fixed length) → data augmentation, eg.
	- resize, crop
	- flip, rotate, scale, color jitter → also teaches CNN scaled/rotated patterns, which it <span style="color:rgb(255, 0, 0)">can't handle automatically</span>

### (2) Create kernels
- pick hyperparams ([[#Hyperparameters]]) → init kernel values **randomly**
- kernel values = unknown params, <span style="color:rgb(255, 0, 0)">learned by backprop</span> ([[Backpropagation]]) → become whatever patterns reduce the loss
	- old-school image processing used hand-designed kernels (blur, sharpen, edge); CNN learns its own
- toy (figure step 2): pretend training learned k1 = vertical edge (bright left, dark right), k2 = horizontal edge (bright top, dark bottom)

### (3) Convolution → feature map
- apply [[#Convolution]] at every window, then activation → <span style="color:rgb(255, 0, 0)">one kernel → one feature map</span>
- toy, the orange window in the figure:
$$\underbrace{\begin{bmatrix}1&0&0\\1&0&0\\1&0&0\end{bmatrix}}_{\text{window}} \cdot \underbrace{\begin{bmatrix}1&0&-1\\1&0&-1\\1&0&-1\end{bmatrix}}_{k_1} + 0 = 1+1+1 = 3 \ \xrightarrow{\text{ReLU}}\ \text{map 1}[0,2] = 3$$
- repeat at all (6 − 3)/1 + 1 = 4 → 4×4 windows → map 1. How to read:
	- 3 = "bright-left / dark-right edge here" (stroke's right side)
	- stroke's left side gives −3 (opposite edge) → ReLU zeroes it ([[Common functions#ReLu]])
	- lower rows are smaller: stroke ends at row 3 → lower windows contain less of it
- k2 → map 2: fires near the bottom end of the stroke
- output = 2 maps stacked → `(2, 4, 4)`; next conv layer treats it as a 2-channel image (kernels `2×3×3`)

### (4) Pooling
- **def**: replace each small block with a summary statistic → **shrink** each feature map
	- 白話: like dropping every other pixel of a photo → lower resolution, but the patterns are still there
	- max pooling (common): keeps *whether* a pattern exists, not its exact position; also mean pooling
- why: <span style="color:rgb(255, 0, 0)">speed up calculation</span> (fewer numbers downstream) + receptive field grows faster
	- cost: loses information (exact position)
- toy (figure step 4): 2×2 max pooling, stride 2 → each 2×2 block → its max: `(2, 4, 4)` → `(2, 2, 2)`
- order: ReLU **before** pool
#### Modern trend: less pooling

| Style                                | Downsampling                              | Head                            |
| ------------------------------------ | ----------------------------------------- | ------------------------------- |
| Classic (LeNet, VGG)                 | pooling                                   | flatten → FFNET                 |
| Modern classifier (ResNet, ConvNeXt) | stride-2 conv (a *learned* downsample)    | GAP → one FC layer              |
| FCN (segmentation, AlphaGo)          | stride-2 conv / pool, or none             | no FC: 1×1 conv outputs a map   |

- **FCN** (fully convolutional network) = <span style="color:rgb(255, 0, 0)">no fully connected layers</span> → about removing the FFNET head, not pooling
- eg. **AlphaGo** policy network did <span style="color:rgb(255, 0, 0)">NOT pool</span> $\because$ every board position matters; dropping rows/cols of a Go board = a different game
	- input 19×19×48 (feature planes) → 5×5 conv, then 11 × 3×3 conv, all padded to keep 19×19, 192 kernels each, ReLU → 1×1 conv → softmax over 19×19 moves

### (5) Flatten → FFNET
- after the last conv/pool: tensor `(c, h, w)` → turn it into a vector, 2 ways
- **flatten** (figure step 5): read every number in order (channel → row → col), no math
	- keeps position; but vector length depends on image size
- **GAP** (global average pooling): mean of each map → one number per channel
	- toy: `[(0+3+0+2)/4, (0+0+1+1)/4] = [1.25, 0.5]`
	- drops position, keeps "how much of each feature appeared anywhere"
	- length = no. of channels → <span style="color:rgb(255, 0, 0)">any input size works</span> & far fewer FFNET params → modern default
- then a plain FFNET ([[2_Deep Learning]]): `logits = W·v + b` → softmax → class
	- eg. learns rules like "strong vertical edges, no horizontal ones → probably digit 1"
- big picture: conv part = <span style="color:rgb(255, 0, 0)">learned feature engineering</span>; FFNET head = classifier on those features

---
## Example: code

#### Manual
- convolution only: k1 on the toy x
```python
import torch
from einops import einsum

x = torch.zeros(6, 6)
x[0:4, 2] = 1                                        # toy image
k1 = torch.tensor([[1., 0, -1]] * 3)                 # vertical-edge kernel

windows = x.unfold(0, 3, 1).unfold(1, 3, 1)          # every 3×3 window: (4, 4, 3, 3)
z = einsum(windows, k1, "h w kh kw, kh kw -> h w")   # dot product per window
# > [[-3, 0, 3, 0],
# >  [-3, 0, 3, 0],
# >  [-2, 0, 2, 0],
# >  [-1, 0, 1, 0]]
```
- `unfold(dim, size, step)`: windows of length `size` every `step` (= stride) along `dim`

#### CNN with einops
- syntax see [[einops]]
- pooling, GAP & flatten as einops ops (from einops docs)
```python
from einops import rearrange, reduce

y = reduce(x, "b c (h 2) (w 2) -> b c h w", "max")   # 2×2 max pooling
y = reduce(x, "b c h w -> b c", "mean")              # GAP
y = rearrange(x, "b c h w -> b (c h w)")             # flatten
```
- same ops as layers inside a model (adapted from einops README), input `(b, 3, 32, 32)`
```python
from torch.nn import Sequential, Conv2d, MaxPool2d, Linear, ReLU
from einops.layers.torch import Reduce

model = Sequential(
    Conv2d(3, 6, kernel_size=5), ReLU(),             # (b, 6, 28, 28)
    MaxPool2d(kernel_size=2),                        # (b, 6, 14, 14)
    Conv2d(6, 16, kernel_size=5), ReLU(),            # (b, 16, 10, 10)
    Reduce("b c (h 2) (w 2) -> b (c h w)", "max"),   # pool + flatten in one step: (b, 400)
    Linear(16*5*5, 120), ReLU(),
    Linear(120, 10),                                 # 10 classes
)
```

### PyTorch 
```python
from torch import nn

model = nn.Sequential(
    nn.Conv2d(1, 2, kernel_size=3),   # (B, 1, 6, 6) -> (B, 2, 4, 4)  step 2 & 3
    nn.ReLU(),
    nn.MaxPool2d(2),                  # -> (B, 2, 2, 2)               step 4
    nn.Flatten(),                     # -> (B, 8)                     step 5
    nn.Linear(8, 10),                 # -> (B, 10) logits
)
# > model(torch.randn(32, 1, 6, 6)).shape = (32, 10)
```
