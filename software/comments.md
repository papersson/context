# Code Commenting Guidelines

Based on "A Philosophy of Software Design" by John Ousterhout.

---

## Core Principle

Comments should capture information that is not obvious from the code. Code shows *what* happens; comments explain *why*, the design rationale, invariants, and non-obvious implications.

---

## What to Comment

### 1. Module/File Headers

Every `.h` and `.c` file should start with a comment explaining:
- **Purpose** — what abstraction does this module provide?
- **Key concepts** — any domain knowledge needed to understand it
- **Usage pattern** — typical calling sequence if not obvious

```c
/*
 * safetensors.h - Load tensors from HuggingFace safetensors format
 *
 * Safetensors layout:
 *   [8 bytes: header_size (little-endian uint64)]
 *   [header_size bytes: JSON metadata]
 *   [remainder: raw tensor data]
 *
 * Usage:
 *   st_open("model.safetensors", &st);
 *   st_parse_header(&st, tensors, max, &count);
 *   st_load(dst, &st, tensors, count, "name", numel);
 *   st_close(&st);
 */
```

### 2. Interface Comments (What, Not How)

Document the abstraction a function provides, not its implementation:
- What does it do?
- What are the semantics of parameters and return values?
- What are the side effects?
- What are the preconditions/postconditions?

```c
/*
 * BF16 is a truncated FP32: same sign bit and 8-bit exponent,
 * but only 7 bits of mantissa instead of 23. Converting to FP32 is just
 * padding the mantissa with zeros, i.e., shifting left 16 bits.
 *
 * This is NOT the same as FP16, which has a 5-bit exponent.
 */
static void convert_bf16_to_fp32(const uint16_t* src, float* dst, size_t count);
```

### 3. Cross-Module Documentation

When changing code in one place requires changes elsewhere, document this at the most obvious location (where developers will encounter it first).

```c
/*
 * CROSS-MODULE: The pointer assignment order here must exactly match
 * the loading order in load_all_weights(). If adding a new weight:
 *   1. Add field to LayerWeights struct
 *   2. Add size calculation in calc_weight_buffer_size()
 *   3. Add pointer assignment here
 *   4. Add st_load() call in load_all_weights() IN THE SAME ORDER
 */
static int allocate_weights(Qwen3Weights* w) {
```

### 4. Non-Obvious Design Decisions

Explain *why* the code is structured a certain way when alternatives exist:

```c
/*
 * Memory layout: All weights are stored in a single contiguous FP32 buffer.
 * This simplifies memory management and will make CUDA porting easier (single
 * cudaMemcpy). The LayerWeights struct contains pointers into this buffer.
 */
```

### 5. Struct Field Semantics

When a type doesn't convey meaning (e.g., `float*` for a weight matrix), document what it represents:

```c
typedef struct {
    float* q_proj;   // [num_heads * head_dim, hidden_size]
    float* k_proj;   // [num_kv_heads * head_dim, hidden_size]
    size_t offset;   // bytes from start of tensor data section, not file
} LayerWeights;
```

### 6. Format Specifications and Magic Numbers

Document formats, protocols, or magic values where they're used:

```c
/*
 *   FP32: [1 sign][8 exp][23 mantissa]
 *   BF16: [1 sign][8 exp][7 mantissa]
 */
uint32_t fp32_bits = ((uint32_t)bf16) << 16;
```

---

## What NOT to Comment

### 1. Don't Restate the Code

```c
// BAD: restates the obvious
i++;  // increment i

// BAD: the function name already says this
// This function opens a file
int open_file(const char* path);
```

### 2. Don't Describe Implementation in Interface Comments

Interface comments should describe *what*, not *how*:

```c
// BAD: describes implementation
// Iterates through the array and checks each element
bool contains(int* arr, int n, int val);

// GOOD: describes contract
// Returns true if val appears in arr[0..n-1]
bool contains(int* arr, int n, int val);
```

### 3. Don't Add Comments That Will Get Stale

If a comment requires updating when code changes but isn't near that code, it will become wrong:

```c
// BAD: will become stale
// There are 12 weights per layer
#define WEIGHTS_PER_LAYER 12  // but what if we add one?

// GOOD: derived from truth
#define WEIGHTS_PER_LAYER (sizeof(LayerWeights) / sizeof(float*))
```

---

## Process

1. **Write comments first** — before or during implementation, not after. This clarifies your thinking about the abstraction.

2. **Review comments in code review** — are they accurate? Do they capture non-obvious information?

3. **Update comments when code changes** — stale comments are worse than no comments.

---

## For Large Codebases

Consider a `designNotes` file for cross-cutting concerns that don't have an obvious home in any single module. Organize by topic, not chronologically.
