# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 85.1 | 100% |
| 4 | 85.0 | 100% |
| 8 | 85.1 | 100% |
| 12 | 85.2 | 100% |
| 24 | 84.8 | 100% |

**Best**: `-t 12` at 85.2 tok/s
**Slowest tested**: `-t 24` at 84.8 tok/s (1.00x spread)
**Against the physical-core default** (`-t 8`, 85.1 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=12 make bench
```

## Your explanation

Đường cong gần như phẳng: 85.0–85.2 tok/s từ 1 đến 24 threads; best là -t 12,
chỉ 1.00x so với baseline 8 physical cores. Không có knee rõ vì decode đã được
offload CUDA (ngl=99), nên phần chính bị giới hạn bởi GPU/backend và memory path,
không phải số thread CPU. Oversubscription 24 threads chỉ giảm nhẹ xuống 84.8 tok/s;
đây là dấu hiệu thêm thread không tạo thêm công việc hữu ích.
