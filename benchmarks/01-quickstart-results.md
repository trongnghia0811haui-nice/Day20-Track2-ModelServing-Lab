# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3159 | 270 / 343 | 30.9 / 40.0 | 2163 / 2399 / 2399 | 32.3 |
| UD-Q2_K_XL | 0.39 | 2057 | 330 / 364 | 29.9 / 34.4 | 2203 / 2487 / 2487 | 33.4 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.03x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Your observation

Trên máy của tôi, UD-Q2_K_XL nhỏ hơn 0.73 GB (2.24 so với 2.97 GB), giảm khoảng 25%;
TPOT P50 giảm từ 12.3 xuống 11.6 ms và decode tăng từ 81.4 lên 86.0 tok/s (1.06x).
TTFT/E2E cũng tốt hơn nhẹ. Spot-check cùng một prompt cho thấy cả hai bản đều trả lời
được, nhưng một prompt chưa đủ để kết luận chất lượng; vì speedup chỉ 6%, tôi vẫn chọn
Q4 cho workload nhạy chất lượng và dùng Q2 khi áp lực VRAM là ưu tiên.
