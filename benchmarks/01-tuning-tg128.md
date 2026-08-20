# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **4 physical · 4 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 15.6 | 83% |
| 2 | 14.2 | 76% |
| 4 | 18.8 | 100% |

**Best**: `-t 4` at 18.8 tok/s
**Slowest tested**: `-t 2` at 14.2 tok/s (1.32x spread)
**Against the physical-core default** (`-t 4`, 18.8 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

Knee nằm ở `-t 4`, đúng bằng 4 physical cores, đạt 18.8 tok/s: nhanh hơn `-t 1`
khoảng 1.20x và nhanh hơn điểm thấp nhất `-t 2` khoảng 1.32x. Đây là workload
CPU-only (`ngl=0`), nên dùng đủ bốn core giúp chia công việc decode; vượt số core
vật lý có thể chỉ làm các thread tranh memory bandwidth và cache, nhưng sweep này
không có điểm trên 4 thread để khẳng định mức giảm đó. Việc `-t 2` chậm hơn `-t 1`
là một điểm không đơn điệu, có thể do nhiễu hệ thống hoặc scheduling; cần lặp lại
nhiều lần trước khi coi đây là đặc tính ổn định.
