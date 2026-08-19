# 29. Security patterns

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Security patterns cho agent — 11 pattern: Identity propagation, Least-privilege agent,
Capability-based authorization, Input guardrail, Retrieval guardrail, Tool guardrail, Output
guardrail, Approval boundary, Agent admission control, Memory và context poisoning, Excessive
agency (ba nguyên nhân gốc).

## 2. Vấn đề cần giải quyết

Agent khác biệt cơ bản với ứng dụng truyền thống ở chỗ nó tự quyết định hành động (gọi tool nào,
truy vấn nguồn nào, thực hiện thay đổi gì) dựa trên suy luận của model — bề mặt tấn công không còn
chỉ là input người dùng nhập trực tiếp, mà mở rộng ra mọi nội dung agent "đọc" trong quá trình xử lý
(tài liệu truy xuất, kết quả tool call, nội dung memory cũ). Không có một bộ pattern bảo mật thiết
kế riêng cho đặc thù này, agent kế thừa nguyên vẹn rủi ro bảo mật ứng dụng truyền thống nhưng lại có
thêm bề mặt tấn công mới (prompt injection, tool instruction injection, memory poisoning) mà kiểm
soát truyền thống không bao phủ.

## 3. Bối cảnh sử dụng

Mọi agent có khả năng gọi tool, truy cập dữ liệu, hoặc thực hiện hành động thay mặt user/tổ chức —
mức độ nghiêm ngặt cần áp dụng tăng theo mức độ nhạy cảm của dữ liệu/hành động agent có quyền chạm
tới.

## 4. Kiến trúc

```
User
  ↓
Gateway
  ↓
Supervisor
  ↓
Remote Agent
  ↓
MCP Tool
  ↓
Data Source
```

Identity phải truyền xuyên suốt toàn bộ chuỗi này; guardrail đặt tại bốn điểm: input, retrieval,
tool, output.

## 5. Thành phần

Identity provider/propagation mechanism; capability registry (gắn quyền với capability cụ thể);
guardrail engine (input/retrieval/tool/output, mục 29.4-29.7); approval workflow (mục 29.8); agent
admission pipeline (mục 29.9); memory provenance tracking (mục 29.10).

## 6. Luồng xử lý chi tiết

**11 pattern:**

- **Identity propagation (29.1)**: user identity phải được truyền nguyên vẹn qua toàn bộ chuỗi
  Gateway → Supervisor → Remote Agent → MCP Tool → Data Source — không được để agent dùng service
  account có quyền quá rộng thay cho user ở bất kỳ điểm nào trong chuỗi, vì điều đó phá vỡ khả năng
  kiểm soát truy cập theo đúng quyền của từng user cụ thể.
- **Least-privilege agent (29.2)**: mỗi agent chỉ có access scope cần thiết cho đúng phạm vi công
  việc được giao — không cấp quyền rộng "phòng khi cần" vì agent tự quyết định hành động dựa trên
  suy luận, quyền dư thừa là bề mặt rủi ro thực tế chứ không chỉ lý thuyết.
- **Capability-based authorization (29.3)**: quyền thực hiện gắn với capability cụ thể (một hành
  động cụ thể agent được phép làm), không chỉ gắn với endpoint kỹ thuật — cho phép kiểm soát mịn hơn
  khi một endpoint có thể phục vụ nhiều loại hành động khác nhau về mức độ nhạy cảm.
- **Input guardrail (29.4)**: kiểm tra đầu vào trước khi agent xử lý — prompt injection, PII,
  malware, tool instruction injection (chỉ thị ẩn giấu trong nội dung nhằm điều khiển agent), invalid
  schema, unsupported file type.
- **Retrieval guardrail (29.5)**: kiểm tra tài liệu/dữ liệu được truy xuất trước khi đưa vào context
  — ACL, tenant isolation, data residency, poisoned document, hidden instruction, source trust,
  version validity.
- **Tool guardrail (29.6)**: kiểm tra trước khi thực thi tool call — tool allowlist, parameter
  constraint, write permission, rate limit, transaction amount, target environment.
- **Output guardrail (29.7)**: kiểm tra đầu ra trước khi trả cho user — data leakage, unsupported
  claims, citation, policy violation, secret, PII.
- **Approval boundary (29.8)**: các hành động không thể đảo ngược (giao dịch giá trị cao, xoá dữ
  liệu, gửi thông tin ra ngoài tổ chức) cần điểm dừng chờ con người phê duyệt trước khi thực thi,
  không tự động hoá hoàn toàn dù agent "tự tin" vào quyết định của mình.
