# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3067.6 | 3067.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2797.9 | 2798.0 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2815.6 | 2815.6 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **2893.7** · total **2893.8**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

N16 Cloud/IaC, N17 data pipeline, N18 lakehouse và N19 vector/features đều là stub
toy-data/keyword overlap trong bản base; N20 llama-server là real. Dominant stage là
llm (mean 2893.7 ms, 100% của total), đúng kỳ vọng vì embed/retrieve gần như 0 ms.
Muốn giảm latency 2x, tôi sẽ tấn công llm/decode trước: quantization hoặc backend/
batching phù hợp có tác động lớn hơn tối ưu retrieval toy.
