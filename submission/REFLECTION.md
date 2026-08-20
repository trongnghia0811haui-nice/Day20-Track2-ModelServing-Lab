# Báo cáo cá nhân — Day 20: Model Serving & Inference Optimization

**Họ tên:** Trần Trọng Nghĩa
**Cohort:** A20/K4
**Ngày nộp:** 2026-08-20

---

## 1. Phần cứng và runtime  *(rubric 1, 2 — 10 điểm)*

Tôi thực hiện toàn bộ Phase 1 trên môi trường Linux cục bộ. Đây là cấu hình mà
`make probe` thực sự nhìn thấy trong lúc chạy lab:

| Hạng mục | Cấu hình |
|---|---|
| Hệ điều hành | Linux 6.8.0-138-generic, x86_64 |
| CPU | 12th Gen Intel(R) Core(TM) i5-12450H |
| Tài nguyên CPU khả dụng | 4 physical cores / 4 logical cores |
| CPU extensions | AVX2: có; AVX-512: không |
| RAM khả dụng | 5.4 GB |
| Accelerator | Không có backend GPU khả dụng; chạy CPU-only |
| llama.cpp | Bản dựng b10488, asset `llama-b10488-bin-ubuntu-x64.tar.gz` |
| Model | Qwen3.5 0.8B |
| Quantization | Q4_K_M (primary) và UD-Q2_K_XL (compare) |

### Quá trình thiết lập

Vì môi trường chỉ có 5.4 GB RAM, tôi chủ động chọn
`LAB_MODEL=qwen35-0.8b` thay cho Gemma mặc định. `make setup` tạo virtual
environment, tải runtime llama.cpp và hai file GGUF thành công. Không có GPU để
offload nên toàn bộ phép đo dùng `ngl=0`; tôi cũng không cần chuyển sang cloud.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

Benchmark dùng 4 threads, context 2048, `ngl=0`, tối đa 64 output tokens và bỏ
lần warm-up. Cả hai quantization đều hoàn thành đủ 10/10 request.

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:---|---:|---:|---:|---:|---:|---:|
| Q4_K_M | 0.50 | 3159 | 270 / 343 | 30.9 / 40.0 | 2163 / 2399 / 2399 | 32.3 |
| UD-Q2_K_XL | 0.39 | 2057 | 330 / 364 | 29.9 / 34.4 | 2203 / 2487 / 2487 | 33.4 |

Q2 giảm 0.11 GB, tương đương 22% dung lượng, nhưng tốc độ decode chỉ tăng từ
32.3 lên 33.4 tok/s, tức 1.03×; TTFT P50 lại tăng từ 270 lên 330 ms. Tôi đã hỏi
cùng một prompt và cả hai đều có chỗ reasoning chưa thuyết phục, nên một mẫu chưa
đủ để chứng minh chất lượng tương đương. Với mức tăng tốc nhỏ này, tôi giữ Q4 làm
mặc định và chỉ dùng Q2 khi thiếu RAM.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

Server chạy với `--parallel 4`, `ctx=2048`, 4 CPU threads và continuous
batching. Hai lần load test đều không có request lỗi.

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.74 | 9200 | 17000 | 40000 | 8.4 | 0.0% |
| 50 | 0.70 | 34000 | 56000 | 57000 | 22.1 | 0.0% |

- Offered load tăng 5× nhưng throughput thực chỉ còn **0.94×**, tức giảm khoảng 6%.
- P95 tăng **3.29×**, từ 17 lên 56 giây.
- Effective concurrency tại 50 users là **22.1**, trong khi server chỉ có 4 slots.
- Peak `n_busy_slots_per_decode` đạt **3.73/4 slots (93%)**.
- `requests_processing` đạt 4 và `requests_deferred` đạt 46.

