**GQA (Grouped Query Attention)**

- GQA is a more memory-efficient variant of multi-head attention that reduces the number of key-value heads while maintaining the same number of query heads. 

- In standard MHA: num_kv_heads = num_heads
  
  In GQA: num_kv_heads < num_heads (typically num_heads // 2 or num_heads // 4)
  
  In MQA (Multi-Query Attention): num_kv_heads = 1

- Each KV head is shared across multiple query heads.

- How it works:
  - For projection matrices:
    Q: (d_model, d_model)

    K: (d_model, num_kv_heads * head_dim)
    
    V: (d_model, num_kv_heads * head_dim)

    where head_dim = d_model // num_heads (in this case, query heads)
  
  - Split it into (B, num_heads, seq_len, head_dim) as usual for Q.
    
    For K and V, split into (B, num_kv_heads, seq_len, head_dim)

    Repeat K and V equal to the number of times required to get equal to Q heads. (equivalent to num_heads / num_kv_heads)

    Convert it back to (B, num_heads, seq_len, head_dim)

    Shape progression:
      
      (batch, num_kv_heads, seq_len, head_dim)
      
      (batch, num_kv_heads, 1, seq_len, head_dim)
      
      (batch, num_kv_heads, num_queries_per_kv, seq_len, head_dim)
      
      (batch, num_heads, seq_len, head_dim)
  
- This reduces the KV cache size during inference, making it more efficient for deployment.

---

**Positional Embeddings**
- Applied to only **Q** and **K** matrices, not V.

- Sinusoidal Embeddings
  - Sine and cosine functions are applied across alternate dimensions across the hidden dimension.

  Formulae:
  $$PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{\frac{2i}{d_{model}}}}\right)$$
  
  $$PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{\frac{2i}{d_{model}}}}\right)$$

  - This positional encoding is then added to each token's embedding.

  - Potential disadvantage of sinusoidal embeddings is that once these are added, the model struggles to understand the relative distance between two words.

- RoPE (Rotary Positional Embeddings)
  - As the name suggests, it "rotates" the embedding vector for a token.

  - The further along the sequence a word is, the more it is rotated.

  - The entire vector is **not** rotated all at once. Instead, it is paired up in chunks of two, and each chunk is rotated by a specific angle.

  - If $m$ is the absolute position of the word, and $\theta$ is a set frequency angle, the rotation matrix for position $m$ looks like this:
  
  $$R_m = \begin{pmatrix} \cos(m\theta) & -\sin(m\theta) \\ \sin(m\theta) & \cos(m\theta) \end{pmatrix}$$

  - We apply this rotation to the Query at position $m$, and the Key at position $n$:
  
    Rotated Query: $q_m = R_m q$ 
    
    Rotated Key: $k_n = R_n k$

  - It is applied **during** attention. To be more precise, before the $QK^T$ calculation.

  - The rotation angle for any specific chunk $i$ is calculated as:
  
    $$\text{Angle}_i = m \cdot \theta_i$$
  
    The $\theta$ values decay exponentially as we move down the vector. The formula used for chunk $i$ (where $d$ is the total dimension size) is:
    $$\theta_i = 10000^{\frac{-2i}{d}}$$

---

**Activations**
- GeLU: Gaussian Error Linear Unit (GELU) activation function.
  - GELU is a smooth approximation to ReLU that weights inputs by their magnitude rather than gating them by their sign.

  - Exact formula:
    GELU(x) = x * Φ(x)
    where Φ(x) is the cumulative distribution function of the standard normal distribution.

  - Approximation (used in practice):
    GELU(x) ≈ 0.5 * x * (1 + tanh(sqrt(2/π) * (x + 0.044715 * x^3)))

  - Smooth and differentiable everywhere
  - Non-monotonic (has a small negative region)
  - Better gradient flow than ReLU
  - Empirically performs better in many transformer models

- SiLU: Sigmoid Linear Unit (SiLU), also known as Swish activation function.
  - Formula:
    SiLU(x) = x * sigmoid(x) = x / (1 + exp(-x))  
  - Smooth and non-monotonic
  - Self-gated (uses its own value for gating)
  - Unbounded above, bounded below
  - Better gradient flow than ReLU
  - Similar performance to GELU in practice

- GLU: Gated Linear Unit (GLU) activation function.
  - GLU splits the input into two halves and uses one half to gate the other.

  - Formula:
    GLU(x) = (x * W + b) ⊗ σ(x * V + c)
    
    where ⊗ is element-wise multiplication and σ is sigmoid.

  - In practice, the input is already split, so:
    GLU(a, b) = a ⊗ σ(b)

  - Provides a gating mechanism
  - Can control information flow
  - Used as part of feed-forward networks

  - This is typically used in combination with linear layers in the FFN.
---

**FeedForwardNetwork**

