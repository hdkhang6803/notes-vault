---
Date: "2026"
Authors: Weinuo Ou
Venue: arxiv
Paper: "Auxiliary-predicted Compress Memory Model(ApCM Model): A Neural Memory Storage Model Based on Invertible Compression and Learnable Prediction"
Memory type:
  - Latent
Agent env: Multi-agent
Record format: Text
Memory architecture:
  - 3-tier
  - graph-a
Tackle Module: Memory design
Need offline initialization: false
Fine-tuning?: false
Other tags:
---
# 1. Terminology
| Term                                                     | Definition                                                                                                                                                                                                        |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IDRP (Invertible Dimensionality Reduction and Predictor) | Core sub-module combining an invertible encoder with a learnable predictor to compress and reconstruct data.                                                                                                      |
| Affine Coupling Layer                                    | Invertible transformation layer that splits input $x \in \mathbb{R}^d$ into halves $x_1, x_2 \in \mathbb{R}^{d/2}$, and scales/translates $x_2$ conditioned on $x_1$​: $y_1 = x_1$, $y_2 = x_2 \odot \exp(s) + t$ |
| PermuteLayer                                             | Fixed random permutation applied after each coupling layer to mix dimensions.                                                                                                                                     |
| $z_{comp}$​                                              | The compressed latent sub-vector,$z_{comp} \in \mathbb{R}^m$, that is actually stored in memory.                                                                                                                  |
| $z_{aux}$                                                | The discarded (not stored) latent sub-vector, later estimated by the predictor.                                                                                                                                   |
| Auxiliary Predictor $g_\phi$                             | Lightweight MLP (Linear → SwiGLU → Linear) that estimates $z_{aux}$​ from $z_{comp}$.                                                                                                                             |
| Memory Bank $M$                                          | Global slot-based store, $M \in \mathbb{R}^{max\_mem \times m}$, each row holds one $z_{comp}$​ vector.                                                                                                           |
| Read Mechanism                                           | Cosine-similarity search over $M$ to retrieve the closest stored slot to a query.                                                                                                                                 |
| Write Mechanism (AFF ctrl policy)                        | "First idle, then least-frequently-used" slot-selection policy for writing new memories.                                                                                                                          |
| $f_\theta$​ / $f_\theta^{-1}$                            | Forward and inverse pass of the invertible encoder network.                                                                                                                                                       |

---

# 2. Method Summary (What)

ApCM decouples memory _storage_ from memory _reconstruction_:
- An invertible neural network compresses input data into a latent vector, split into a stored part ($z_{comp}$) and a discarded part ($z_{aux}$). 
- A lightweight predictor learns to infer the discarded part from the stored part, so reconstruction quality hinges on how much information $z_{comp}$ retains about $z_{aux}$. 
This IDRP module is paired with a slot-based Memory Bank supporting similarity-based reads and frequency-based writes, forming a complete runtime memory system for AI/LLM use.

---

# 3. What it Solves (Why)

Prior compression/memory approaches suffer from rigid, non-learnable trade-offs between storage cost and reconstruction fidelity:
- **PCA / linear compression:** linear assumptions limit modeling of complex, nonlinear data distributions; compression is lossy with no learnable mechanism to recover what's discarded.
- **Key-Value memory (full storage):** avoids information loss by storing everything, but at the cost of large storage dimensionality (no compression benefit).
- **Stateless LLM parameters:** all knowledge is frozen at training time, with no mechanism for runtime storage, retrieval, or update of new information.

**=> ApCM's fix:** split the latent into a stored component and a _predictable_ discarded component, then train a lightweight network to reconstruct the discarded part from the stored part, turning lossy compression into a learnable, optimizable process instead of a fixed linear trade-off.

---

# 4. Methodology (How)

> **Running example:** an input image patch $x$ is fed through the pipeline below.

## **System / Memory structure**

1. Invertible Network Encoder $f_\theta$ maps $x$ to a latent $z = f_\theta(\text{flatten}(x))$ via $N$ stacked affine coupling layers + random permutations.
    - Each coupling layer splits its input into $x_1, x_2$
    - a sub-network (Linear → SwiGLU → Residual blocks → Linear) computes $(s, t)$ from $x_1$
    - outputs $y_1 = x_1$, $y_2 = x_2 \odot \exp(s) + t$ --> exactly invertible $x_2 = (y_2 - t) \odot \exp(-s)$.
    - Random Permutation layers ensure all dimensions eventually interact across layers.
2. Latent $z$ is split: $z = [z_{comp}, z_{aux}]$.
    - $z_{comp} \in \mathbb{R}^m$ is retained for storage.
    - $z_{aux}$ is discarded entirely.
