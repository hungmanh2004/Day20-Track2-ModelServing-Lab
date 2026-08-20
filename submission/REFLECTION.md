# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** _Trần Mạnh Hùng 2A202601058_
**Cohort:** _<A20-K4>_
**Ngày submit:** _2026/08/21_

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** _Windows 11 (AMD64)_
- **CPU:** _12th Gen Intel(R) Core(TM) i5-1240P_
- **Cores:** _12 physical · 16 logical cores_
- **RAM:** _15.7 GB_
- **Accelerator:** _Vulkan_
- **llama.cpp asset đã tải:** _prebuilt release b10488  (llama-b10488-bin-win-vulkan-x64.zip)_
- **Model đã dùng:** _Gemma 4 E2B_ (`LAB_MODEL=`_gemma4-e2b_)
- **Quantization:** _UD-Q2_K_XL_ + _UD-Q4_K_XL_ (từ `models/active.json`)

**Chạy ở đâu:** _laptop của tôi_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

_Máy không có GPU rời nên dùng backend Vulkan (không phải CUDA) — probe tự chọn đúng.
`.\lab.ps1 serve` fail với lỗi `invalid argument: Tran\OneDrive` vì đường dẫn repo nằm
trong OneDrive có khoảng trắng: `os.execv` trên Windows không quote argv, tách nhầm
đường dẫn. Sửa `serve.py` để dùng `subprocess.run` thay `os.execv` trên Windows._

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 23553 | 944 / 4434 | 63.8 / 65.3 | 4915 / 8449 / 8449 | 15.7 |
| UD-Q2_K_XL | 2.24 | 15358 | 1250 / 15998 | 111.0 / 112.7 | 8216 / 22865 / 22865 | 9.0 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

_2-bit **không** nhanh hơn — nó decode **chậm hơn 1.74x** (9.0 vs 15.7 tok/s) dù nhẹ hơn
0.73GB, vì máy tôi (CPU yếu, không GPU rời) bị giới hạn bởi compute/dequantization chứ
không phải memory bandwidth, nên bit thấp hơn không giúp được gì. Không đáng đánh đổi —
mất cả tốc độ lẫn chất lượng để đổi lấy 25% dung lượng ít hơn, nên tôi không đi tiếp
việc so sánh chất lượng câu trả lời song song nữa; kết quả tốc độ đã đủ quyết định._

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.26 | 27000 | 56000 | 56000 | 7.9 | 0.0% |
| 50 | 0.33 | 27000 | 61000 | 61000 | 9.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** _1.25×_
- **P95 tăng:** _1.09×_
- **Effective concurrency ở 50 users:** _9.9_ so với `--parallel` = _4_ slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): _3.92_ / _4_ slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

_Bão hoà đã bắt đầu ở ≤10 users: effective concurrency (7.9) đã ~2x số slot (4), và
`n_busy_slots_per_decode` đạt đỉnh 3.92/4 (98%) — slot gần như luôn bận. Biết đó là
**queue time** (không phải compute time) nhờ `requests_deferred` = 46 > 0 cùng lúc:
request chờ vì hết slot, trong khi P95 chỉ tăng 1.09x dù RPS tăng 1.25x — compute mỗi
slot vẫn còn dư địa. Knob đổi trước: **tăng `--parallel`**, vì nút thắt là số slot chứ
không phải tốc độ decode._

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | chạy `localhost` (không k8s/Compose) | stub |
| N17 Data pipeline | `TOY_DOCS` — list Python in-memory (không Airflow DAG) | stub |
| N18 Lakehouse | không bảng persist nào, chỉ list trong RAM (không SQLite/Delta) | stub |
| N19 Vector + features | keyword overlap trên `TOY_DOCS` (không embed server, không vector index) | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: _0.0 ms_
- retrieve: _0.2 ms_
- llm: _5299.7 ms_
- **stage chiếm nhiều nhất:** _llm_ (_100%_ của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

_Bottleneck là stage `llm` (100% total) — đúng như kỳ vọng, vì embed/retrieve chỉ là
keyword overlap trên 6 doc toy nên gần như miễn phí (0.0/0.2ms). Muốn giảm 2× thì phải
tấn công `llm`: giảm `max_tokens`, dùng quant/threads nhanh hơn, hoặc tăng `--parallel`
— tối ưu retrieval vô nghĩa vì nó không đóng góp gì vào tổng latency._

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** _hạ `-t` từ 32 (ép oversubscribe 2x số luồng logic) xuống 12 (đúng bằng số nhân vật lý)_

```
before:  12.8 tok/s  (-t 32)
after:   16.9 tok/s  (-t 12)
speedup: 1.32×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

_CPU của tôi (i5-1240P) là kiến trúc lai P-core/E-core: 12 nhân vật lý (4 P-core có
Hyper-Threading + 8 E-core không có) nhưng chỉ 16 luồng logic. Giai đoạn decode (tg128)
bị giới hạn bởi **memory bandwidth** chứ không phải compute — mỗi token cần đọc lại
toàn bộ trọng số mô hình từ RAM. Với `-t 12`, mỗi luồng chạy trên một nhân vật lý riêng
nên tận dụng đúng số kênh thực thi/độ rộng bus bộ nhớ sẵn có, gần như không tranh chấp.
Vượt quá 12 (`-t 16`) đã bắt đầu dùng luồng SMT anh em trên các P-core — hai luồng này
chia sẻ chung băng thông bộ nhớ và cache của một nhân, nên không có compute "thêm" nào
thực sự được cấp phát, chỉ tranh chấp cache, kết quả giảm nhẹ (15.2 tok/s)._

_Ở `-t 32`, hệ thống phải ép 32 software thread chạy trên chỉ 16 hardware thread — tức
oversubscribe gấp đôi. OS phải liên tục context-switch giữa các luồng để chia sẻ core,
mỗi lần switch làm mất tính cache-resident của working set (trọng số + KV cache đang
dùng bị đẩy khỏi L2/L3), buộc phải đọc lại từ RAM nhiều hơn — vừa tốn chi phí lập lịch
vừa tăng áp lực memory bandwidth vốn đã là nút thắt. Đó là lý do tốc độ **giảm** (không
chỉ chững lại) khi thêm luồng — đúng như deck cảnh báo về oversubscription trên máy
memory-bandwidth-bound, không phải một kết quả bất ngờ._

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