GLU FFN
- Gated Linear Unit (GLU) Feed-Forward Network.

- GLU variants use a gating mechanism to control information flow. They have been shown to improve performance in language models.

- GLU(x) = (xW1 + b1) ⊗ activation(xW2 + b2)
  
  where ⊗ is element-wise multiplication.

- In practice, we can compute both projections together and split:
  
  GLU(x) = Linear(x)[:, :d_ff] ⊗ activation(Linear(x)[:, d_ff:])

- Structure:
  
  Linear(d_model -> 2*d_ff) -> Split -> [Identity, Activation] -> Multiply -> Linear(d_ff -> d_model)

MoE (Mixture of Experts)
- In a standard transformer, every parameter is used to get the output.

- In MoE, **total** parameters is decoupled from **active** parameters. Meaning, even if the total parameters stored **on disk** does not change, the number of parameters with which compute is happening is **reduced**.

- It works by replacing the FFN by multiple smaller FFNs or "experts".

- There is a single "router" that decides to which expert(s) the token will be sent.

- Usually only top k (1/2) experts are chosen.

- The output $y$ for a given input token $x$ is the sum of the expert outputs $E_i(x)$, multiplied by the router's gating probability $G(x)_i$:
$$y = \sum_{i=1}^{n} G(x)_i E_i(x)$$

- Steps for calculation:
  - Input matrix is passed through a linear layer with dimension (d_model, num_experts).

  - Based on the logits of the layer, we get the topk logits and indices.

  - We normalize the logits to use them as probabilities of the router.

  - For each expert, we pass the token through it to get the output. Each expert is made of the same ffn, which is (d_model, d_ff), where d_ff is the expanded dimension and then (d_ff, d_model).

- Load balancing loss is applied so that the router does not choose a single expert every time. It is calculated by multiplying the frequency and the probability together for each expert, summing them all up, and multiplying by the total number of experts ($N$).

$$L_{balance} = \alpha \cdot N \sum_{i=1}^{N} f_i \cdot P_i$$

---

**Normalization**
- LayerNorm
  - LayerNorm normalizes the inputs across the feature  dimension for each sample independently. It computes the mean and variance across all features for each sample in the batch.

  - Formula:
    
    y = (x - mean) / sqrt(var + eps) * gamma + beta

    where:
    mean = mean(x) across feature dimension
    
    var = variance(x) across feature dimension
    
    gamma, beta = learnable parameters (scale and shift)
    
    eps = small constant for numerical stability

  - Normalizes each sample independently (unlike BatchNorm)
  
  - Works well with variable sequence lengths
  
  - Helps with gradient flow in deep networks

- RMSNorm
  - RMSNorm is a simpler and more efficient variant of LayerNorm that only uses the root mean square (RMS) for normalization, without centering (no mean subtraction).

  - Formula:
    
    y = x / RMS(x) * gamma

    where:
    
    RMS(x) = sqrt(mean(x^2) + eps)
    
    gamma = learnable scale parameter
    
    eps = small constant for numerical stability

  - Simpler computation (no mean calculation and subtraction)
  
  - Fewer parameters (no bias term)
  
  - Faster training and inference
  
  - Similar or better performance in practice

- PreNorm
  $$x_{l+1} = x_l + \text{SubLayer}(\text{LayerNorm}(x_l))$$
  Residual is added after normalizing the layer output.

  At initialization, this architecture keeps the variance of the gradients stable across all layers.

- PostNorm
  $$x_{l+1} = \text{LayerNorm}(x_l + \text{SubLayer}(x_l))$$
  Every residual connection has to pass through a layer normalization.

  Expected gradients at the start are expected to be large, which is why a warm up is required for the learning rate. The LN is started near zero and then slowly increased with number of iterations.

- SFT (Supervised Fine-Tuning)
  
  - Place in lifecycle:
    - Pre-training: Read the internet (Learn language and facts).

    - **Supervised Fine-Tuning (SFT)**: Read human-written Q&A pairs (Learn how to be an assistant).

    - Alignment (RLHF / DPO): Learn from human feedback (Learn what makes a great answer versus an okay answer, and learn to reject unsafe prompts).
  
  - LoRA (Low Rank Adaptation)
    - Keep the base model frozen and train two smaller matrices.

    - There are usually added alongside Q and K matrices.

      For example, $A$ and $B$, based on a small "Rank" ($r$), like $r=8$:
      
      Matrix $A$: $4096 \times 8$
      
      Matrix $B$: $8 \times 4096$

    - Consider input $x$. On entering the transformer, it is multiplied by $Q$ to generate the query vector.

    - At the same time, $x$ is multiplied by $A$ first and then $B$. Because it first ranks down and then ranks up to the original dimension, the output shape is same as the input.

    - This multiplication result is then added back to the Query vector.

      $Q_x$ = $x.Q$ + $(x.A)B$
    
    

