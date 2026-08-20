# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 44 | 0.74 | 9200 | 17000 | 40000 | 8.4 | 0.0% |
| 50 | 41 | 0.70 | 34000 | 56000 | 57000 | 22.1 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.94x** (19% of linear) |
| P95 latency | **3.29x** |
| Effective concurrency at 50 users | 22.1 vs `--parallel 4` slots (occupancy/slot ratio 5.52) |

**Saturated.** Throughput delivered only 0.94x for 5x the offered load, and effective concurrency (22.1) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.94x while P95 moved 3.29x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server đã bão hòa ở hoặc trước mức 10 users: ngay tại u10, effective concurrency
đã là 8.4, vượt 4 decode slots. Khi offered load tăng 5x lên u50, throughput còn
giảm từ 0.74 xuống 0.70 RPS (0.94x), trong khi P95 tăng từ 17 lên 56 giây (3.29x)
và `requests_deferred` đạt 46; latency tăng thêm chủ yếu là queue time. Để tăng
goodput@SLO, trước tiên tôi sẽ giới hạn admission/concurrency gần năng lực 4 slots,
thay vì tăng `--parallel` trên CPU 4 core vốn dễ làm tăng contention; sau đó mới đo
lại P95 và thử tối ưu decode hoặc bổ sung compute.
