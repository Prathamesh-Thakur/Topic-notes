# vLLM
## What is vLLM?
- vLLM is an open-source library designed specifically for fast, high-throughput, and memory-efficient inference and serving of Large Language Models (LLMs).
- It enables you to run any LLM locally on your own compute.
- Since it runs locally, it can only work with **open-weights** models, or models whose weights are publicly available for use.

## Phases of inference:
1. Prefill Phase
- This is the very first prompt in the chat, where the Q, K and V matrices have to be created from scratch.
- This is compute bound due to the large matrix multiplications which need to be carried out.
- The K and V matrices computed during this step are then cached to create the KV cache for subsequent runs.

2. Decode Phase: 
- This is the actual autoregressive phase of the LLM.
- This phase is Memory-Bandwidth Bound, since there is a lot of memory movement during this phase.

## KV Cache:
Previously computed K and V matrices are stored in GPU cache so that they need not be recomputed from scratch for every new token. Instead, each new token's K and V vector are appended to these matrices.

## Why is it needed?
- The biggest breakthrough which vLLM provides is **PagedAttention**.
- Previously, there used to be a bottleneck. Since the KV cache needed to be read fast during the Decode Phase, it used to be stored as a **contiguous** block of memory for a single user.
- However, the actual length of the answer itself was unknown. So, the maximum length used to be reserved for every user.
- But in cases where the answer was very short in length, the remaining memory used to remain unused, leading to fragmentated usage of memory.
- vLLM solved this problem.

### PagedAttention
- Instead of demanding one giant contiguous block of VRAM, PagedAttention breaks the KV cache down into tiny, fixed-size chunks called Blocks.
- Fixed-Size Blocks: Each block is designed to hold the Keys and Values for just a handful of tokens (usually 16).
- Physical vs. Logical Separation: These blocks do not need to be next to each other in the physical VRAM. Block 1 could be at memory address 0x001, and Block 2 could be at memory address 0x999. They are scattered wherever there is free space.
- The Block Table (The Map): vLLM maintains a "Block Table." When the model's attention mechanism tries to read the KV cache (which it thinks is one continuous, logical sequence), vLLM intercepts the read request. The Block Table acts as a map, seamlessly pointing the math cores to the scattered physical blocks in the VRAM.
- Dynamic Allocation: When a user asks a question, vLLM does not pre-allocate 2,000 tokens. It allocates exactly enough 16-token blocks to hold the prompt. As the model starts generating new tokens during the Decode phase, it fills up the current block. Only when that block hits token #16 does vLLM dynamically allocate one new block from the VRAM pool. **Note** that, the entire prompt is still being processed, it is just that the matrix size is kept to a minimum for Q, K, V.
- To maintain coherency in spite of breaking of the matrix, it also uses **FlashAttention** (Online Softmax).
- If we consider the original dimensions (B, seq_len, hidden_dim):
    - B indicates number of concurrent users, seq_len indicates the length of input sequence (or prompt).
    - vLLM will break the seq_len for multiple users, and store different user blocks in different memory locations depending on which is free.
    - The block table keeps track of which blocks are part of which user's sequence.
    - On execution, it simply fetches these blocks from their addresses and uses **FlashAttention** (Online Softmax) to combine their results.
    - vLLM essentially manages the 'Batch' dimension entirely in software via the Block Table, rather than relying on a rigid hardware dimension.

## Continous batching
- The vLLM engine stops thinking about "batches of users" and starts thinking about a continuous conveyor belt of individual tokens. Every single time the model does a forward pass to generate a single token, the scheduler runs a check.
    1. User 1 emits an <EOS> (End of Sequence) token. They are done.
    2. The vLLM scheduler instantly yanks User 1 out of the active batch and streams their completed text back to their application.
    3. Same Iteration as abovee: The scheduler checks its waiting queue, grabs a brand new request from User 5, and slots them directly into the vacant space left by User 1.
    4. Moving Forward: User 5 starts their very first "Prefill Phase" step right alongside the "Decode Phase" steps of Users 2, 3, and 4.
- Because PagedAttention chopped the memory layout into a modular pool of 16-token blocks, continuous batching becomes effortlessly lightweight.
- When a user finishes, vLLM doesn't have to restructure a giant tensor. It simply marks that user's physical blocks as "Free" in the master pool.
- When a new user is instantly slotted into the batch, the scheduler just assigns those freshly freed physical blocks to that user's new Block Table.