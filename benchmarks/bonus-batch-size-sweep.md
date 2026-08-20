# Bonus - Batch-size sweep (chunked prefill)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=4` `ngl=0` · metric `pp2048`

| -b (logical) | -ub (micro) | pp2048 (tok/s) | vs best |
|:--|--:|--:|--:|
| 128 | 128 | 156.4 | 98% |
| 256 | 256 | 160.4 | 100% |
| 512 | 256 | 157.9 | 98% |
| 512 | 512 | 155.7 | 97% |
| 1024 | 512 | 148.6 | 93% |
| 2048 | 512 | 152.9 | 95% |

Best: `-b 256 -ub 256` at 160.4 tok/s
(1.08x the slowest point tested).

This sweep only measures the throughput half of the trade. The cost it hides is
TTFT for queued requests: a larger micro-batch holds the device longer per step,
so anything waiting behind it waits longer. To see both halves, re-run
`make load-50` with your best and worst settings via
`.venv/bin/python labs/02-serve/serve.py -- -b N -ub M` and compare P95.

## Your finding

Trong sáu cấu hình đã thử, `-b 256 -ub 256` đạt prefill throughput cao nhất,
160.4 tok/s. So với mặc định của `llama-bench` trong build này là
`-b 2048 -ub 512` (152.9 tok/s), cấu hình được chọn nhanh hơn **1.05x**, tương
đương khoảng **4.9%**. Toàn bộ dải đo có spread 1.08x, từ 148.6 đến 160.4
tok/s.

Đường cong không đơn điệu: tăng batch hoặc micro-batch không liên tục làm
throughput tốt hơn. Trên CPU 4 core, `256/256` có vẻ đủ lớn để khấu hao overhead
mỗi bước mà chưa làm working set và tranh chấp cache/memory bandwidth tăng quá
mức; `1024/512` lại là cấu hình chậm nhất. Đây là một giả thuyết cơ chế phù hợp
với số đo, không phải kết luận về latency serving.

Tôi sẽ chọn tạm thời `-b 256 -ub 256` cho production, nhưng sweep `pp2048` này
chỉ đo throughput prefill đơn lẻ. Trước khi triển khai, tôi cần chạy cùng một
`load-50` cho cấu hình mặc định và cấu hình được chọn, rồi so TTFT P95, E2E P95,
RPS, failures và `requests_deferred`. Nếu throughput tăng nhưng P95 hoặc queueing
xấu đi đáng kể, tôi sẽ giữ cấu hình mặc định hoặc giảm micro-batch.
