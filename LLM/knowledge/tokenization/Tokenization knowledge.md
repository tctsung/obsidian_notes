---
created: 2025-08-19T22:45
updated: 2025-08-20T08:47
---
* [ref huggingface](https://huggingface.co/learn/llm-course/en/chapter2/4)

## Intro
* purpose: turn text into <span style="color:rgb(255, 0, 0)">meaningful numeric representation</span>
	* $\because$ ML model cannot ingest string, they are train on numbers
* terms:
	* <span style="color:rgb(255, 0, 0)">tokenization</span>-- chopping long text into pieces (still text format, but token unit)
	* <span style="color:rgb(255, 0, 0)">encoding</span>-- turn text piece to numeric vector
	* token-- a basic unit for tokenized text
	* vocabulary size-- total number of unique tokens the tokenizer can recognize
	* Embedding Dimension-- length of token that represent one word (always the same for same tokenizer)
	* corpus-- massive input text data used for training tokenizer
* commonly seen special token (serve special purpose, have same embedding dim)
	* `BOS`: beginning of sequence
	- `CLS`: classification token, beginning of sentence (special token for BERT)
	- `PAD`: padding (for making each batch input same number of tokens)
	- `EOS`: end of sequence
	- `UNK`: unknown token(== out-of-vocabulary token/OOV)
	- `SEP`: separator between sentences
- Padding side for decoder only structure should be <span style="color:rgb(255, 0, 0)">left</span> $\because$ 
		![[Untitled.png]]
## Common Types

| type       | Word-based                                                                                                          | Character-based                                                                                                                                                                                         | Subword tokenization |
| ---------- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| token unit | a word                                                                                                              | a character                                                                                                                                                                                             | decomposed subword   |
| pros       | - easy setup                                                                                                        | - small vocabulary size<br>- less OOV/unknown<br>- can recognize misspelled word<br>- good for ideogram-based language                                                                                  |                      |
| cons       | - huge vocabulary size<br>- <UNK> if not in corpus<br>- need to train every word, even similar ones (eg. dog, dogs) | - very small unit lead to:<br>a. limited input info in one context window<br>b. huge input data len<br>- each unit is less meaningful (roman-based language loose a lot of info when break-down to chr) |                      |
| Ex.        | [he, 's, a, cool, guy]                                                                                              | [h, e , ', s, c, o, o, l]                                                                                                                                                                               |                      |

* Other common types
