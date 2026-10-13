# Lab 25B — Report

**Họ tên / nhóm:** Đinh Tiến Mạnh - 2A202602458  **Commit:** 7519d03  **Điểm tự động (`make grade`):** 84.0 / 88 (Public: 84.0 / 84, Hidden: 0.0 / 4)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | **PASS** |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.911 | 1.059 | 0.008 | 2.161 | 0.0 | **PASS** |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.738 | 1.058 | 0.034 | 2.141 | 0.0 | **PASS** |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | **PASS** |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | null | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | null | **PASS** |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.018 | 2.196 | 0.0 | **PASS** |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Dựa trên G4 và G6: median service của provider là 1.2–1.8s. Đặt 8.0s giúp cắt sớm các request bị treo ở G6/G4 để kịp failover trong turn deadline 30s mà không gây timeout giả. |
| max_attempts_per_provider | 2 | Dựa trên G2, G3: chỉ cho phép 1 lần retry trên cùng 1 provider (tổng 2 attempts) trước khi failover. Giúp giảm `retry_ratio` ở G3 xuống 0.034 (so với naive là 0.164). |
| backoff_base_s / cap_s | 0.25 / 4.0 | Tránh thundering herd khi provider bắt đầu hồi phục; cap 4.0s đảm bảo retry đủ nhanh để hoàn thành trong deadline 30s của người dùng. |
| retry_budget_ratio | 0.1 | Dựa trên G3: giới hạn lượng retry không vượt quá 10% lưu lượng ban đầu. Giữ `retry_ratio` luôn ở mức an toàn 0.034 <= 0.10. |
| breaker.window_size / min_calls | 20 / 10 | Dựa trên G2 và G5: window 20 đủ nhanh để bắt sự cố, min_calls 10 ngăn ngừa việc mở breaker nhầm khi lưu lượng thấp lúc đêm muộn. |
| breaker.failure_rate_threshold | 0.5 | Dựa trên G2: khi 50% call trong window bị lỗi 529/timeout thì lập tức OPEN để chuyển tải sang `alpha-eu` và `beta`. |
| breaker.slow_call_s | 5.0 | Dựa trên G4: thời gian xử lý bình thường là 1.2s; nếu >= 5.0s tức là vùng bị suy thoái (slow region), trip breaker giúp p95 giảm từ 13.76s xuống 5.56s. |
| breaker.open_duration_s | 10.0 | Dựa trên G2: cho provider 10 giây để hạ tải và xả hàng đợi trước khi bắt đầu probe thăm dò. |
| breaker.half_open_max_probes | 2 | Dựa trên G2: chỉ thả tối đa 2 request thăm dò khi HALF_OPEN, ngăn chặn thundering herd làm sập lại provider vừa mới phục hồi. |

