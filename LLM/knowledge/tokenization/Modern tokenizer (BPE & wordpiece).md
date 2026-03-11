---
created: 2026-03-07T07:53
updated: 2026-03-11T07:30
---
## Intro
- details of most widely used tokenization algorithms: BPE & wordpiece
- automatically determine vocab by raw text frequency
- 
## BPE (byte-pair encoding)
- intuition: build tokenizer by repeatedly **merging the most frequent adjacent token pair** as a new token
- ref
	- [CS336](https://youtu.be/SQ3fZ1sAqXI?si=jVWDR75FAGxntr-g&t=4323), [youtube](https://youtu.be/tOMjTCO0htA?si=Y30VGw6WKMR8-2K8)
![[mermaid-diagram.png]]
#### Example
- **training**
	- merge rules are generated & thus <span style="color:rgb(255, 0, 0)">frequency-ordered</span>
```markdown
Corpus:
low lower lowest

Step 0 (characters)
l o w
l o w e r
l o w e s t

Step 1 merge (l,o) → lo  # generate rule one
lo w
lo w e r
lo w e s t

Step 2 merge (lo,w) → low  # generate rule two
low
low e r
low e s t
```
- **inference** (apply merge rules **sequentially**)
	- merge rules are ordered by training frequency, NOT token len
```markdown
input: lowering

Step 0 (characters)
l o w e r i n g

Merge rules order:
1. l + o → lo  
2. lo + w → low  
3. e + r → er
   
Apply rules:

Step 1 (l + o → lo)
lo w e r i n g

Step 2 (lo + w → low)  
low e r i n g

Step 3: e + r → er  
low er i n g
```
#### Pseudo-code
- C: input corpus strings
- k: number of merges
- V: vocabulary table
- t: token
```python
# initialize vocabulary with characters
V = unique_characters(C)

# represent corpus as character tokens
tokens = list(word for word in C) # eg. "low" → [l, o, w]

for i in range(k):

    # 1. find most frequent adjacent pair
    (t_L, t_R) = most_frequent_pair(tokens)

    # 2. create merged token
    t_new = t_L + t_R

    # 3. update vocabulary
    V.add(t_new)

    # 4. replace pair in corpus
    tokens = replace(tokens, (t_L, t_R) → t_new)

return V
```

### Optimization





## Word-piece