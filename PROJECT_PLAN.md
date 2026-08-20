# Kế hoạch thực hiện Day 20 — Model Serving & Inference Optimization

## 1. Phạm vi và nguyên tắc

Đây là kế hoạch thực hiện lab theo đúng [README.md](README.md), [GUIDE.md](GUIDE.md), [HARDWARE-GUIDE.md](HARDWARE-GUIDE.md) và [rubric.md](rubric.md).

- Base track: 100 điểm, bắt buộc.
- Bonus track: tối đa 20 điểm, chỉ làm sau khi base track hoàn tất.
- Chỉ so sánh before/after trên cùng máy hoặc cùng cloud VM; không so tốc độ tuyệt đối với máy khác.
- Base track không cần GPU, compiler hoặc Docker.
- Grader chấm artifact thật trong repo, không chấm lời khẳng định không có bằng chứng.
- Làm theo thứ tự: setup → measure → serve → load/metrics → saturation → integrate → reflection → verify → submit.
- Không sửa starter code/config nếu rubric không yêu cầu.

## 2. Hiện trạng repo sau khi rà soát

- Đã rà toàn bộ 47 file được Git theo dõi, gồm 15 file Markdown, mã nguồn core, notebook cloud, bonus, submission và verifier.
- Nhánh hiện tại là main; working tree và staging area sạch.
- Chưa có .venv, hardware.json, models/, runtime/ hoặc kết quả benchmark.
- benchmarks/ mới có .gitkeep.
- submission/REFLECTION.md vẫn là template.
- Máy Windows đã phát hiện NVIDIA GeForce RTX 3050 Ti Laptop GPU 4 GB qua nvidia-smi; RAM hệ thống phải xác nhận bằng probe.
- Trong shell kiểm tra hiện tại, lệnh python chưa được nhận diện ổn định. Cần xác nhận Python thật trong PowerShell thông thường trước khi chạy bootstrap.

## 3. Kiến trúc và luồng thực hiện

