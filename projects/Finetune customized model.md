---
created: 2025-12-03T21:01
updated: 2025-12-10T12:32
---

Use Colab/RTX2070 + llama-factory to train a < 7B model / RAG a API
### TODO
**data preparation**
1. backend dataset
	- topic dictionary (definition of terms)
	- translator (En/Tw of specific terms)
	- vector database (lyrics with metadata)
2. finetune dataset
	- persona of PCT (read paper to prep)
	- Q&A (answer by lyrics)

midterm goal:
- publish on huggingface
- public API to call (use Oracle)
---
**Model/Agent preparation**
1. RAG
2. finetune: SFT + DPO
3. finetune: ORPO

Midterm goal: 
1. publish on huggingface: 
	1. reproducible Colab tutorial
2. agent workflow using Gemini
---

### Application
- chatbot in my personal website
- tutorials
- streamlit UI for agent (song rec)

### Papers
- [LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/pdf/2106.09685)
- [Training language models to follow instructions with human feedback](https://arxiv.org/pdf/2203.02155)
- [ZEPHYR: DIRECT DISTILLATION OF LM ALIGNMENT](https://arxiv.org/pdf/2310.16944)
	- learn the golden standard (SFT -> DPO)
- [RoleLLM: Benchmarking, Eliciting, and Enhancing Role-Playing Abilities of Large Language Models](https://arxiv.org/pdf/2310.00746)
	- learn a special case of SFT for persona
- [ORPO: Monolithic Preference Optimization without Reference Model](https://arxiv.org/pdf/2403.07691)
	- advanced concept (low priority)

### Long term interests
- multimodal
- sound/music model