Sơ đồ quyết định của `on_error` (loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

```text
                               [ on_error ]
                                     |
                          Phân loại lỗi (classify)
                                     |
     +-------------------------------+-------------------------------+
     |                               |                               |
[ CLIENT_ERROR ]           [ RATE_LIMITED / QUOTA ]       [ OVERLOADED / SERVER / TIMEOUT ]
 (400, 401, 422)               (429 with retry-after           (529, 503, 500, status 0)
     |                             or quota_exhausted)                       |
  Breaker: release                   |                               Breaker: on_failure
  Action: GiveUp                     +-----------------------+               |
  (Không retry,              [ RATE_LIMITED ]       [ QUOTA ]        Đủ điều kiện retry?
   không failover)                   |                  |             - tries < max_attempts (2)
                             Breaker: release    Breaker: release     - budget cho phép (>= 1 token)
                             Cooldown:           Cooldown:            - còn kịp deadline (now+delay+1 <= deadline)
                             now + retry_after   now + 300s                  |
                                     |                  |                    +---------------+
                                     +---------+--------+                    |               |
                                               |                           [ Có ]          [ Không ]
                                       Failover ngay lập tức                 |               |
                                       (idx += 1, tries = 0,                 |           Failover sang
                                        delay = 0.0)                         |           provider kế tiếp
                                                                             |           (idx += 1, tries = 0,
                                                                             |            delay = 0.0)
                                                                             v
                                                                   Retry cùng provider
                                                                   (delay = full_jitter_backoff)
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối:**
   Khi giả lập breaker cấu hình sai (`breaker.min_calls: 1000000000`) và đặt `max_attempts_per_provider: 5`:
   - Nếu **giữ retry budget** (1a): Ở G2, `availability = 99.96%`, `amplification = 1.382`, `retry_ratio = 0.058`, `recovery = 0.0s`. Ở G3, `availability = 100%`, `amplification = 1.135`, `retry_ratio = 0.067`.
   - Nếu **tắt retry budget** (1b): Ở G2, `availability` sụt còn `96.49%`, `amplification` tăng vọt lên `1.672`, `retry_ratio` tăng vọt lên `0.491` và hệ thống mất `30.0s` mới hồi phục! Ở G3, `retry_ratio` tăng vọt lên `0.162` (vi phạm ngưỡng 0.10).
   - *Kết luận:* Khi breaker hoạt động đúng, nó là chốt chặn cắt lưu lượng xấu sớm nhất. Nhưng khi breaker gặp sự cố cấu hình sai hoặc chưa đủ số mẫu (`min_calls`), **Retry Budget chính là phòng tuyến độc lập cuối cùng** giới hạn cứng tổng lượng retry $\le 10\%$, ngăn chặn triệt để thảm họa retry storm. Vì vậy kiến trúc tin cậy bắt buộc phải có cả hai lớp bảo vệ song song (defense-in-depth).

2. **`attempt_timeout_s: 30`:**
   Khi cấu hình `attempt_timeout_s: 30.0` (bằng turn deadline):
   - Ở G4 (`slow_region`), `availability = 100%`, `p95 = 5.683s`, `amplification = 1.014`.
   - Nhưng ở G6 (`provider_outage`), `availability` sụt xuống chỉ còn **`94.88%`** (thất bại vì yêu cầu $\ge 95\%$), độ sẵn sàng giảm mạnh do các request bị treo đến 30s trên cụm provider A trước khi client nhận ra lỗi để failover, dẫn đến khách hàng bị quá hạn turn deadline.
   - *Kết luận:* Timeout "đúng" **tuyệt đối không được đặt bằng turn deadline (30s)**. Timeout phải được tính toán dựa trên phân vị $p99$ của thời gian xử lý bình thường của provider (median 1.2–1.8s) cộng một dung sai an toàn hợp lý (khoảng 6–8s), nhằm cho phép client fail fast và còn đủ thời gian failover sang provider dự phòng.

3. **Bỏ lọc tool calling khỏi `ladder()`:**
   Khi bỏ lọc năng lực tool calling trong hàm `ladder()` và chạy lại băng G6:
   - `availability` tụt thảm hại từ `100%` xuống chỉ còn **`89.27%`**, đồng thời xuất hiện tới **`256` vi phạm `incompatible_routes`** do gửi tool call sang `gamma-mini` (model không hỗ trợ tool).
   - *Góc nhìn Product Manager:* "Hệ thống đã điều hướng sai 256 yêu cầu hoàn tiền của khách hàng vào một model giá rẻ không có khả năng thực thi tác vụ ngân hàng. Hậu quả là 256 khách hàng này nhận trải nghiệm lỗi sập tính năng, gây bức xúc, mất lòng tin nghiêm trọng vào thương hiệu và trực tiếp làm tăng chi phí vận hành khi khách phải gọi điện khiếu nại lên tổng đài người thật."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1,574,200 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

**Trả lời:**
- *Vì sao chỉ idempotency key là chưa đủ?* Idempotency key chỉ bảo vệ phía nhà cung cấp dịch vụ bên ngoài (Payment/Email) không thực thi hai lần nếu nhận cùng một request. Nhưng nếu không có **checkpoint**, khi worker bị crash, worker mới sẽ không biết quy trình đã chạy đến đâu và buộc phải chạy lại toàn bộ từ bước 1 $\rightarrow$ gây lãng phí chi phí gọi LLM vô ích (`wasted_llm_calls`), tăng độ trễ và làm chậm tiến độ hoàn tiền.
- *Vì sao chỉ checkpoint là chưa đủ?* Checkpoint chỉ giúp lưu trạng thái cục bộ của worker. Nếu worker gọi thanh toán thành công nhưng bị rớt mạng ngay trước khi nhận response hoặc crash trước khi kịp ghi checkpoint, worker mới sẽ thực hiện lại bước thanh toán. Nếu không có **idempotency key ổn định**, cổng thanh toán sẽ coi đó là giao dịch mới và trừ tiền lần 2 $\rightarrow$ gây hoàn tiền trùng lặp và thất thoát tài chính nghiêm trọng (`overpaid_cents`).
- *Điều gì xảy ra nếu ghi checkpoint TRƯỚC khi gọi payment?* Nếu ghi checkpoint hoàn tiền thành công trước khi gọi API payment thực tế, và ngay lúc đó worker bị crash (hoặc API thanh toán thất bại), worker mới khởi động lại sẽ kiểm tra checkpoint, thấy bước hoàn tiền "đã xong" và bỏ qua bước này để nhảy sang gửi email $\rightarrow$ Dẫn đến **mất hoàn tiền (`lost_refunds > 0`)**: khách hàng nhận được email thông báo đã hoàn tiền nhưng thực tế tài khoản ngân hàng không nhận được một xu nào!

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | paged (delay 1m) | page | 27 min |
| provider_outage | page | paged (delay 0m) | page | 4 min |
| deploy_blip | none | paged (false alarm) | none | — |
| slow_leak | ticket | none (bỏ sót) | ticket | 1337 min |
| degraded_evening | page | paged (delay 214m) | page | 291 min |
| night_spike | none | paged (false alarm) | none | — |
| small_bump | none | paged (false alarm) | none | — |

**Phân tích Error Budget 30 ngày:**
- Tổng số requests: `25,901,755`
- Số lỗi tối đa cho phép (`allowed_bad` với SLO 99.5%): `129,508.8`
- Số lỗi thực tế: `55,618`
- Availability thực tế: **`99.7853%`** (vượt mục tiêu SLO 99.50%)
- Tỷ lệ budget đã tiêu thụ: **`42.95%`**, ngân sách lỗi còn lại: **`57.05%`**.
- Quyết định đóng băng release: **Không đóng băng (`freeze_releases: False`)** vì ngân sách lỗi còn hơn 57%.
- *Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?* Sự cố `small_bump` chỉ diễn ra trong 20 phút và chỉ tiêu tốn một phần cực nhỏ ngân sách lỗi (~0.5% budget tháng), không đủ đe dọa cam kết SLO 30 ngày. Đánh thức kỹ sư on-call cho những dao động ngắn tự hết sẽ gây mệt mỏi cảnh báo (alert fatigue), giảm năng suất làm việc và phân tán sự chú ý khỏi các sự cố nghiêm trọng thực sự. Vấn đề này chỉ cần ghi nhận qua dashboard hoặc gửi ticket xem xét trong giờ hành chính.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**
Vào thời điểm $t=120s$, cụm máy chủ chính `alpha` bị mất đột ngột 75% năng lực xử lý phần cứng GPU trong 3 phút, gây ùn ứ hàng đợi và trả về mã lỗi 529. Bản `NaiveClient` với cơ chế retry 3 tầng chồng chéo không có jitter, không có circuit breaker và không có retry budget đã khuếch đại lưu lượng lên tới 9.9 lần, biến sự cố tạm thời thành thảm họa sập hệ thống kéo dài (metastable failure). Ngay cả sau khi phần cứng của `alpha` đã hồi phục hoàn toàn ở $t=300s$, hệ thống vẫn bị tê liệt thêm 155 giây và độ khả dụng trong toàn bộ sự cố chỉ đạt 45.6%.

**Ảnh hưởng:**
- Tổng cộng **1,285 lượt người dùng thất bại** (chỉ có 1,077 / 2,362 lượt thành công, availability tụt xuống 45.60%).
- Sự cố kéo dài tổng cộng **335 giây** (từ $t=120s$ đến $t=455s$).
- Chi phí vận hành trung bình đội lên gấp hơn 2 lần (**4.627** so với mức chuẩn 2.10).
- Tỷ lệ request thất bại trong sự cố là **54.4%** (1,285 / 2,362 turns). Với cam kết SLO 99.5% (ngân sách lỗi tối đa cho phép là 0.5%), mức tiêu thụ error budget lên tới khoảng **10,880%** (gấp ~109 lần ngân sách lỗi cho phép trong khoảng thời gian diễn ra sự cố).

**Timeline (giây ảo — trích xuất từ `naive_G2_overload_storm_timeseries.csv`):**

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | Alpha mất 75% GPU, hàng đợi bắt đầu nghẽn | `attempts_alpha = 5`, `calls_ok = 7`, bắt đầu xuất hiện độ trễ tăng cao. |
| 136 | Bão retry storm bùng phát | `attempts_alpha` tăng vọt từ 24 lên **53**, rồi chạm **82** ($t=137$) và **122** attempts/s ($t=138$). |
| 150 | Hệ thống sập hoàn toàn | `calls_ok = 0`, `attempts_alpha = 153`, hàng loạt client timeout và lỗi 529 xuất hiện. |
| 300 | Alpha hồi phục 100% dung lượng phần cứng | `calls_ok = 0`, `attempts_alpha = 115`, hệ thống vẫn tê liệt hoàn toàn. |
| 350 | Hàng đợi Alpha vẫn nghẽn bởi request rác | `calls_ok = 0`, client tiếp tục gửi retry và timeout dồn dập. |
| 455 | Hàng đợi rác được xả xong, hệ thống bắt đầu phục hồi | `calls_ok = 4`, lần đầu tiên có request thành công sau 335 giây gián đoạn. |

**Trigger vs root cause:**
- **Trigger:** Cụm GPU của nhà cung cấp `alpha` bị mất 75% tài nguyên tính toán từ $t=120s$ đến $t=300s$.
- **Root cause:** 
  1. Kiến trúc client có **3 tầng retry độc lập lồng nhau** (app $\times$ gateway $\times$ SDK), tạo ra hệ số nhân lên tới 36 lần thử cho mỗi cuộc gọi logic.
  2. Không có **Circuit Breaker** để ngắt tải khi provider suy thoái và không có cơ chế failover sang các cụm dự phòng (`alpha-eu`, `beta`).
  3. Không có **Retry Budget** để giới hạn tỷ lệ retry và không có **Jitter** khiến các request dội lại cùng một thời điểm.
- **Cơ chế Metastable Failure (Vòng lặp tự duy trì):** Khi client timeout, hàng đợi server vẫn tiếp tục xử lý các request rác mà client đã bỏ cuộc. Lượng retry mới liên tục bơm vào với tốc độ $> 100$ req/s, vượt xa năng lực xử lý tối đa của server (18 slots $\approx$ 15 rps). Do đó, ngay cả khi GPU đã phục hồi ở $t=300s$, server vẫn ngập chìm trong hàng đợi các request đã hết hạn từ trước, khiến hệ thống tự duy trì trạng thái sập thêm 155 giây.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Làm tệ hơn:* Retry cố định không jitter làm đồng pha tải; thiếu circuit breaker khiến traffic liên tục đâm vào provider đang nghẽn; việc server tiếp tục xử lý request sau khi client đã timeout gây lãng phí năng lực quý giá.
- *Giúp giải quyết:* Triển khai Circuit Breaker sliding-window phát hiện lỗi và chuyển tải sang `alpha-eu`/`beta`; áp dụng Retry Budget kìm hãm retry $\le 10\%$; áp dụng Full Jitter Backoff làm phẳng đỉnh tải; fail fast ngay tại client khi hết hạn turn deadline.

**Action items (cụ thể, có owner, đo được):**

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế toàn bộ `NaiveClient` bằng `ResilientClient` có Circuit Breaker và Retry Budget trong toàn bộ microservices. | Prevent | SRE Team / Dev Lead | Đạt `amplification <= 1.10` và `availability >= 98%` trên băng G2 trong CI. |
| 2 | Thiết lập cảnh báo đa cửa sổ Multi-window Multi-burn-rate Alerting theo chuẩn Google SRE. | Detect | Monitoring / SRE | Phát hiện sự cố outage trong $\le 5$ phút, $0$ cảnh báo giả trên dữ liệu 30 ngày. |
| 3 | Triển khai deadline propagation (truyền timeout header sang server) để server tự hủy request khi client đã hủy/timeout. | Mitigate | Backend Platform | `recovery_s = 0.0s` ngay khi provider phục hồi trong các bài test Game Day. |

## 7. Một điểm yếu còn lại

**Điểm yếu:** 
Khi **cả 4 provider đồng thời bị suy thoái hoặc phản hồi chậm** (ví dụ sự cố mạng diện rộng trên toàn cầu):
- Client sẽ lần lượt gửi request sang từng provider trong ladder (`alpha` $\rightarrow$ `alpha-eu` $\rightarrow$ `beta` $\rightarrow$ `gamma-mini`).
- Mỗi lần thử đều phải chờ một khoảng thời gian trước khi timeout hoặc breaker mở, dẫn đến tổng thời gian chờ tích lũy vượt quá turn deadline (30s) của người dùng, khiến toàn bộ request thất bại và chi phí tăng do các provider vẫn xử lý dở dang.

**Giải pháp khắc phục:**
1. **Hedging / Speculative Execution:** Sau khi gửi request tới primary provider một khoảng thời gian nhất định (ví dụ $p95 \approx 2.5s$) mà chưa nhận được phản hồi, client chủ động bắn thêm một request dự phòng song song sang secondary provider và lấy kết quả của bên nào về trước (có hủy request còn lại).
2. **Global Adaptive Concurrency Limiting:** Theo dõi độ trễ end-to-end trên toàn bộ hệ thống; khi nhận thấy độ trễ tổng thể tăng vọt trên mọi provider, chủ động từ chối sớm (shed load) một phần lưu lượng không quan trọng ngay tại cửa ngõ API Gateway để bảo vệ các request cốt lõi của người dùng.

## 8. Cache (Part G)

Số liệu từ `reports/cache/resilient.json` và `reports/cache/cached.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | **0.9124** | **0.9645** | **0.652** | **157** | **0** | **3** | **0.915** | **PASS** |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Starter đặt 0.80 dẫn tới 157 câu trả lời sai (`wrong_answers: 157` do `other_question`). Nâng lên 0.92 giúp đưa `wrong_answers` về đúng 0 mà vẫn duy trì tỷ lệ hit cao (65.2%). |
| ttl_s | 60.0 | Starter đặt 600s khiến khi bảng phí ship đổi âm thầm không báo webhook ở $t=480s$, cache phục vụ tới 80 câu trả lời cũ. Hạ xuống 60.0s giúp `outdated_answers` chỉ còn 3 (ngưỡng cho phép $\le 15$). |
| max_stale_s | 300.0 | Cho phép phục vụ câu trả lời cũ trong 120s mọi provider sập ($t=240-360s$). Đẩy `faq_availability` từ 87.33% lên 96.45% nhờ phục vụ 157 câu trả lời stale an toàn. |

**Thí nghiệm (kèm số liệu thực tế):**

1. **Ngưỡng similarity:** 
   - Với `threshold = 0.80`: `hit_ratio = 66.6%`, nhưng phát sinh tới **`wrong_answers = 157`** (`wrong_answers_by_reason={'other_question': 157}`), `cost/turn = 0.965`. Các câu hỏi có ý nghĩa khác nhau như "thời gian hoàn tiền" và "chính sách hoàn tiền" bị nhận diện nhầm là một.
   - Với `threshold = 0.92`: `hit_ratio = 65.2%`, **`wrong_answers = 0`**, `outdated = 3`, `cost/turn = 0.915` (tối ưu nhất).
   - Với `threshold = 0.97`: `hit_ratio` bị bóp nghẹt xuống còn `58.1%`, `cost/turn` tăng lên `0.964` vì bỏ lỡ nhiều câu hỏi đồng nghĩa hợp lệ.
2. **TTL:** 
   - Chạy với `ttl_s = 30.0`: `hit_ratio = 59.2%`, `outdated_answers = 2`.
   - Chạy với `ttl_s = 60.0`: `hit_ratio = 65.2%`, `outdated_answers = 3` ($\le 15$, đạt chuẩn).
   - Chạy với `ttl_s = 300.0`: `outdated_answers` vọt lên tới **`95`** (vi phạm nghiêm trọng ngưỡng 15).
   - *Giải thích:* Webhook chỉ hoạt động khi hệ thống nguồn (CMS) chủ động gửi tín hiệu thông báo. Khi đơn vị vận chuyển âm thầm thay đổi bảng phí ship trên hệ thống của họ mà không có webhook (`silent_updates`), cache hoàn toàn không có cách nào biết nội dung đã thay đổi. **TTL chính là cơ chế bảo vệ duy nhất** giúp tự động giải phóng dữ liệu sau một khoảng thời gian định trước để lấy lại dữ liệu mới nhất.
3. **Stale-if-error:** 
   - Khi cấu hình `max_stale_s: 0.0` (tắt stale-if-error): Trong 2 phút mọi provider sập ($t=240-360s$), các câu hỏi FAQ quá hạn TTL đều thất bại, khiến `faq_availability` sụt xuống chỉ còn **`87.33%`** (thất bại so với yêu cầu $\ge 93\%$) và `stale_served = 0`.
   - Khi bật `max_stale_s: 300.0`: Hệ thống phục vụ an toàn **`157`** câu trả lời stale trong lúc provider ngừng hoạt động, đưa `faq_availability` lên mức xuất sắc **`96.45%`**.

**Trả lời câu hỏi:**
- *Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?* 
  1. **Bảo mật và rò rỉ dữ liệu (Security & Privacy):** Độ tương đồng semantic giữa hai câu hỏi cá nhân của hai người dùng khác nhau có thể rất cao (ví dụ: "Đơn #123 của tôi đâu" và "Đơn #456 của tôi đâu"). Nếu cache theo semantic có thể vô tình trả thông tin cá nhân/địa chỉ của khách hàng này cho khách hàng khác.
  2. **Dữ liệu động thay đổi liên tục:** Trạng thái đơn hàng, số dư tài khoản, điểm tích lũy thay đổi theo thời gian thực (đơn đang giao, đã giao, hủy...). Việc cache dữ liệu này sẽ dẫn tới trả lời sai lệch thực tế, gây hiểu lầm nghiêm trọng cho khách hàng.
- *Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?*
  - Nếu chỉ alert trên `turn_availability` hoặc `user_success_rate`, dashboard vẫn sẽ xanh vì cache đang trả lời tốt cho các câu hỏi FAQ.
  - On-call bắt buộc phải thiết lập alert dựa trên **Upstream Provider Error Rate** (tỷ lệ lỗi trả về từ các provider thực tế), **Upstream Request Rate / Breaker State** (khi Circuit Breaker của các provider chuyển sang trạng thái `OPEN`), và **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_rate` hoặc `stale_served_ratio > 0`)**. Khi chỉ số `stale_served` tăng vọt hoặc số lượng call thành công tới provider rớt về 0, hệ thống cảnh báo lập tức page kỹ sư trực on-call kể cả khi khách hàng bên ngoài vẫn đang nhận được phản hồi từ cache.

