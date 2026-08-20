# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 8408.3 | 8408.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 7466.9 | 7466.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 5560.3 | 5560.3 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **7145.2** · total **7145.2**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> The user is asking about the difference between **goodput** and **raw throughput** in the context of LLMs.

**1. Goodput:**
*   **Definition:** Goodput is a specific metric that is often used to represent the **best-effort** or **best-effort** performance of the model. It is a subset of the broader "goodput" metric.
*   **Context:** It is often used to measure the quality of the model's output.
* 

**What problem does PagedAttention actually solve?**

> The question is not a valid question.

PagedAttention is a specific implementation of the Attention mechanism.

The question asks "what problem it solves", which refers to the **Attention problem** in the context of the **Attention mechanism** (specifically `pagedAttention` or `pagedAttention`), not the general problem of the entire model.

The context provided is:
> "PagedAttention stores the KV 

**When does splitting prefill and decode help?**

> The context provided does not contain the specific answer to the question "When does splitting prefill and decode help?" directly.

However, the context highlights a **conceptual principle** (that prefilling is compute-bound and decoding is memory-bound) to explain how different approaches are used in the context of the model.

Based on this context, **splitting prefill and decode** is generally c


## Which N16-N19 pieces are real

- **N16 Cloud/IaC:** stub, pipeline chỉ chạy trên localhost.
- **N17 Data pipeline:** stub, dữ liệu là một danh sách in-memory.
- **N18 Lakehouse:** stub, corpus/toy dictionary được hard-code trong script.
- **N19 Vector + features:** stub, retrieval dùng keyword overlap thay vì vector DB.
- **N20 Serving:** real, request được gửi tới `llama-server` cục bộ.

LLM là stage chi phối với mean 7145.2 ms, chiếm 100% tổng latency; kết quả này đúng
kỳ vọng vì embed và retrieval stub gần như 0 ms. Muốn giảm latency pipeline xuống
một nửa, tôi sẽ tối ưu LLM trước—đặc biệt decode/output length, backend hoặc compute—
vì tối ưu hai stage đang đo 0.0 ms không thể tạo speedup đáng kể.
