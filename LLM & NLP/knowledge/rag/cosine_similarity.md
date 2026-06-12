# Cosine Similarity

**Definition:** the **angle** between two vectors — direction only, magnitude ignored.

**Use case:** most common scoring method for **dense retrieval** — rank documents by similarity to the query embedding & return top-k.

## Intuition

**Direction = meaning** Each dimension is a **latent feature**. A vector's direction is its *mix of features*. Cosine asks: *do these two items emphasize the same features in the same proportions?* Aligned feature patterns → higher similarity. 

**Magnitude = length/repetition.** A doc saying `"cat"` and one saying `"cat cat cat cat"` point the same direction (same topic) but differ in magnitude (length). Cosine ignores the magnitude → judges them identical.

## The math

```
              a · b            Σ aᵢbᵢ
cos(a,b) = ───────────── = ──────────────────────
            ‖a‖ · ‖b‖      √(Σaᵢ²) · √(Σbᵢ²)
```

- **Numerator** = dot product (element-wise product, summed).
- **Denominator** = product of the two vectors' **magnitudes** (Euclidean norms = √sum-of-squares). 

Range:
```
 cos =  1  → same direction (most similar)
 cos =  0  → perpendicular (unrelated)
 cos = -1  → opposite direction
```
For embeddings (often anisotropic / non-negative), values usually sit in `[0, 1]`.

**Shortcut:** if vectors are normalized (`‖·‖ = 1`, as many embedding models output), the denominator is 1, so `cos(a,b) = a · b` — cosine is just the dot product. This is why vector DBs can store normalized vectors and use plain dot product.

## Toy calculation

```
a = [1, 0],  b = [1, 1]

dot = 1·1 + 0·1 = 1
‖a‖ = √1 = 1
‖b‖ = √2 = 1.414

cos = 1 / (1 · 1.414) = 0.707   → 45° apart
```

Magnitude is ignored — `a=[1,2,3]` and `b=[2,4,6]` (b = 2a) give `cos = 1` even though `b` is twice as long, because they point the same way.

## What cosine actually captures: distributional similarity

A high cosine means **"related / same context,"** not strictly "same meaning."

Embedding models learn by **predicting text from context** — BERT fills a masked blank, GPT predicts the next token, Word2Vec predicts neighbors. So texts in **similar contexts** get **similar vectors** — the **distributional hypothesis** ("know a word by the company it keeps").

**Antonym trap:** "hot" and "cold" have *high* cosine, not negative. They fill the same blanks (`"the coffee is too ___"`), so they sit close. Negative cosine does **not** reliably mean "opposite meaning," and is rare in practice (spaces are anisotropic — even unrelated pairs are mildly positive). **Rely on relative ranking, not the absolute value/sign.**

## From distributional → semantic (fine-tuning)

| Stage | Objective | What cosine captures |
|---|---|---|
| Base pretraining | predict from context | distributional / contextual (antonyms close) |
| Retrieval fine-tuning | **contrastive** on pairs (query↔answer, paraphrases) | nudged toward task-relevant **semantic** similarity |

The semantic sharpening comes from the **fine-tuning stage on labeled pairs**, layered on top of the base model — it never fully escapes the distributional foundation.
