# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **12 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 16.1 | 95% |
| 6 | 16.7 | 99% |
| 12 | 16.9 | 100% |
| 16 | 15.2 | 90% |
| 32 | 12.8 | 76% |

**Best**: `-t 12` at 16.9 tok/s
**Slowest tested**: `-t 32` at 12.8 tok/s (1.32x spread)
**Against the physical-core default** (`-t 12`, 16.9 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=12 make bench
```

## Your explanation (required -- replace this line)

_Where is the knee, and why there? If the peak sits at your physical core count
and drops above it, say what the extra threads are competing for. If your curve
does something else -- flat, or still climbing at 2x logical cores -- say that
instead and reason about why. A result that contradicts the expected shape is
worth more than one that matches it, as long as you explain it._

Điểm gãy nằm chính xác tại mức 12 luồng (threads=12) với tốc độ đạt tối đa 16.9 tok/s. Nói cách khác, nó đạt đỉnh tại số lượng nhân vật lý (12 physical cores) và sụt giảm ngay lập tức khi vượt qua mốc này (chỉ còn 15.2 tok/s ở mức 16 luồng và tụt thê thảm xuống 12.8 tok/s ở mức 32 luồng). Khi bạn ép hệ thống sử dụng nhiều hơn số nhân vật lý (tức là bắt đầu dùng đến các luồng logic/luồng ảo do công nghệ Hyper-Threading hoặc SMT tạo ra), các luồng bổ sung này tranh chấp khốc liệt các tài nguyên phần cứng như bộ nhớ đệm, băng thông bộ nhớ và đơn vị thực thi.