Máy đã bão hòa ở hoặc trước mức 10 users: ngay tại u10, effective concurrency 8.4
đã vượt 4 slots. Khi tăng lên u50, RPS còn giảm trong khi P95 tăng hơn ba lần và
46 request bị deferred, nên phần latency tăng thêm chủ yếu là thời gian xếp hàng.
Con số 3.73/4 phản ánh slot đang decode, còn 22.1 bao gồm cả request đang chờ. Để
tăng goodput@SLO, tôi sẽ giới hạn admission/concurrency gần năng lực thực trước;
tăng thêm `--parallel` trên CPU 4 core có nguy cơ chỉ làm contention nặng hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

| Day | Thành phần | Trạng thái |
|---|---|---|
| N16 Cloud/IaC | Chạy trên localhost | Stub |
| N17 Data pipeline | Danh sách in-memory | Stub |
| N18 Lakehouse | Toy dictionary hard-code | Stub |
| N19 Vector + features | Keyword overlap | Stub |
| N20 Serving | `llama-server` cục bộ | Real |

Latency trung bình của ba query:

- Embed: **0.0 ms**
- Retrieve: **0.0 ms**
- LLM: **7145.2 ms**
- Tổng: **7145.2 ms**

LLM chiếm 100% tổng latency, đúng với kỳ vọng vì embedding và retrieval đều là
stub rất nhẹ. Tuy nhiên, context được lấy đúng mà câu trả lời vẫn lặp và đôi lúc
mâu thuẫn với context, nên chất lượng model cũng là một giới hạn thực tế. Hai query
đầu chạm trần 200 output tokens. Nếu cần giảm latency một nửa, tôi sẽ giảm output
length và tối ưu decode/backend trước; tối ưu hai stage đang đo 0.0 ms gần như
không tạo khác biệt.

---

## 5. Thay đổi có tác động rõ nhất  *(rubric 11 — 10 điểm)*

**Thay đổi:** tăng số thread từ `-t 2` lên `-t 4`, bằng số physical core mà môi
trường cấp cho tiến trình.

```
before:  14.2 tok/s (-t 2)
after:   18.8 tok/s (-t 4)
speedup: 1.32× (nhanh hơn khoảng 32%)
```

Do `ngl=0`, toàn bộ phép tính đều chạy trên CPU. Với `-t 2`, model chỉ dùng một
nửa số core vật lý khả dụng; chuyển sang `-t 4` giúp llama.cpp phân phối kernel
decode lên đủ bốn core và tăng mức song song. Tốc độ không tăng 2× vì các thread
vẫn chia sẻ memory bandwidth và cache, đồng thời phải trả chi phí đồng bộ.

Đường cong không hoàn toàn đơn điệu: `-t 2` còn chậm hơn `-t 1` (14.2 so với
15.6 tok/s). Tôi xem đây là dấu hiệu của nhiễu hệ thống hoặc scheduling, không phải
một quy luật chắc chắn. Dù vậy, `-t 4` vẫn là điểm tốt nhất đã đo với 18.8 tok/s.
Sweep không có điểm trên 4 thread, nên tôi không kết luận về oversubscription nếu
chưa chạy thêm và lặp lại mỗi cấu hình nhiều lần.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

Tôi chưa thực hiện bonus; ưu tiên của lần chạy này là hoàn thiện và hiểu rõ toàn bộ
base track.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều bất ngờ nhất là tăng số user lên 5× không làm throughput tăng mà còn giảm nhẹ,
trong khi P95 tăng hơn ba lần. Kết quả này làm tác động của queueing rõ ràng hơn hẳn
so với chỉ nhìn vào số request/giây.

---

## 8. Self-check trước khi push

- [x] Đã tạo đủ artifact của Phase 1.
- [x] Đã hoàn thành các phần nhận xét bắt buộc trong báo cáo benchmark.
- [x] Đã có 5 ảnh trong `submission/screenshots/`.
- [x] `make verify` trả về exit code 0.
- [x] `models/*.gguf` và `runtime/` được giữ ngoài Git.
- [x] Rà soát `git diff`, stage và commit các thay đổi hiện tại.
- [x] Push lên GitHub ở chế độ public.
- [x] Gửi URL repository lên LMS.

Repository cần được giữ ở chế độ public cho tới khi có kết quả chấm.
