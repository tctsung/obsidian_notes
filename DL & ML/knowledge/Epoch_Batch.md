---
created: 2026-09-03T10:44
updated: 2026-09-03T10:44
---
## Training Loop Terminology
- 1 epoch too big for one forward pass of training -> split to batches

| Term                 | Definition                                                     | Notes                                                                                                       |
| -------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Epoch                | 1 full forward & backward pass of all <u>training data</u> (N) | Hyperparameter                                                                                              |
| Batch size (B)       | No. of training samples per iteration, usually power of 2      | **GPU memory** $\propto$ B<br>$\rightarrow$ need to feed whole batch into GPU for simultaneous forward pass |
| Iteration/batch/step | an update of $\theta$ (all learnable params)                   | N / B = iterations per epoch                                                                                |

* <span style="color:rgb(255, 0, 0)">dim=0 is always batch dimension</span> because DataLoader stacks samples along a new first axis
	* eg. single image = `(3, 224, 224)` → batch of 32 images = `(32, 3, 224, 224)`