3. Predictor $g_\phi$ (Linear → SwiGLU → Linear) is trained to estimate $\hat{z}_{aux} = g_\phi(z_{comp})$.
4. The Memory Bank $M \in \mathbb{R}^{max_mem \times m}$ holds one $z_{comp}$ per slot, each with an access-frequency counter (AFF ctrl).

## **Retrieval flow (Read)**

1. Encode query $x$ → $q = z_{comp}$.
2. Compute cosine similarity between $q$ and every stored slot vector in $M$.
3. Select the slot $M_{i^*}$ with highest similarity.
4. Predict $\hat{z}_{aux} = g_\phi(M_{i^*})$, concatenate $[M_{i^*}, \hat{z}_{aux}]$, pass through $f_\theta^{-1}$ to reconstruct $\hat{x}_{mem}$.
5. Increment the access-frequency counter of the retrieved slot.

## **Updating flow (Write)**

1. Encode a batch of inputs ${x_j}$ to get ${z_{comp}^{(j)}}$; compute mean $\bar{z}$ as the write vector.
2. Slot selection follows "first idle, then least frequently used":
    - Prioritize any slot with AFF ctrl == 0 (unused).
    - Otherwise, overwrite the slot with the lowest access frequency.
3. Write $\bar{z}$ into the chosen slot and reset its access counter to 0.

> Applied to the running example: the image patch's $z_{comp}$ is written into an empty slot on first encounter; on a later query with a similar patch, cosine similarity retrieves that slot, $g_\phi$ predicts the missing $z_{aux}$, and $f_\theta^{-1}$ reconstructs a close approximation of the original patch.

Training jointly optimizes $f_\theta$ and $g_\phi$ end-to-end by minimizing reconstruction loss.

---

# 5. Benchmarks

**5.1 Other baselines**

- **PCA**: linear dimensionality reduction, used as the primary compression baseline.
- **Key-Value Memory Network**: full (uncompressed) storage baseline for the random-data comparison.

**5.2 Benchmarks**

|Setting|Data|Metrics|
|---|---|---|
|Synthetic-train / Real-test|Image patches|PSNR, MAE, MSE|
|Real-train / Real-test|Image patches|PSNR, MAE, MSE|
|Random-data comparison vs. Key-Value Memory|Random vectors|MSE, MAE, storage dim, inference time|
|Ablation (no $z_{aux}$ prediction)|Image patches|Qualitative — reconstruction ≈ random noise|

**5.3 Notable Results**

- Synthetic-trained IDRP tested on real images: PSNR ~13.5–17.9 dB — worse than PCA's ~27–30 dB in this mismatched setting.
- Real-trained IDRP tested on real images: PSNR jumps to ~39–44 dB, clearly surpassing PCA's ~27–30 dB, confirming the nonlinear modeling advantage when train/test distributions match.
- vs. Key-Value Memory (random data): ApCM gets lower MSE (0.987 vs 1.001) and MAE (0.766 vs 0.799) using only 128 storage dims vs. 1024, but is 180x slower at inference (0.18s vs 0.001s).
- Ablation: removing $z_{aux}$ prediction makes reconstructions indistinguishable from random noise, confirming the predictor is load-bearing.

---

# 6. Strengths

- **Storage efficiency:** achieves comparable or better reconstruction accuracy than Key-Value Memory using 8x fewer storage dimensions.
- **Exact invertibility:** the coupling-layer encoder is mathematically lossless in the forward direction, so all reconstruction error is cleanly attributable to the predictor.
- **Strong nonlinear modeling:** substantially outperforms PCA when train/test data distributions align.
- **Complete memory system:** cosine-similarity read + frequency-based write eviction makes this a usable end-to-end memory module, not just a compression technique.
- **Clean ablation:** directly demonstrates the predictor's necessity by showing its removal collapses reconstruction to noise.

---

# 7. Gaps

- **Inference latency:** 180x slower than the Key-Value Memory baseline, a significant obstacle for real-time or production LLM deployment.
- **Distribution sensitivity:** synthetic-train/real-test results underperform even PCA, showing fragile generalization under train/test distribution shift.
- **No LLM-scale validation:** all experiments use image patches or random vectors; no test on text/token data, long-context tasks, or actual LLM integration.
- **No uncertainty quantification:** the paper names "structural noise" from predictor error as a concern but doesn't measure or bound it experimentally.
- **Limited evaluation scale:** only 4 test images per setting, no variance/statistical significance reporting.
- **Proposed future gaps:** 
	- (1) robustness studies under deliberate distribution shift;
	- (2) latency optimization or approximate retrieval methods to close the speed gap with Key-Value Memory; 
	- (3) formal capacity-vs-compression-ratio analysis for the Memory Bank; 
	- (4) end-to-end integration study measuring downstream LLM metrics (perplexity, long-context QA accuracy) rather than only reconstruction fidelity.

---

# 8. Highlights

_(left blank)_