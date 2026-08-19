# 30. Reliability patterns cho agent

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Reliability patterns cho agent — 11 pattern: Checkpoint and resume, Retry with backoff, Circuit
breaker, Timeout budget, Dead-letter task, Fallback model, Graceful degradation, Deterministic state
machine, Termination guard, Saga for long-running agent tasks, Context rot mitigation.

## 2. Vấn đề cần giải quyết

Agent chạy nhiều bước, gọi nhiều tool/model, và có thể xử lý task kéo dài — mỗi điểm trong chuỗi đó
là một điểm có thể lỗi (model timeout, tool không phản hồi, mạng gián đoạn, agent lặp vòng lặp không
dừng). Không có bộ pattern reliability riêng cho đặc thù agent, hệ thống hoặc dừng hoàn toàn khi gặp
lỗi cục bộ (không tận dụng được phần đã hoàn thành), hoặc chạy vô hạn/tốn kém không kiểm soát được
(agent tự lặp lại hành động không hội tụ tới kết quả).

## 3. Bối cảnh sử dụng

Mọi agent chạy quá một lượt gọi model đơn lẻ, đặc biệt agent xử lý task dài (nhiều bước, nhiều tool
call) hoặc task có chi phí cao nếu phải làm lại từ đầu khi gặp lỗi giữa chừng.

## 4. Kiến trúc

```
Task bắt đầu
  → Bước 1 [checkpoint] → Bước 2 [checkpoint] → ... → Bước N
       ↓ lỗi                  ↓ lỗi
   [Retry/Circuit breaker]  [Retry/Circuit breaker]
       ↓ vẫn lỗi                ↓ vẫn lỗi
   [Dead-letter / Fallback model / Graceful degradation]

Toàn bộ luồng bị giới hạn bởi [Termination guard]: số vòng lặp, token, chi phí, thời gian, số tool
call, số lần handoff.
```

## 5. Thành phần

Checkpoint store (thường chính là State store, file 23); retry policy engine; circuit breaker state
(closed/open/half-open cho mỗi tool/agent); timeout configuration theo từng cấp (task/step/tool);
dead-letter queue; fallback model routing; termination guard counters.

## 6. Luồng xử lý chi tiết

**11 pattern:**

- **Checkpoint and resume (30.1)**: lưu state sau mỗi bước quan trọng (liên hệ trực tiếp mục 23.4)
  — nếu hệ thống gián đoạn (restart, crash), task resume đúng từ điểm dừng gần nhất thay vì phải bắt
  đầu lại toàn bộ từ đầu.
- **Retry with backoff (30.2)**: retry với giới hạn số lần rõ ràng và exponential backoff (thời gian
  chờ giữa các lần retry tăng dần) — tránh vừa bỏ cuộc quá sớm với lỗi thoáng qua, vừa tránh dội liên
  tục vào một dịch vụ đang gặp sự cố thực sự.
- **Circuit breaker (30.3)**: tạm ngừng gọi tới một tool hoặc agent đang lỗi liên tục (mở circuit)
  thay vì tiếp tục retry vô ích — cho dịch vụ đang gặp sự cố thời gian phục hồi, định kỳ thử lại
  (half-open) để phát hiện khi nó hoạt động trở lại.
- **Timeout budget (30.4)**: mỗi task, mỗi step, và mỗi tool call có deadline riêng, phân bổ rõ ràng
  từ ngân sách thời gian tổng của task — tránh một bước đơn lẻ treo vô thời hạn kéo theo toàn bộ task
  không bao giờ hoàn thành.
- **Dead-letter task (30.5)**: task thất bại sau khi đã retry hết mức được đưa vào một hàng chờ riêng
  để con người inspect hoặc xử lý lại thủ công, thay vì mất hẳn hoặc lặp lại retry vô ích mãi mãi.
- **Fallback model (30.6)**: nếu model chính timeout hoặc trả kết quả không đạt ngưỡng chất lượng,
  tự động chuyển sang một model dự phòng khác để hoàn thành task, đánh đổi lấy chất lượng có thể thấp
  hơn một chút để giữ được tính khả dụng.
- **Graceful degradation (30.7)**: nếu một nguồn dữ liệu/thành phần hỏng, hệ thống vẫn trả kết quả từ
  các nguồn còn lại và nêu rõ giới hạn của kết quả đó cho user, thay vì từ chối trả lời hoàn toàn.
- **Deterministic state machine (30.8)**: dùng code (không phải LLM) để kiểm soát các transition
  quan trọng trong luồng xử lý, chỉ dùng LLM cho các quyết định thực sự cần suy luận ngữ nghĩa — giảm
  rủi ro agent "trôi" khỏi luồng dự kiến ở những bước lẽ ra nên xác định (deterministic).
