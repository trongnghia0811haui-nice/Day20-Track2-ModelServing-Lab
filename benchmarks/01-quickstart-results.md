# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 4029 | 252 / 453 | 12.3 / 12.5 | 1014 / 1226 / 1226 | 81.4 |
| UD-Q2_K_XL | 2.24 | 3184 | 246 / 364 | 11.6 / 11.9 | 970 / 1097 / 1097 | 86.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.06x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

Trên máy này, UD-Q2_K_XL nhỏ hơn 0.73 GB (2.24 so với 2.97 GB), giảm khoảng 25%;
TPOT P50 giảm từ 12.3 xuống 11.6 ms và decode tăng từ 81.4 lên 86.0 tok/s (1.06x).
TTFT/E2E cũng tốt hơn nhẹ. Spot-check cùng một prompt cho thấy cả hai bản đều trả lời
được, nhưng một prompt chưa đủ để kết luận chất lượng; vì speedup chỉ 6%, tôi vẫn chọn
Q4 cho workload nhạy chất lượng và dùng Q2 khi áp lực VRAM là ưu tiên.
