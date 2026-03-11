---
created: 2025-08-19T22:45
updated: 2026-03-11T08:24
---
* [ref huggingface](https://huggingface.co/learn/llm-course/en/chapter2/4)
## Intro
* **goal**: turn text into <span style="color:rgb(255, 0, 0)">meaningful numeric representation</span>
* **What does tokenization do?**
	<span style="color:rgb(255, 0, 0)">turn text into integer IDs</span>, so that ML model (eg. NNET) can digest, process them
- **Does token has semantic meaning?**
	No, token ID are just indices. Need an extra embedding model/layer to interpret them 
	(modern ways: train the embedding layer as **built-in of the LLM**)

$$  
\text{Raw Text}  
\xrightarrow{\text{Tokenizer}}  
\text{Token IDs}  
\xrightarrow{\text{Embedding layer or model}}  
\text{Vector Embeddings}  
$$
Eg.
$$
\text{"Hello world"}
\xrightarrow{\text{Tokenizer}}
[15496,\, 995]
\xrightarrow{\text{Embedding Lookup}}
\begin{bmatrix}
0.21 & -0.83 & 0.44 & \dots \\
0.09 & 0.51 & -0.77 & \dots
\end{bmatrix}
\xrightarrow{\text{forward next NNET layer}}
$$

### terms
* **token**: <span style="color:rgb(255, 0, 0)">basic unit</span> used to convert text into numbers for an LM
		* each token is probability distribution representation of unicode strings (in int format)
		* eg. `some_word_indices = [15496, 11, 995, 0]`
* **encode**: turn strings to tokens
* **decode**: turn tokens back to strings
* tokenizer: a class that implements encode & decode methods
* <span style="color:rgb(255, 0, 0)">vocabulary size</span>: # of unique tokens the tokenizer can recognize
	* nrow of the vocabulary table (unique count of what ONE index can be)
	* eg. `<pad>`, `10`, `1032` -> each is a vocabulary
* <span style="color:rgb(255, 0, 0)">Embedding Dimension</span>-- length of token that represent one word (always the same for same tokenizer)
* corpus-- massive input text data used for training tokenizer
* commonly seen special token (serve special purpose, have same embedding dim)
	* `BOS`: beginning of sequence
	- `CLS`: classification token, beginning of sentence (special token for BERT)
	- `PAD`: padding (for making each batch input same number of tokens)
	- `EOS`: end of sequence
	- `UNK`: unknown token(== out-of-vocabulary token/OOV)
	- `SEP`: separator between sentences
- Padding side for decoder only structure should be <span style="color:rgb(255, 0, 0)">left</span> $\because$ 
		![[assets/Untitled.png]]

## Common Types 
-  [Tiktokenizer: nice interactive website](https://tiktokenizer.vercel.app/) to visualize tokenization & splitting
- [sentencepiece library: most common tokenization algorithms](https://github.com/google/sentencepiece)
- **compression ratio = $\frac{\text{total bytes of original text}}{\text{total no. of tokens generated}}$**
	- def: measure of how efficiently tokenizer pack raw text into tokens
	- why use "byte" as  unit: finest basic unit, & modern tokenizers start w byte (eg. BPE, wordpiece)
	- higher ratio:
		- compress text into **denser, more meaningful chunks**
		- less space: save training computation
	- low ratio:
		- sequence will be long, bad for self-attention mechanism

| type              | Byte-based                                                                        | Character-based                                                                                                                                                                               | Subword-based                                                                                                                                             | Word-based                                                                                                                               |
| ----------------- | --------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| token unit        | byte                                                                              | a character                                                                                                                                                                                   | decomposed subword                                                                                                                                        | a word                                                                                                                                   |
| Ex.               | `01000001` (8 bit)                                                                | [h,e,',s,v,i,b,i,n,g]                                                                                                                                                                         | [he, ', s, vib, ing]                                                                                                                                      | [he, 's, vibing]                                                                                                                         |
| summary           | promising but still in early stage                                                | suboptimal                                                                                                                                                                                    | <span style="color:rgb(255, 0, 0)">optimal</span> (BPE, wordpiece widely use for modern LLM)                                                              | suboptimal                                                                                                                               |
| vocabulary size   | 256 (0~255)                                                                       | 127758 (inefficient)                                                                                                                                                                          | adapted based on frequency                                                                                                                                | almost unbounded                                                                                                                         |
| compression ratio | 1                                                                                 | 1.5                                                                                                                                                                                           | 1.5~4                                                                                                                                                     | 4                                                                                                                                        |
| pros              | - less sparsity<br>- small vocabulary size<br>- elegant, read by machine directly | - less unknown token (OOV)<br>- can recognize misspelled word<br>- good for ideogram-based language (中文)                                                                                      | - based on corpus statistics<br>- can recognize misspelled word (worst case: split to char-based)<br>- nice balance of vocabulary size & token meaing<br> | - easy setup                                                                                                                             |
| cons              | - long sequence (bad for self-attention)<br>- low compression ratio               | - roman-based language loose a lot of info when break-down to chr)<br>- many characters are rare, but we use a token to represent each of them (inefficient)<br>- vocabulary size quite large | extra training                                                                                                                                            | - huge vocabulary size<br>- unknown token (not seen in training corpus)<br>- need to train every word, even similar ones (eg. dog, dogs) |


