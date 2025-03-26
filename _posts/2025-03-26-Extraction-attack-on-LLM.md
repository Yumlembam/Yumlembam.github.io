---
layout: post
title: "Understanding Extraction Attacks on Language Models through Simple Examples"
date: 2025-03-26
---

# Understanding Extraction Attacks on Language Models through Simple Examples

Large Language Models (LLMs) such as GPT-4, PaLM-2, and others have revolutionized natural language processing by enabling human-like text generation and a myriad of applications. However, behind their impressive performance lies a critical security vulnerability: extraction attacks. Recent research has shown that even black-box models—accessed solely via APIs—can be exploited to recover sensitive internal parameters. In particular, Carlini et al. (2024) demonstrated that an adversary can extract the embedding projection layer (and thus determine the hidden dimension) with relatively low cost.

In this blog, we walk through simplified examples to illustrate these extraction techniques, discuss the underlying methods, and highlight key experimental insights from the paper. We cover:

1. **Identifying the Hidden Dimension using SVD (Singular Value Decomposition)**
2. **Extracting the Projection Layer up to a Transformation (Symmetry)**
3. **Extraction Attacks Using Top-K Logit-Bias APIs**

These methods not only reveal critical model details but also expose vulnerabilities in current API designs, prompting the need for better security measures.

---

## 1. Identifying the Hidden Dimension with SVD

**Concept Overview:**  
Although the output logit vectors of a language model reside in a high-dimensional space, they originate from a lower-dimensional hidden representation. By querying the model repeatedly with different prompts, one can collect output vectors that—despite being high-dimensional—actually lie in an \(h\)-dimensional subspace. Here, \(h\) is the hidden dimension.

**Toy Example:**  
Consider a simple model where:
- **Hidden Dimension (h):** 2  
- **Logit Dimension (l):** 4

We define a projection matrix \(W\) (size \(4 \times 2\)):

$$
W = \begin{bmatrix}
1 & 0 \\
0 & 1 \\
2 & 1 \\
1 & 2
\end{bmatrix}
$$

For three hidden state vectors:

$$
h_1 = [1, 1], \quad h_2 = [2, 0], \quad h_3 = [0, 1]
$$

the corresponding logit vectors are:

$$
Q = W \cdot H = \begin{bmatrix}
1 & 2 & 0 \\
1 & 0 & 1 \\
3 & 4 & 1 \\
3 & 2 & 2
\end{bmatrix}
$$

**Using SVD to Reveal Hidden Dimensionality:**  
By performing Singular Value Decomposition on \(Q\):

$$
Q = U \cdot \Sigma \cdot V^\top
$$

the significant singular values indicate the number of independent directions—i.e., the true hidden dimension. In our toy example, once we exceed two queries, the additional logit vectors become linearly dependent, confirming that the hidden dimension is exactly **2**.

**Algorithmic Insight:**  
The paper introduces a *Hidden-Dimension Extraction Attack* (Algorithm 1) where the adversary:
- Collects \(n\) output vectors (with \(n > h\)) from the model using random prompts.
- Forms the matrix \(Q\) whose columns are these logit vectors.
- Computes the singular values of \(Q\) and identifies a sharp multiplicative gap that reveals \(h\).

This simple yet powerful idea shows that even with black-box access, one can infer internal model characteristics.

---

## 2. Extracting the Projection Layer (Up to a Symmetry)

After determining the hidden dimension, the next step is to recover the projection matrix \(W\) that maps hidden states to the output logits.

**Method Overview:**  
Upon computing the SVD of \(Q\):

$$
Q = U \cdot \Sigma \cdot V^\top
$$

it turns out that the product $$U \cdot \Sigma$$ approximates the projection matrix \(W\) up to an unknown transformation \(G\):

$$
U \cdot \Sigma = W \cdot G
$$

This means the extraction recovers \(W\) only up to an internal symmetry (an affine transformation, typically a rotation and scaling) of the hidden space.

