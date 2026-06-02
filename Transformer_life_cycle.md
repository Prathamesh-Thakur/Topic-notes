# Life cycle of text generation in Decoder only Transformers

**Dimension notations:**

- B: Denotes the batch dimension, which indicates the number of sequences being processed at once.

- seq_len: Denotes the sequence length, which indicates the length of the sequence of tokens being processed

- W: Denotes the width of a single element of the batch, which is also equivalent to the hidden dimension or the length of the vector for a single token's embedding.

- vocab_size: The total size/length of the vocabulary, which is equal to the number of unique tokens after tokenization.

- n_heads: Number of attention heads

- head_dim: The number of parts of the hidden dimension being processed by a head. (W / n_heads)

---

Input: A matrix of shape (B, seq_len, W).

Steps:

Tokenization -> Encoding -> Embedding -> Decoder blocks -> Decoding

Decoder block:

Self Attention -> FeedForward Neural Network -> Residual

After the last decoder block, the output is passed through a ffn which has shape (W, vocab_size) with a softmax activation, which creates a probability distribution over the entire vocabulary. It represents the likeliness of any token being the next token.

---

**Tokenization**

- The text is broken into tokens or indivisible units.

- Different strategies include by word, sentence or character. Most famous is Byte Pair Tokenizer.

- All unique tokens make up the vocabulary for the model, which simply represents the choices provided to the model for generation.

---

**Encoding** 

Each token in the vocabulary is mapped to a number.

---

**Embedding** 

- The encoded tokens are mapped to a dense vector representation. Common embedding dimensions include 512, 276. 

- After embedding the tokens, a positional embedding is also added to indicate the position of the token in the sequence. Types include sinusoidal, RoPE.

---

**Self attention**

Formula: $$softmax(\frac{QK^T}{\sqrt{d_{k}}})V$$

Input and output dimensions are equivalent to the global input dimensions.

$d_{k}$ represents the head_dim (number of columns per head). It is a scaling factor which makes sure the values do not explode.

Input matrix is projected into three matrices:
- Query(Q): Represents the current input prompt token being processed
- Key(K): Represents what does this represent (grammatically, positionally)
- Value(V): Represents the actual content of the token

For projection, a single linear layer is used with dimension (W, W).

After projection, all the matrices are split across attention "heads". Each "head" may be considered as processing different parts of the sequence, like syntax, grammar, etc.

- Dimension changes: (B, seq_len, W) -> (B, seq_len, n_heads, head_dim) -> (B, n_heads, seq_len, head_dim)

- The middle two dimensions are transposed so that all tokens are processed across all attention heads.

- $Q.K^T$: (B, n_heads, seq_len, head_dim) @ (B, n_heads, head_dim, seq_len) -> (B, n_heads, seq_len, seq_len)

  After this multiplication, a lower triangular matrix is applied, in which all entries above the diagonal are replaced with -inf. This prevents the model from seeing future tokens.

  Apply softmax across the last dimension. All -inf values get converted to zero.

- Multiplying by V: (B, n_heads, seq_len, seq_len) @ (B, n_heads, seq_len, head_dim) -> (B, n_heads, seq_len, head_dim)

- Transpose the dimensions back, and compress it to 3d to get (B, seq_len, W).

---

**Feed Forward Neural Network**

- The attention matrix is then passed through an MLP.
- It first expands it's dimensions, and then compresses it down.
- Expansion layer has shape (W, expanded_dim), and compression layer has shape (expanded_dim, W).

- Layer Normalization in ffn
  - To make sure that the activations do not explode, a normalization strategy is applied across the hidden features or across the W dimension.
  - It is applied to both the inputs and outputs of the layers.

- **After** the layer progression, the original input matrix is added to the output. It is also known as a residual/skip connection. It is used to make sure the output remains connected to the original prompt in deeper networks.

---

**Prediction**

- After the prompt passes through all the decoder blocks, the final vector is chosen for prediction of the next token.

- Shape: (1, seq_len, W)
The index of this will be the last index in every mini batch of tokens.

- This vector is then multiplied by a linear layer of shape (W, vocab_size) to get logits for each word in the vocabulary. After applying softmax, they get converted to probabilities.

- Depending on whether the mode is training or inference, the next token can be chosen either as the one having the max probability or through other strategies like multinomial sampling coupled with top_k/top_p.

- If training, the cross entropy loss is calculated, and backpropogation occurs.

- The predicted token is then concatenated to the input, and is again passed through the transformer.

---

**Decoding**
- The token which is chosen, the index of it is mapped to the vocabulary, and the specific token id is retrieved.

- This is then mapped to the original token through reverse tokenization.

---

**KV Caching**
- For autoregressive models, since the output of the attention matrix is proportional to sequence length, it can get computationally expensive.

- So we do KV caching.

- After the next token is predicted, we treat only that single token as the query.

- The K and V representations for this single token are simply **concatenated** to the end of the already calculated K and V matrices.

- This works because re calculation of attention values for the past tokens will waste computation cycles.

- Instead, we only calculate scores for the newly generated token.