~~~text
lab.ps1 / Makefile
        │
        ├── 00-setup
        │   ├── detect-hardware.py → hardware.json
        │   ├── fetch-runtime.py   → runtime/b10488 + runtime/active.json
        │   └── download-model.py  → models/*.gguf + models/active.json
        │
        ├── 01-measure
        │   ├── benchmark.py → TTFT / TPOT / E2E / percentiles
        │   └── tune.py      → thread sweep và before/after
        │
        ├── 02-serve
        │   ├── serve.py → llama-server + OpenAI API + /metrics
        │   ├── smoke-test.py → completion + metric non-zero
        │   ├── load-test.py → Locust 10/50 users
        │   ├── record-metrics.py → continuous-batching evidence
        │   └── load-report.py → Little's Law + saturation reading
        │
        ├── 03-integrate
        │   └── pipeline.py → RAG toy pipeline, context và latency split
        │
        └── scripts/verify.py → submission readiness
~~~

lib/labkit.py là lớp dùng chung để chọn model, đọc manifest, chọn thread/GPU/context, tìm binary, chạy server nền, parse llama-bench và ghi report.

## 4. Phạm vi file được phép thay đổi

### 4.1. Các file phải giữ nguyên

Không chỉnh các file starter sau vì tài liệu và rubric không yêu cầu:

- .env.example
- .gitignore
- README.md
- GUIDE.md
- HARDWARE-GUIDE.md
- lab.ps1 (giữ nguyên logic; chỉ chuẩn hóa UTF-8 BOM nếu Windows PowerShell parse lỗi)
- Makefile
- pyproject.toml
- requirements.txt
- rubric.md
- scripts/verify.py
- lib/labkit.py
- các script trong labs/00-setup, labs/01-measure, labs/02-serve
- cloud/Day20-lab.ipynb nếu chạy local

### 4.2. Artifact base được phép tạo/điền

- hardware.json
- models/active.json
- benchmarks/01-quickstart-results.md và JSON
- benchmarks/01-tuning-tg128.md và JSON
- benchmarks/locust-10_stats.csv
- benchmarks/locust-50_stats.csv
- benchmarks/02-server-metrics-u50.csv
- benchmarks/02-server-batching-u50.md
- benchmarks/02-server-results.md và JSON
- benchmarks/03-integration-results.md và JSON
- các mục nhận xét trong những report trên
- submission/REFLECTION.md
- tối thiểu 5 ảnh trong submission/screenshots/

### 4.3. Ngoại lệ chỉ khi chủ động chọn

- Chỉ sửa labs/03-integrate/pipeline.py nếu muốn thay TOY_DOCS và retrieve() bằng N19 thật.
- Chỉ tạo bonus/<challenge>.md hoặc benchmarks/bonus-*.md khi làm bonus/challenge.
- Chỉ sửa registry trong lib/labkit.py nếu upstream đổi tên model và workaround đổi tên file không giải quyết được.

### 4.4. Không commit

- models/*.gguf
- runtime/
- .venv/
- bonus/llama.cpp/
- benchmarks/.llama-server.log
- benchmarks/locust-*_stats_history.csv
- benchmarks/locust-*_failures.csv
- benchmarks/locust-*_exceptions.csv

## 5. Kế hoạch chi tiết theo từng phần

### Phần 0 — Preflight và khóa phạm vi

Mục tiêu: bảo đảm môi trường chạy được mà chưa tạo artifact ngoài ý muốn.

~~~powershell
git status --short --branch
python --version
nvidia-smi
.\lab.ps1
~~~

Yêu cầu:

- Python >=3.10.
- Có đủ RAM và dung lượng đĩa theo model được chọn.
- Không có process chiếm port 8080 hoặc 8099.
- Không sửa file starter để xử lý lỗi môi trường.
- Không tạo .env; dự án đọc biến trực tiếp từ process environment.

Nếu python không hoạt động, cài Python chính thức hoặc sửa PATH trước khi tiếp tục. bootstrap.ps1 và lab.ps1 đều gọi trực tiếp python.

### Phần 1 — Probe, chọn model và setup

Rubric: mục 1–2, tổng 10 điểm.

Chạy probe:

~~~powershell
.\lab.ps1 probe
~~~

Quyết định model:

| Điều kiện | Model | Lệnh |
|---|---|---|
| RAM >=8 GB | Gemma 4 E2B, chất lượng tốt hơn | .\lab.ps1 setup |
| RAM 4–8 GB hoặc cần chạy nhẹ | Qwen3.5 0.8B | đặt LAB_MODEL rồi chạy setup |
| RAM <4 GB hoặc local không khắc phục được | Cloud notebook | cloud/Day20-lab.ipynb |

Nhánh Qwen:

~~~powershell
$env:LAB_MODEL = 'qwen35-0.8b'
.\lab.ps1 setup
~~~

Với GPU 4 GB, Qwen là nhánh thực dụng hơn nếu ưu tiên thời gian và độ ổn định. Tuy nhiên quyết định cuối cùng phải dựa trên RAM hệ thống, không chỉ VRAM.

Kiểm tra sau setup:

~~~powershell
.\lab.ps1 probe
Get-Content hardware.json
Get-Content models\active.json
~~~

Tiêu chí hoàn thành:

- hardware.json mô tả đúng máy, runtime environment và backend.
- models/active.json có model, repo, primary model và compare model.
- Hai GGUF đúng tên và có kích thước hợp lý.
- Runtime llama.cpp là build b10488.
- Chụp submission/screenshots/01-hardware-probe.png.

Chỉ commit hai manifest; không commit weights/runtime.

### Phần 2 — Baseline hai quantization

Rubric: mục 3–5, tổng 20 điểm.

~~~powershell
.\lab.ps1 bench
~~~

benchmark.py tự bật server tạm trên port 8099, bỏ warm-up, gửi 10 prompt streaming cho từng quantization và sinh:

- TTFT P50/P95.
- TPOT P50/P95.
- E2E P50/P95/P99.
- Decode tok/s.
- Load time và metadata runtime.

Kiểm tra:

- Cả hai quantization đều có kết quả.
- Tốt nhất đạt 10/10 request cho mỗi quantization.
- TTFT và TPOT được báo riêng.
- Chụp 02-bench.png, thấy cả hai hàng và cột TTFT/TPOT.

Đánh giá chất lượng bằng cùng một câu hỏi:

1. Chạy primary bằng .\lab.ps1 serve.
2. Dừng server bằng Ctrl+C.
3. Chạy compare bằng .venv\Scripts\python.exe labs\02-serve\serve.py --compare.
4. Ghi nhận khác biệt về chất lượng, không chỉ tốc độ và kích thước.

Sau lần đo cuối, thay section Your observation trong 01-quickstart-results.md bằng nhận xét cá nhân: nhỏ hơn bao nhiêu, nhanh hơn bao nhiêu, chất lượng ra sao và có đáng dùng không.

### Phần 3 — Tune thread count

Rubric: dữ liệu cho mục 11, thuộc phần phân tích 20 điểm.

~~~powershell
.\lab.ps1 tune
~~~

Phải phân tích:

- Thread winner.
- Knee của đường cong.
- Before/after so với physical-core default.
- Speedup thực tế.
- Vai trò của memory bandwidth, cache contention và scheduling.
- Nếu đường cong không giống kỳ vọng, giữ nguyên kết quả và giải thích.

Artifact: benchmarks/01-tuning-tg128.md và JSON tương ứng.

Không viết nhận xét trước khi chốt lần đo cuối. Chạy lại tune sẽ ghi đè report và placeholder đã điền.

### Phần 4 — Server và smoke test

Rubric: mục 6–7, tổng 15 điểm.

Trong mọi terminal, dùng cùng cấu hình:

~~~powershell
$env:LAB_N_THREADS = '<thread-tốt-nhất>'
$env:LAB_PARALLEL = '4'
$env:LAB_SERVER_PORT = '8080'
~~~

Terminal 1:

~~~powershell
.\lab.ps1 serve
~~~

Terminal 2:

~~~powershell
.\lab.ps1 smoke
~~~

Ảnh 03-serve-and-smoke.png phải thể hiện:

- Server đang listen.
- Có completion thật.
- llamacpp:tokens_predicted_total khác 0 và tăng sau request.

Nếu đổi port, phải đặt cùng LAB_SERVER_PORT trong server, smoke, load test, metrics và pipeline. Không dùng --parallel khác với LAB_PARALLEL, vì report metrics đọc slot count từ biến môi trường.

### Phần 5 — Load test và continuous batching

Rubric: mục 8–9, tổng 10 điểm.

Giữ server ở terminal 1.

Terminal 2:

~~~powershell
.\lab.ps1 load-10
~~~

Chụp 04-locust-10.png, phải thấy request count, RPS, Median, 95%ile và 99%ile.

Sau đó chạy đồng thời:

Terminal 2:

~~~powershell
.\lab.ps1 load-50
~~~

Terminal 3, ngay khi load-50 bắt đầu:

~~~powershell
.\lab.ps1 metrics
~~~

Không chạy metrics khi server rảnh. Mục tiêu là quan sát n_busy_slots_per_decode tăng về phía --parallel, cùng requests_deferred nếu load vượt slot.

Artifact cần có:

- benchmarks/locust-10_stats.csv
- benchmarks/locust-50_stats.csv
- benchmarks/02-server-metrics-u50.csv
- benchmarks/02-server-batching-u50.md

### Phần 6 — Đọc saturation

Rubric: mục 10, 10 điểm.

Sau khi cả hai load test hoàn tất:

~~~powershell
.\lab.ps1 load-report
~~~

Report phải dùng Little's Law, effective concurrency = RPS × average latency, và trả lời:

- Offered load từ 10 lên 50 users tăng 5× nhưng throughput thực tăng bao nhiêu.
- P95/P99 phồng lên bao nhiêu.
- RPS có plateau không.
- Effective concurrency ở 50 users so với số slot.
- Server bão hòa ở đâu.
- Latency tăng thêm là compute time hay queue time.
- Nếu cần tăng goodput@SLO, knob đầu tiên sẽ đổi là gì và vì sao.

Thay Your reading trong benchmarks/02-server-results.md bằng lập luận dựa trên số thật.

### Phần 7 — Integration pipeline

Rubric: mục 12–13, tổng 15 điểm.

Giữ server chạy:

~~~powershell
.\lab.ps1 pipeline
~~~

Yêu cầu:

- Có đủ 3 query.
- Có context/provenance đã retrieve.
- Có answer.
- Có latency embed, retrieve, llm, total cho từng query và mean.
- Có dominant stage.

Base track cho phép giữ nguyên TOY_DOCS và keyword-overlap retrieval. Khi đó phải khai báo N16–N19 là stub/toy và N20 llama-server là real.

Artifact: benchmarks/03-integration-results.md và JSON tương ứng.

Thay section Which N16-N19 pieces are real trong report; sau đó copy latency split và trạng thái real/stub vào Reflection §4.

### Phần 8 — Hoàn thiện reports và Reflection

Rà placeholder trong mọi report:

~~~powershell
Get-ChildItem benchmarks -Filter *.md |
    Select-String -Pattern 'required -- replace this line'
~~~

Kết quả cuối phải không còn marker.

Điền submission/REFLECTION.md:

- §1: metadata, hardware, runtime, model và setup story <=80 chữ.
- §2: bảng benchmark hai quantization và nhận xét <=60 chữ.
- §3: load/saturation, throughput, P95 và queueing <=80 chữ.
- §4: N16–N20 real/stub, latency split và bottleneck <=60 chữ.
- §5: thay đổi quan trọng nhất, before/after/speedup và giải thích cơ chế trong 1–2 đoạn.
- §6–7: để trống nếu không làm bonus.

§5 phải giải thích cơ chế như memory bandwidth, vector width, cache residency, scheduling hoặc queueing; chỉ chép con số sẽ không đủ điểm.

Nếu dùng Qwen, lấy nguyên bảng từ report sinh ra để tránh giữ nhãn Gemma UD-Q4_K_XL trong template.

### Phần 9 — Screenshots

Năm ảnh bắt buộc:

1. 01-hardware-probe.png
2. 02-bench.png
3. 03-serve-and-smoke.png
4. 04-locust-10.png
5. 05-locust-50.png

Ảnh phải crop sát, chữ đọc được, PNG/JPG và nên dưới khoảng 2 MB. Ảnh tune, batching và pipeline là optional.

### Phần 10 — Verify, staging và submit

Stage explicit paths, không dùng git add .:

~~~powershell
git add -- hardware.json models/active.json
git add -- benchmarks/01-quickstart-results.md benchmarks/01-quickstart-results.json
git add -- benchmarks/01-tuning-tg128.md benchmarks/01-tuning-tg128.json
git add -- benchmarks/locust-10_stats.csv benchmarks/locust-50_stats.csv
git add -- benchmarks/02-server-metrics-u50.csv benchmarks/02-server-batching-u50.md
git add -- benchmarks/02-server-results.md benchmarks/02-server-results.json
git add -- benchmarks/03-integration-results.md benchmarks/03-integration-results.json
git add -- submission/REFLECTION.md
git add -- submission/screenshots/01-hardware-probe.png
git add -- submission/screenshots/02-bench.png
git add -- submission/screenshots/03-serve-and-smoke.png
git add -- submission/screenshots/04-locust-10.png
git add -- submission/screenshots/05-locust-50.png
git diff --cached --name-only
git diff --cached --check
~~~

Danh sách staged chỉ được chứa artifact nộp bài và PROJECT_PLAN.md; tuyệt đối không chứa starter files, model weights hoặc runtime.

Chạy:

~~~powershell
.\lab.ps1 verify
~~~

verify.py hiện có hai giới hạn cần kiểm tra thủ công:

- Trên Windows, WindowsPath dùng dấu ngược còn git ls-files dùng dấu xuôi, nên nested file có thể bị báo nhầm là chưa commit.
- git ls-files có thể xem file staged là tracked dù chưa có trong HEAD.

Nếu native verify báo sai path, xác minh bằng WSL nếu có:

~~~powershell
wsl bash -lc "cd /mnt/d/Tailieu-2026/vin/0.Lab_vin/Day20_lab/Day20-Track2-ModelServing-Lab && python3 scripts/verify.py"
~~~

Sau khi commit, kiểm tra thật:

~~~powershell
git show --name-only --format= HEAD
git status --short
~~~

Cuối cùng push lên GitHub public và paste URL vào LMS. Không cần tạo PR.

## 6. Rubric và artifact mapping

| Rubric | Bằng chứng | Điểm |
|---|---|---:|
| Setup | hardware.json, models/active.json | 10 |
| Measurement | Hai quantization, TTFT/TPOT/percentiles, observation | 20 |
| Serving | Completion, /metrics, load 10/50, batching | 25 |
| Analysis | Saturation report và single change | 20 |
| Integration | Ba query, context, real/stub, latency split | 15 |
| Submission | Reflection, verify, 5 screenshots | 10 |
| **Tổng base** |  | **100** |

## 7. Bonus sau khi base đã verify

Chỉ bắt đầu bonus khi base đã có verify thành công.

Nhánh phù hợp với Windows + RTX 3050 Ti:

- B1: build llama.cpp bằng CMake/Visual Studio Build Tools rồi chạy compare-builds.
- B2: ưu tiên sweep-gpu; nếu runtime không thấy accelerator, dùng sweep-quant.
- B3: lấy before/after từ B1 hoặc B2, không dùng lại kết quả make tune của base.
- B4: chọn C2 KV-cache quantization hoặc C5 model nhỏ nhất còn hữu ích.
- B5: C8 semantic cache offline là lựa chọn nhẹ; C9 cần embedding server và thêm áp lực VRAM.

Không nên ưu tiên C6 trên Windows nếu mục tiêu là so Vulkan với CUDA; hướng này phù hợp Linux hơn theo runtime của repo.

Mỗi bonus report cũng có marker required -- replace this line; phải thay marker và ghi insight vào Reflection §6 hoặc file challenge riêng.

## 8. Rủi ro và cách kiểm soát

| Rủi ro | Cách xử lý |
|---|---|
| Report bị ghi đè khi chạy lại script | Chốt lần đo cuối trước khi viết nhận xét |
| metrics chạy khi server idle | Chạy ngay trong thời gian load-50 |
| Thiếu compare quantization | Không xóa file compare; kiểm tra models/active.json |
| Dùng .env nhưng biến không có hiệu lực | Set $env:... trong từng terminal |
| --parallel thực tế khác report | Dùng LAB_PARALLEL nhất quán |
| Cloud load 30 giây | Giữ 60 giây vì rubric yêu cầu 60 giây cho cả hai mức |
| Cloud artifact list thiếu integration | Vẫn phải chạy và mang về 03-integration-results.md |
| Native verify báo nested file chưa commit | Kiểm tra git ls-files, git show HEAD, rồi dùng WSL verifier |
| Windows PowerShell parse lab.ps1 lỗi ở ký tự <target> | Chuẩn hóa file sang UTF-8 có BOM; không đổi logic script |
| Commit nhầm model/runtime | Kiểm tra git diff --cached --name-only trước commit |
| Pipeline bị khai báo sai là real | Ghi rõ toy/stub; stub không mất điểm |

## 9. Checklist hoàn tất

- [ ] Python >=3.10 chạy được trong PowerShell.
- [ ] hardware.json có trong repo.
- [ ] models/active.json hợp lệ và có primary/compare.
- [ ] Hai quantization có benchmark đầy đủ.
- [ ] TTFT và TPOT được báo riêng.
- [ ] 01-quickstart-results.md đã có observation.
- [ ] 01-tuning-tg128.md đã có explanation.
- [ ] Server smoke trả completion và metric non-zero.
- [ ] Load test 10 users đã chạy đủ 60 giây.
- [ ] Load test 50 users đã chạy đủ 60 giây.
- [ ] metrics đã overlap với load-50.
- [ ] 02-server-batching-u50.md đã có observation.
- [ ] 02-server-results.md đã có saturation reading.
- [ ] Pipeline chạy đủ 3 query.
- [ ] 03-integration-results.md đã khai báo real/stub và bottleneck.
- [ ] Reflection §§1–5 đã điền, số liệu khớp report.
- [ ] Không còn required -- replace this line.
- [ ] Có đủ 5 screenshots.
- [ ] verify exit 0 trong môi trường phù hợp.
- [ ] Staged/committed paths chỉ nằm trong phạm vi cho phép.
- [ ] Không commit .venv, runtime/, models/*.gguf hoặc source build.
- [ ] Repo GitHub ở chế độ public.
