# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.73 of 4 slots (93%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 10025 |

Highest sampled value was **3.73 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak `n_busy_slots_per_decode` là 3.73/4 slots (93%), đồng thời
`requests_processing` đạt 4 và `requests_deferred` đạt 46. Con số này không cần
bằng effective concurrency 22.1 ở u50: 3.73 là số slot bận trung bình trên mỗi bước
decode, còn 22.1 theo Little's Law tính toàn bộ request đang xử lý lẫn đang chờ.
Vì vậy tôi tin metric server 3.73/4 khi đánh giá continuous batching, và dùng 22.1
cùng deferred=46 làm bằng chứng cho queueing; hai phép đo bổ sung, không mâu thuẫn.