- **Agent admission control (29.9)**: agent mới muốn tham gia hệ thống (đặc biệt trong kiến trúc
  multi-agent có registry) phải qua capability validation, schema validation, security scanning,
  evaluation suite, và registration approval — dynamic agent registry không chỉ lưu endpoint, mà
  cần validate đầy đủ trước khi agent được phép tham gia.
- **Memory và context poisoning (29.10)**: khác retrieval guardrail (xử lý tài liệu bị đầu độc nạp
  vào khi truy xuất), đây là rủi ro nội dung độc hại bị ghi vào chính memory dài hạn của agent
  (episodic, semantic, entity memory — mục 22.4-22.7) qua một tương tác trước đó, ảnh hưởng tới
  quyết định ở các phiên sau dù nguồn gây hại ban đầu không còn trong context hiện tại — cần validate
  nội dung trước khi ghi vào memory dài hạn, gắn provenance cho mỗi memory record, có cơ chế memory
  forgetting áp dụng được cho nội dung bị nghi ngờ độc hại.
- **Excessive agency — ba nguyên nhân gốc (29.11)**: tách "quyền tự chủ quá mức" thành ba nguyên
  nhân riêng biệt để audit có hệ thống — *Excessive functionality* (agent có thể gọi tool/hành động
  vượt phạm vi task, liên hệ 29.3), *Excessive permissions* (tool agent gọi có quyền rộng hơn cần
  thiết, liên hệ 29.2), *Excessive autonomy* (hành động tác động lớn thực thi không qua điểm dừng
  con người, liên hệ 29.8 và Mandate-based transaction mục 11.10).

## 7. State và dữ liệu

Guardrail decision (pass/block/flag) nên được log kèm lý do cụ thể cho mỗi lượt kiểm tra; memory
record cần trường provenance (mục 29.10) ghi rõ nguồn gốc để hỗ trợ audit và memory forgetting có
chọn lọc.

## 8. Thuật toán liên quan

Prompt injection/PII detection thường dùng classifier chuyên biệt hoặc LLM-based guardrail; capability
-based authorization dựa trên mô hình kiểm soát truy cập (access control model) truyền thống áp dụng
vào ngữ cảnh agent; không có thuật toán ML riêng cho approval boundary hay admission control (chủ
yếu là logic nghiệp vụ/quy trình).

## 9. Cách triển khai

1. Vẽ rõ toàn bộ chuỗi identity propagation (mục 4) cho hệ thống cụ thể, xác nhận không có điểm nào
   "rơi" xuống service account quyền rộng.
2. Triển khai guardrail tại cả bốn điểm (input, retrieval, tool, output) — thiếu một điểm để lỗ hổng
   cho loại tấn công tương ứng đi qua mà không bị chặn.
3. Định nghĩa rõ ràng danh sách hành động cần approval boundary (mục 29.8) dựa trên mức độ không thể
   đảo ngược, không chỉ dựa trên giá trị tiền tệ.
4. Với hệ thống multi-agent có khả năng thêm agent động, bắt buộc admission pipeline (mục 29.9)
   trước khi agent mới được cấp quyền tham gia.
5. Audit định kỳ theo khung ba nguyên nhân gốc (mục 29.11) thay vì chỉ kiểm tra "agent có quá nhiều
   quyền không" một cách chung chung — mỗi nguyên nhân cần biện pháp khắc phục khác nhau.

## 10. Tham số cần tuning

Ngưỡng nhạy của guardrail (chặt quá gây false positive làm gián đoạn công việc hợp lệ, lỏng quá bỏ
lọt tấn công thực); ngưỡng giá trị/tác động kích hoạt approval boundary; tần suất chạy lại evaluation
suite cho agent đã qua admission.

## 11. Failure modes

- **Identity "rơi rớt" giữa chừng chuỗi**: một điểm trong chuỗi Gateway→...→Data Source dùng service
  account chung thay vì truyền đúng identity user, phá vỡ khả năng kiểm soát truy cập chính xác.
- **Guardrail chỉ đặt ở input**: bỏ qua retrieval/tool/output guardrail, để lọt tấn công không tới
  qua input trực tiếp mà qua tài liệu truy xuất hoặc kết quả tool call.