**Experimental Findings:**  
- The paper shows that by aligning the extracted matrix $$U \cdot \Sigma$$ with the actual \(W\) (via a least-squares approach), the reconstruction error can be extremely low—on the order of \(10^{-4}\) RMS error.
- Such high-fidelity recovery confirms that the output layer is low-rank and that its structure is susceptible to extraction attacks.

**Implications:**  
Reconstructing \(W\) not only provides the hidden dimension size but also yields insights into the model’s architecture, such as the overall width and potential parameter count. This information is valuable for an attacker and challenges the assumption that API access alone sufficiently protects internal model details.

---

## 3. Extraction Attacks Using Top‑K Logit‑Bias APIs

**Practical Challenges:**  
In real-world settings, production APIs typically do not return the full logit vector. Instead, they offer:
- **Top‑K Token Probabilities:** Only the probabilities (or log‑probabilities) for the top few tokens.
- **Logit Bias Options:** A mechanism to adjust token probabilities by applying a bias to selected tokens.

**Attack Technique:**  
The paper outlines a method to overcome these restrictions:
- **Logit Bias Exploitation:** By applying a large bias \(B\) to a specific token, an attacker can force that token into the top‑K results even if its original logit is low.
- **Sequential Querying:** The adversary issues a series of queries with different bias configurations. For instance, with a top‑5 API, one can cycle through the vocabulary by biasing different groups of tokens, thereby reconstructing the entire logit vector.
- **Relative Comparisons:** Since the API returns log‑probabilities rather than raw logits, the attack involves comparing the differences between a reference token and other tokens. This process cancels out the additive constant introduced by the softmax normalization.

**Toy Example:**  
Consider a toy vocabulary of 5 tokens: A, B, C, D, and E. Suppose that for a given prompt the true (hidden) logits are:

- $$z_A = 3.0$$
- $$z_B = 1.0$$
- $$z_C = 0.5$$
- $$z_D = -1.0$$
- $$z_E = -2.0$$

Without any bias, the API might return only the top‑2 tokens (A and B) with their log‑probabilities. Now, to recover the logit for token D (which normally wouldn’t appear), an attacker applies a large bias, for example:

$$
B = 100.
$$

This boosts token D’s effective logit:

$$
z'_D = z_D + B = -1.0 + 100 = 99.0.
$$

In the biased query, the API now returns token D along with another high‑value token (say, token A). Let the returned log‑probabilities be $$y_D^{(B)}$$ and \$$y_A^{(B)}$$. Because softmax is invariant to additive constants, the difference

$$
y_A^{(B)} - y_D^{(B)} - B
$$

approximates the original difference $$z_A - z_D$$. For our values:

$$
z_A - z_D = 3.0 - (-1.0) = 4.0.
$$

By repeating such queries with large biases applied to other tokens, the attacker can determine the relative differences between all tokens’ logits. Stitching these differences together allows the full logit vector to be reconstructed (up to an overall additive constant).

**Cost and Efficiency:**  
- The attack is shown to be cost-effective. For instance, the paper estimates that extracting the full projection layer for models like OpenAI’s ada or babbage costs only a few dollars.
- Detailed analysis in the paper compares token costs and query costs, demonstrating that even under constrained API conditions, the attack remains practical.

---

## Conclusion

The work presented in *"Stealing Part of a Production Language Model"* offers a compelling demonstration of how extraction attacks can breach the secrecy of black-box LLMs. By:

- **Identifying the hidden dimension via SVD,** adversaries can exploit the low-rank structure of the final projection layer.
- **Recovering the projection matrix up to an affine transformation,** significant internal model details can be extracted.
- **Leveraging restricted top-K logit-bias APIs,** even production systems with seemingly secure interfaces are vulnerable.

These findings have profound implications for model security and intellectual property protection. They underscore the importance of revisiting API designs such as restricting or modifying logit bias capabilities, adding calibrated noise to outputs, or altering architectural designs to mitigate the risks posed by such attacks.

Understanding these vulnerabilities is crucial for researchers and practitioners alike, as it guides the development of safer, more robust machine learning systems in an era where models are both powerful and widely accessible.

---

*References:*  
Carlini, N. et al. (2024). *Stealing Part of a Production Language Model*.  
Additional literature on model extraction and adversarial machine learning.
