# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Tuấn Anh
**MSSV:** 2A202602535
**Cohort:** K4-L3
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux (Fedora 44 / Linux 6.19.10-300.fc44.x86_64)
- **CPU:** 12th Gen Intel(R) Core(TM) i7-12700H
- **Cores:** 14 physical / 20 logical
- **CPU extensions:** AVX2
- **RAM:** 15.3 GB
- **Accelerator:** CPU only
- **llama.cpp asset đã tải:** llama-b10488-bin-ubuntu-x64.tar.gz
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Đường truyền mạng ban đầu bị ngắt kết nối TLS khi tải file lớn từ Hugging Face. Em chuyển sang model Qwen3.5 0.8B (~0.9 GB) để tối ưu tải nhanh hơn, dùng curl resume file runtime và tải trọn vẹn 2 bản quant.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2177 | 259 / 361 | 40.2 / 42.1 | 2743 / 3013 / 3013 | 24.9 |
| UD-Q2_K_XL | 0.39 | 2041 | 333 / 388 | 40.0 / 40.6 | 2853 / 2906 / 2906 | 25.0 |

**Quan sát** (≤ 60 chữ): 2-bit giảm 22% dung lượng nhưng decode tương đương (25.0 vs 24.9 tok/s), TTFT chậm hơn 28% (333ms vs 259ms) do chi phí dequantize trên CPU. Trên RAM 15.3GB, việc hy sinh ngữ nghĩa để bớt 110MB là không đáng.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.68 | 12000 | 21000 | 22000 | 8.0 | 0.0% |
| 50 | 0.74 | 32000 | 55000 | 57000 | 21.3 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.09×
- **P95 tăng:** 2.62×
- **Effective concurrency ở 50 users:** 21.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 4.00 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa ở ~10-15 users. Bằng chứng: RPS tăng 1.09x trong khi P95 phình 2.62x (21s lên 55s), 46 requests bị deferred. Trễ tăng thêm là queue time do vượt 4 slot decode. Muốn nâng goodput@SLO, em sẽ tăng `--parallel` lên 8 slot trước vì RAM còn dư nhiều.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Embedding & Chunking | stub |
| N17 Data pipeline | Keyword retrieval | stub |
| N18 Lakehouse | Reranker | stub |
| N19 Vector + features | In-memory toy docs | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 7439.3 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection** (≤ 60 chữ): Bottleneck 100% tại LLM decode (~40ms/tok) do CPU memory bandwidth bound. Muốn giảm 2x latency, em sẽ giảm max_tokens và đặt LAB_N_THREADS=7 kết hợp prefix caching cho system prompt.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Giảm số luồng CPU từ `-t 14` (toàn bộ 14 physical cores) xuống `-t 7` (chỉ dùng cụm Performance cores)

```
before:  26.3 tok/s (-t 14)
after:   29.7 tok/s (-t 7)
speedup: 1.13x
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Chip Intel Core i7-12700H sở hữu kiến trúc hybrid kết hợp 6 Performance cores (P-cores) xung nhịp cao và 8 Efficient cores (E-cores) tiết kiệm điện. Khi cấu hình mặc định dùng tất cả physical cores (`-t 14`), khối lượng công việc nhân ma trận song song bị chia đều cho cả P-core lẫn E-core. Vì thuật toán cần đồng bộ thread tại các barrier, các E-core chậm hơn đã trở thành điểm nghẽn straggler kéo lùi toàn bộ cụm tính toán, đồng thời gây xung đột trên kênh RAM dual-channel, khiến tốc độ giảm xuống 26.3 tok/s.

Khi hạ số luồng xuống `-t 7`, toàn bộ tính toán decode được gom trọn vẹn vào các P-cores có xung nhịp cao và bộ đệm L3 dồi dào, loại bỏ hoàn toàn chi phí chờ đồng bộ từ E-core và giảm áp lực nghẽn bộ nhớ. Kết quả throughput tăng vọt lên 29.7 tok/s (tăng 1.13x so với `-t 14` và 2.44x so với oversubscription `-t 40`).

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 (`make sweep-ctx`), B3 (Reflection §6 before/after), B4 (Challenge C7: Prefill vs Context Length Scaling), B5 (Challenge C8: `make semantic-cache-offline`).

**Numbers:**

```
before:  1827.7 ms prefill at 256 tokens (140.1 tok/s)
after:   69844.0 ms prefill at 8192 tokens (117.3 tok/s)
speedup: 1.19x super-linear degradation (O(N^2) attention overhead vs linear)
Semantic Cache (C8): 38% hit rate (3/8 hits), 0 ms latency on hit (100% compute saved)
```

**Điều này nói lên gì mà deck chưa nói:**

Trong slide lý thuyết, chúng ta học rằng context window lớn (8k, 32k, 128k) giúp đưa nhiều tài liệu vào RAG hơn. Tuy nhiên, số liệu thực nghiệm đo đạc từ `make sweep-ctx` phơi bày một cái giá đắt đỏ mà lý thuyết thường nói giảm: **Độ trễ TTFT bùng nổ theo cấp số phi tuyến (super-linear)**.

Khi tăng context từ 256 lên 8192 tokens (gấp 32 lần), thời gian prefill không chỉ tăng 32 lần (~58 giây) mà thực tế tăng vọt lên **gần 70 giây (69.84s, gấp 38.2 lần, tức 1.19x so với tỷ lệ tuyến tính)**. Thông lượng prefill tụt từ 152 tok/s xuống còn 117 tok/s do chi phí tính toán ma trận Attention (N^2)$ bắt đầu lấn át.

Điều này mang lại bài học kiến trúc sống còn cho RAG:
1. **Không nhồi nhét chunk (Context Budgeting):** Nếu một hệ thống RAG nhồi 10-15 chunks (~6000 tokens), người dùng phải ngồi chờ hơn 45 giây chỉ để nhận token đầu tiên. Giải pháp đúng đắn là dùng Reranker lọc gọn còn 2-3 chunks chất lượng cao (<= 1500 tokens, TTFT <= 10s).
2. **Cần tách biệt Prefill/Decode (Disaggregated Serving):** Prefill ở 8k tokens ngốn 100% tài nguyên CPU/GPU trong suốt 70 giây. Nếu gộp chung một server, tất cả luồng decode khác sẽ bị bỏ đói (starved) hoàn toàn trong 70 giây đó.
3. **Giá trị của Semantic Cache (C8):** Qua thực nghiệm `make semantic-cache-offline`, tầng Cache ngữ nghĩa (Layer 1) chặn được 38% câu hỏi paraphrase phổ biến, trả lời ngay trong 0 ms và triệt tiêu hoàn toàn chi phí prefill/decode nặng nề này.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều ngạc nhiên nhất là việc tăng thêm core CPU không hề giúp tăng tốc LLM decode trên máy laptop, mà ngược lại còn làm chậm đi do độ trễ đồng bộ giữa P-core và E-core cùng nút thắt cổ chai băng thông bộ nhớ RAM.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng AI Assistant hỗ trợ debug mạng tải model, đọc hiểu tài liệu hướng dẫn và phân tích cơ chế đo lường độ trễ serving.