- **Memory poisoning không bị phát hiện**: nội dung độc hại ghi vào memory dài hạn từ lâu, ảnh hưởng
  âm thầm tới quyết định agent ở các phiên sau, khó truy vết vì nguồn gây hại ban đầu đã không còn
  trong context.
- **Agent registry không validate đầy đủ**: chỉ lưu endpoint mà không qua admission pipeline, cho
  phép agent chưa được kiểm định tham gia hệ thống.

## 12. Security considerations

(Chính bản thân file này là nội dung security — xem mục 6 cho đầy đủ 11 pattern.) Điểm nhấn quan
trọng nhất cần nhắc lại: guardrail cần bao phủ toàn bộ bốn điểm (input/retrieval/tool/output), không
chỉ input như trực giác ban đầu thường nghĩ tới.

## 13. Observability

Log mọi quyết định guardrail (pass/block/flag) kèm lý do; dashboard riêng theo dõi tỷ lệ block theo
từng loại guardrail và từng agent để phát hiện agent nào đang gặp nhiều cảnh báo bất thường; audit
trail đầy đủ cho mọi hành động qua approval boundary.

## 14. Evaluation metrics

Tỷ lệ tấn công injection bị chặn đúng (true positive) so với tỷ lệ chặn nhầm nội dung hợp lệ (false
positive); thời gian trung bình xử lý qua approval boundary; tỷ lệ agent admission bị từ chối và lý
do; tỷ lệ memory record có provenance đầy đủ.

## 15. Ưu điểm

Bao phủ đúng đặc thù bề mặt tấn công của agent (không chỉ input người dùng mà cả nội dung agent tự
"đọc" trong quá trình xử lý), giảm đáng kể rủi ro so với chỉ áp dụng kiểm soát bảo mật ứng dụng
truyền thống.

## 16. Nhược điểm

Thêm nhiều lớp kiểm tra có thể tăng độ trễ end-to-end; guardrail quá nghiêm ngặt có thể cản trở công
việc hợp lệ của agent, cần cân bằng liên tục giữa an toàn và hiệu quả vận hành.

## 17. Khi nên dùng

Mọi agent production có khả năng truy cập dữ liệu nhạy cảm hoặc thực hiện hành động có tác động thực
— không có ngoại lệ "agent nhỏ nên bỏ qua security", vì bề mặt tấn công (prompt injection, tool
instruction injection) không phụ thuộc vào quy mô agent.

## 18. Khi không nên dùng

Agent hoàn toàn read-only, không truy cập dữ liệu nhạy cảm, chạy trong môi trường sandbox cô lập
hoàn toàn (ví dụ demo nội bộ không kết nối hệ thống thật) — vẫn nên có input guardrail cơ bản, nhưng
có thể giảm nhẹ tool/output guardrail nếu không có tool thực sự nhạy cảm.

## 19. Pattern liên quan

Bao phủ toàn bộ kiến trúc mô tả ở các file trước — Tool use patterns (file 11, liên hệ tool
guardrail), Memory patterns (file 22, liên hệ memory poisoning), MCP/A2A (file 25-26, liên hệ identity
propagation qua các chuẩn interoperability), Mandate-based transaction (mục 11.10) và Agent Payments
Protocol (file 28, liên hệ approval boundary cho giao dịch tài chính).

## 20. Ví dụ kiến trúc thực tế

Agent xử lý yêu cầu hỗ trợ khách hàng có quyền truy vấn CRM và thực hiện hoàn tiền: identity user hỗ
trợ viên được truyền xuyên suốt từ gateway tới MCP tool gọi CRM (không dùng service account chung);
input guardrail chặn prompt injection trong nội dung ticket khách hàng gửi; retrieval guardrail đảm
bảo agent chỉ thấy dữ liệu khách hàng đúng tenant; tool guardrail giới hạn số tiền hoàn tối đa agent
tự thực hiện được; giao dịch hoàn tiền vượt ngưỡng kích hoạt approval boundary chờ quản lý duyệt;
output guardrail rà soát phản hồi cuối trước khi gửi khách hàng để tránh rò rỉ thông tin nội bộ.

---

*Nguồn tham khảo dùng khi biên soạn: tổng hợp và diễn giải lại các khuyến nghị an ninh cho ứng dụng
agentic phổ biến trong ngành 2025-2026 (bao gồm hướng tiếp cận của khung OWASP Top 10 cho ứng dụng
Agentic). Nội dung không trích dẫn nguyên văn bất kỳ tài liệu cụ thể nào.*
