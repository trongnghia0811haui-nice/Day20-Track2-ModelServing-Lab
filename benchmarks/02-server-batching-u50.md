# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.94 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 19044 |

Highest sampled value was **3.94 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak n_busy_slots_per_decode đạt 3.94/4 (98%), với requests_processing = 4 và
requests_deferred khoảng 45 trong lúc load-50 đang chạy. Đây là bằng chứng trực tiếp
continuous batching đang đóng gói các request vào cả bốn slot. Effective concurrency
41.2 trong load report lớn hơn 4 vì bao gồm request đang xếp hàng; vì vậy 3.94/4 là
chỉ báo slot utilization, còn 41.2 là chỉ báo queueing.