- **Termination guard (30.9)**: giới hạn cứng cho mọi vòng lặp agent — số vòng lặp, tổng token, chi
  phí, thời gian, số tool call, số lần handoff — đảm bảo agent luôn dừng lại trong một giới hạn tài
  nguyên xác định trước, không bao giờ chạy "vô hạn" dù logic bên trong có lỗi khiến nó không tự hội
  tụ.
- **Saga for long-running agent tasks (30.10)**: một task lớn được chia thành nhiều transaction nhỏ,
  mỗi transaction có một compensating action tương ứng (hành động bù trừ để hoàn tác) — nếu một bước
  giữa chừng thất bại, hệ thống chạy các compensating action của các bước đã hoàn thành trước đó theo
  thứ tự ngược lại, đưa hệ thống về trạng thái nhất quán thay vì để lại trạng thái nửa vời.
- **Context rot mitigation (30.11)**: đặt context rot (mục 24.5 — hiện tượng chất lượng suy giảm khi
  context quá dài/nhiễu) như một failure mode chính thức cần termination guard và observability theo
  dõi chủ động — agent chạy càng lâu càng cần chủ động compact/note-taking (mục 24.1-24.2) thay vì để
  context tự phình tới giới hạn cứng rồi mới xử lý.

## 7. State và dữ liệu

Checkpoint (mục 30.1) ghi trực tiếp vào State store (file 23) — cần đủ chi tiết để reconstruct chính
xác điểm dừng (không chỉ "đã xong bước mấy" mà cả dữ liệu trung gian cần thiết để tiếp tục đúng);
circuit breaker state (closed/open/half-open) thường là session state nhẹ, không cần durable; saga
cần lưu rõ trạng thái từng transaction con và compensating action tương ứng để biết cần hoàn tác đến
đâu nếu có lỗi.

## 8. Thuật toán liên quan

Exponential backoff (công thức tăng dần thời gian chờ, thường có jitter ngẫu nhiên để tránh nhiều
client retry đồng loạt cùng lúc); circuit breaker state machine (closed/open/half-open transition
logic); không có thuật toán ML — đây thuần là kỹ thuật kiến trúc hệ thống phân tán áp dụng vào agent.

## 9. Cách triển khai

1. Áp dụng Termination guard (mục 30.9) ngay từ bản đầu tiên của bất kỳ agent nào có vòng lặp — đây
   là lưới an toàn tối thiểu bắt buộc, không phải tính năng "thêm sau nếu cần".
2. Xác định rõ bước nào cần checkpoint (mục 30.1) dựa trên chi phí phải làm lại nếu mất — bước rẻ
   không cần checkpoint dày đặc, bước tốn kém (gọi nhiều tool, xử lý lâu) cần checkpoint ngay sau khi
   hoàn thành.
3. Với mọi lệnh gọi ra ngoài (tool, model, service khác), bọc bằng retry with backoff (mục 30.2) và
   circuit breaker (mục 30.3) — không gọi trực tiếp không có lớp bảo vệ.
4. Chỉ dùng saga (mục 30.10) khi task thực sự chia được thành các transaction có compensating action
   rõ ràng — nếu một bước không thể hoàn tác được về mặt nghiệp vụ, saga không áp dụng được, cần thiết
   kế khác (ví dụ approval boundary trước khi thực hiện bước đó).
5. Theo dõi context rot (mục 30.11) như một chỉ số observability chính thức, không chỉ phát hiện khi
   user báo cáo chất lượng giảm.

## 10. Tham số cần tuning

Số lần retry tối đa và hệ số backoff; ngưỡng lỗi liên tiếp để circuit breaker mở; thời gian chờ trước
khi circuit breaker thử half-open; giới hạn cụ thể của termination guard (số vòng lặp, token, chi phí,
thời gian, tool call, handoff) — cần điều chỉnh theo đặc thù từng loại task, không dùng một bộ giới
hạn chung cho mọi agent.

## 11. Failure modes

- **Không có termination guard**: agent lặp vòng lặp không hội tụ, tiêu tốn chi phí/thời gian không
  giới hạn — đây là failure mode nghiêm trọng nhất vì tác động trực tiếp tới chi phí vận hành.
- **Checkpoint thiếu chi tiết**: resume sai vị trí hoặc mất dữ liệu trung gian cần thiết dù cơ chế
  checkpoint "có chạy" — liên hệ trực tiếp failure mode đã nêu ở mục 11 của file 23 (State).
  reconstruct.
- **Retry không có giới hạn/backoff**: dội liên tục vào dịch vụ đang gặp sự cố, làm sự cố nghiêm
  trọng hơn thay vì phục hồi.
