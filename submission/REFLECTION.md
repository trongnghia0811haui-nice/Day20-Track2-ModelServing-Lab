# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** _<Họ Tên>_
**Cohort:** _<A20-K1 / A20-K2 / ...>_
**Ngày submit:** _2026-08-20_

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11
- **CPU:** Intel Core i5-12450H
- **Cores:** 8 physical / 12 logical
- **CPU extensions:** — (probe Windows không báo extension)
- **RAM:** 15.7 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 Ti 4 GB (CUDA; Vulkan cũng được phát hiện)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-cuda-12.4-x64.zip
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Python launcher được gọi bằng `py` vì lệnh `python` chưa có trong PATH. Tôi đặt
`PYTHONIOENCODING=utf-8` để tránh lỗi console CP1252, sau đó cài dependency, tải
llama.cpp CUDA b10488 và hai GGUF thành công.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 4029 | 252 / 453 | 12.3 / 12.5 | 1014 / 1226 / 1226 | 81.4 |
| UD-Q2_K_XL | 2.24 | 3184 | 246 / 364 | 11.6 / 11.9 | 970 / 1097 / 1097 | 86.0 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 giảm 0.73 GB và nhanh hơn 1.06x (86.0 so với 81.4 tok/s), nhưng mức tăng chỉ
6%. Spot-check cùng prompt cho thấy cả hai đều coherent; tôi giữ Q4 cho workload
nhạy chất lượng và chỉ chọn Q2 khi VRAM là ràng buộc chính.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 2.85 | 2400 | 3900 | 4900 | 7.3 | 0.0% |
| 50 | 2.99 | 15000 | 17000 | 17000 | 41.2 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.05×
- **P95 tăng:** 4.36×
- **Effective concurrency ở 50 users:** 41.2 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.94 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hòa ở 50 users: RPS chỉ tăng 1.05x nhưng P95 tăng 4.36x, effective
concurrency lên 41.2 so với 4 slots và metrics ghi deferred khoảng 45. Vì vậy
latency thêm chủ yếu là queue time. Tôi sẽ kiểm soát admission/concurrency trước;
chỉ tăng parallel sau khi đo lại VRAM và P95.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost-only | stub |
| N17 Data pipeline | in-memory list | stub |
| N18 Lakehouse | toy dictionary | stub |
| N19 Vector + features | keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 2893.7 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Kết quả đúng kỳ vọng: toy retrieval gần như miễn phí còn LLM chiếm toàn bộ latency.
Muốn giảm 2x, tôi sẽ tối ưu llm/decode bằng quantization/backend hoặc scheduling;
thay keyword overlap không tạo khác biệt đáng kể trên pipeline nhỏ này.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ thread setting từ 24 xuống 12 sau khi tune; dùng -t 12 làm cấu hình tốt nhất

```
before:  84.8 tok/s (oversubscribed -t 24)
after:   85.2 tok/s (-t 12)
speedup: 1.00× (0.47% thực tế)
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Đường cong gần như phẳng từ 1 đến 24 threads vì decode đã offload CUDA với ngl=99;
GPU/backend và memory path chi phối nhiều hơn CPU thread count. -t 12 đạt 85.2
tok/s, chỉ nhỉnh hơn -t 8 ở 85.1 tok/s và cao hơn -t 24 ở 84.8 tok/s. Kết quả
cho thấy tuning thread không phải bottleneck chính trên máy này; tăng thread quá mức
không tạo thêm công việc hữu ích.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
