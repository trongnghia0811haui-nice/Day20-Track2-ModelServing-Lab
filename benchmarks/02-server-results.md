# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=12` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 166 | 2.85 | 2400 | 3900 | 4900 | 7.3 | 0.0% |
| 50 | 174 | 2.99 | 15000 | 17000 | 17000 | 41.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.05x** (21% of linear) |
| P95 latency | **4.36x** |
| Effective concurrency at 50 users | 41.2 vs `--parallel 4` slots (occupancy/slot ratio 10.29) |

**Saturated.** Throughput delivered only 1.05x for 5x the offered load, and effective concurrency (41.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.05x while P95 moved 4.36x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server đã bão hòa ở mức 50 users: offered load tăng 5x nhưng throughput chỉ tăng
1.05x (2.85 lên 2.99 RPS), trong khi P95 tăng 4.36x (3.9 lên 17.0 giây). Effective
concurrency lên 41.2 so với 4 slots và metrics ghi peak 3.94/4 cùng deferred khoảng
45; phần latency tăng thêm vì vậy chủ yếu là queue time. Để tăng goodput@SLO, trước
tiên tôi sẽ kiểm soát concurrency/admission thay vì tăng slot mù quáng; nếu VRAM còn
đủ, thí nghiệm kế tiếp là parallel lớn hơn và đo lại P95 cùng KV-cache pressure.