- **Saga thiếu compensating action cho một bước**: khi bước đó thất bại giữa chừng, hệ thống không
  thể đưa lại trạng thái nhất quán, để lại dữ liệu nửa vời khó xử lý thủ công.
- **Bỏ qua context rot**: agent chạy dài dần suy giảm chất lượng quyết định mà không có cảnh báo nào,
  chỉ phát hiện khi kết quả cuối đã sai.

## 12. Security considerations

Dead-letter task (mục 30.5) có thể chứa dữ liệu nhạy cảm của task thất bại — cần kiểm soát truy cập
tương đương dữ liệu production thông thường, không để hàng chờ debug trở thành điểm rò rỉ dữ liệu.
Fallback model (mục 30.6) cần đảm bảo model dự phòng có cùng mức kiểm soát bảo mật/guardrail (liên hệ
file 29) như model chính, không hạ chuẩn an toàn khi chuyển sang fallback.

## 13. Observability

Theo dõi riêng biệt: tỷ lệ checkpoint/resume thành công, số lần circuit breaker mở theo từng tool/
agent, tỷ lệ task rơi vào dead-letter, tần suất kích hoạt fallback model, và các chỉ số context rot
(độ dài context theo thời gian chạy, số lần compaction) như observability chính thức riêng cho tầng
reliability, tách biệt với observability logic nghiệp vụ của agent.

## 14. Evaluation metrics

Thời gian resume trung bình từ checkpoint; tỷ lệ task hoàn thành thành công sau retry so với thất bại
hẳn; tỷ lệ task cần dead-letter xử lý thủ công; chi phí/chất lượng chênh lệch khi dùng fallback model
so với model chính.

## 15. Ưu điểm

Cho phép agent xử lý task dài, phức tạp mà không mất toàn bộ tiến độ khi gặp lỗi cục bộ; giới hạn rõ
ràng ngân sách tài nguyên (thời gian, chi phí) đảm bảo hệ thống luôn kết thúc trong giới hạn dự đoán
được.

## 16. Nhược điểm

Thêm nhiều lớp hạ tầng (checkpoint store, circuit breaker, dead-letter queue) cần vận hành; cấu hình
sai các ngưỡng (retry, timeout, termination guard) có thể gây trải nghiệm kém (dừng quá sớm) hoặc lãng
phí tài nguyên (dừng quá muộn).

## 17. Khi nên dùng

Mọi agent chạy production, đặc biệt agent xử lý task nhiều bước hoặc tốn kém nếu phải làm lại từ đầu
— termination guard (mục 30.9) nên là yêu cầu tối thiểu bắt buộc cho bất kỳ agent có vòng lặp nào.

## 18. Khi không nên dùng

Agent xử lý một request đơn giản, một lượt gọi model duy nhất, không có vòng lặp hay tool call phức
tạp — phần lớn 11 pattern này (đặc biệt checkpoint, saga) là overhead không cần thiết cho use case
đơn giản như vậy.

## 19. Pattern liên quan

Checkpoint and resume phụ thuộc trực tiếp vào State (file 23); Context rot mitigation liên hệ chặt
với Context Engineering (file 24, đặc biệt mục 24.5); Fallback model và Graceful degradation liên hệ
Evaluation (Phần VIII) để xác định ngưỡng chất lượng kích hoạt; Saga liên hệ Mandate-based transaction
(mục 11.10) khi giao dịch cần compensating action.

## 20. Ví dụ kiến trúc thực tế

Agent xử lý quy trình đặt hàng nhiều bước (kiểm tra tồn kho, giữ chỗ, thanh toán, xác nhận vận
chuyển): mỗi bước checkpoint vào state store; nếu bước thanh toán timeout, retry with backoff trước
khi coi là thất bại; nếu dịch vụ thanh toán liên tục lỗi, circuit breaker mở và task chuyển dead-letter
để đội vận hành kiểm tra; toàn bộ quy trình được mô hình hoá dạng saga — nếu bước xác nhận vận chuyển
thất bại sau khi đã giữ chỗ và thanh toán thành công, compensating action tự động hoàn tiền và huỷ giữ
chỗ để đưa hệ thống về trạng thái nhất quán; termination guard giới hạn toàn bộ quy trình không vượt
quá một khoảng thời gian và chi phí xử lý tối đa.

---

*Nguồn tham khảo dùng khi biên soạn: tổng hợp và diễn giải lại các pattern reliability phổ biến trong
hệ thống phân tán (retry, circuit breaker, saga) áp dụng vào kiến trúc AI agent. Nội dung không trích
dẫn nguyên văn bất kỳ tài liệu cụ thể nào.*
