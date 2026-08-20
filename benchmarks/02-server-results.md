# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=12` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 15 | 0.26 | 27000 | 56000 | 56000 | 7.9 | 0.0% |
| 50 | 20 | 0.33 | 27000 | 61000 | 61000 | 9.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.25x** (25% of linear) |
| P95 latency | **1.09x** |
| Effective concurrency at 50 users | 9.9 vs `--parallel 4` slots (occupancy/slot ratio 2.48) |

**Saturated.** Throughput delivered only 1.25x for 5x the offered load, and effective concurrency (9.9) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (1.09x vs 1.25x), so this server still has headroom at 50 users.

> **Small sample.** Only 15 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading (required -- replace this line)

Server đã bão hòa ngay từ mức tải thấp, chứ không phải chỉ ở 50 người dùng. Con số
thuyết phục tôi nhất là **effective concurrency ở 10 người dùng đã là 7.9**, trong khi
server chỉ cấu hình `--parallel 4` slot -- tức tỷ lệ occupancy/slot đã là ~2.0x ngay từ
điểm đo đầu tiên (và tăng lên 2.48x ở 50 người dùng). Nói cách khác, ngay cả ở tải thấp
nhất đã đo, số request thực sự "đang bay" đã gấp đôi số slot decode có sẵn -- phần lớn
trong số đó không được xử lý song song mà đang xếp hàng chờ slot trống. Bằng chứng thứ
hai củng cố điều này: đưa tải lên gấp 5x (10 -> 50 users) chỉ kéo throughput lên 1.25x
(0.26 -> 0.33 RPS) -- xa mức tuyến tính -- đúng như một hệ thống đã hết chỗ chứa cho
công việc song song bổ sung.

Điều thú vị là P95 latency gần như không đổi (1.09x, từ 56s lên 61s) dù throughput tăng
1.25x -- nghĩa là bản thân việc decode/token vẫn còn dư địa (GPU/CPU chưa bị nghẽn về
compute), cái đang thiếu là **số slot song song** để khai thác dư địa đó. Đây là dấu
hiệu kinh điển của "queue-bound" chứ không phải "compute-bound".

Vì vậy, knob đầu tiên tôi sẽ chỉnh là **tăng `--parallel`** (ví dụ 4 -> 8), chứ không
phải `--ctx-size`, `-t` (threads) hay `-ngl`. Lý do:
- Occupancy/slot ratio đã > 2x ngay ở tải thấp -> nút thắt nằm ở *số lượng slot*, không
  phải ở tốc độ decode của từng slot.
- P95 latency còn nhiều headroom (chỉ tăng 9% trong khi RPS tăng 25%) -> có thể "mua"
  thêm goodput bằng cách thêm slot mà chưa chắc đã vi phạm SLO độ trễ.
- Tăng `threads`/`ngl` sẽ chỉ tăng tốc mỗi lượt decode đơn lẻ (vốn không phải điểm nghẽn
  hiện tại), còn tăng `ctx-size` sẽ tốn thêm VRAM/RAM cho mỗi slot mà không giải quyết
  vấn đề số lượng request được xử lý đồng thời.

Việc cần làm tiếp theo: tăng `--parallel`, chạy lại load test dài hơn (`-t 3m`) để có đủ
mẫu, rồi theo dõi P95 và `/metrics` (slot utilisation) -- nếu P95 bắt đầu tăng nhanh hơn
throughput sau khi tăng slot, đó là dấu hiệu đã chạm trần compute thật sự và lúc đó mới
nên cân nhắc các knob khác (giảm `ctx-size` để nhồi thêm slot vào VRAM, quant nhẹ hơn, …).
