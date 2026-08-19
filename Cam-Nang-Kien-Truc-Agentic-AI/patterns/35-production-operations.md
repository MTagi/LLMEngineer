# 35. Production Operations

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Production Operations — ba nhóm pattern vận hành: Caching nhiều tầng (exact-match, semantic,
prompt/prefix), Cost governance (model routing, cost theo trace, budget/alert), Deployment và
versioning (version hoá đồng bộ, rollout dần, rollback plan).

## 2. Vấn đề cần giải quyết

Một hệ thống RAG/agent có thể đạt chất lượng tốt trong đánh giá (file 31-33) và có tracing đầy đủ
(file 34) nhưng vẫn thất bại ở khía cạnh vận hành thực tế: chi phí mỗi request quá cao để scale, độ
trễ không chấp nhận được khi lặp lại truy vấn tương tự, hoặc một bản deploy mới âm thầm làm giảm chất
lượng mà không ai phát hiện kịp thời vì thiếu quy trình rollout/rollback có kiểm soát. Đây là nhóm vấn
đề đặc thù của giai đoạn vận hành production quy mô lớn, khác với chất lượng logic (evaluation) hay
khả năng chẩn đoán sự cố (observability).

## 3. Bối cảnh sử dụng

Hệ thống RAG/agent đã qua giai đoạn phát triển, đang hoặc chuẩn bị phục vụ traffic thực ở quy mô cần
kiểm soát chi phí, độ trễ, và rủi ro khi thay đổi (đổi model, đổi prompt, đổi index).

## 4. Kiến trúc

```
Request
  ↓
1. Exact-match cache — cùng input y hệt → trả thẳng response đã lưu
  ↓ (miss)
2. Semantic cache — input khác chữ nhưng cùng ý nghĩa (so bằng embedding) → trả response gần nhất
  ↓ (miss)
3. Prompt/prefix cache — phần đầu prompt (system prompt, context tĩnh) không đổi → tái dùng phần
   tính toán đã cache ở phía provider, chỉ tính phần mới
  ↓ (miss hoàn toàn)
Gọi model đầy đủ
```

## 5. Thành phần

Exact-match cache store (key-value đơn giản); semantic cache (embedding index + ngưỡng similarity);
prompt/prefix caching (cơ chế phía model provider, tái dùng KV-cache); cost tracking gắn với trace
(liên hệ file 34); budget/alert system; version registry (prompt, model, index, tool schema); rollout
mechanism (canary/shadow traffic).

## 6. Luồng xử lý chi tiết

**35.1 Caching nhiều tầng.** Ba loại cache production hiện đại thường xếp chồng, mỗi loại giải quyết
một vấn đề khác nhau nên xử lý tuần tự: request trước tiên kiểm tra exact-match cache (input giống hệt
đã từng xử lý → trả thẳng response lưu sẵn, nhanh nhất nhưng chỉ bắt được trùng lặp tuyệt đối); nếu
miss, kiểm tra semantic cache (dùng embedding phát hiện câu hỏi khác chữ nhưng cùng ý nghĩa, bỏ qua
hẳn lệnh gọi model nếu đã có câu trả lời tương tự đủ tin cậy — cần threshold similarity đủ chặt để
tránh trả lời sai ngữ cảnh khi hai câu hỏi trông giống nhau nhưng ý nghĩa thực tế khác biệt); nếu vẫn
miss, prompt/prefix caching tái sử dụng phần tính toán (KV-cache) cho đoạn đầu prompt không đổi giữa
các lần gọi (system prompt, tool schema, context nền tảng dài và lặp lại) — hiệu quả nhất khi phần
đầu prompt chiếm tỷ trọng lớn và ổn định qua nhiều request. Hai loại cache đầu giải quyết vấn đề khác
nhau về bản chất (một giảm hẳn số lệnh gọi model, một giảm chi phí tính toán của lệnh gọi vẫn phải
thực hiện) nên thường dùng cùng lúc, không thay thế nhau.

**35.2 Cost governance.** Model routing định tuyến câu hỏi đơn giản tới model rẻ hơn và câu phức tạp
tới model mạnh hơn, tránh dùng model đắt nhất cho mọi request không phân biệt độ khó; cost cần theo
dõi ở mức từng trace (liên hệ file 34), không chỉ tổng hoá đơn cuối tháng — chỉ nhìn tổng hoá đơn
không cho biết chi phí đang tăng ở loại request nào, tenant nào, hay agent nào; đặt budget và alert ở
cấp tenant, agent, hoặc task type cụ thể để phát hiện sớm chi phí bất thường (ví dụ một agent bị lỗi
logic gây vòng lặp tốn token) trước khi nó tích luỹ thành vấn đề tài chính lớn.

**35.3 Deployment và versioning.** Bốn thành phần dễ lệch pha nhau nếu không quản lý tập trung: prompt,
model, index (đổi embedding model bắt buộc phải re-index toàn bộ corpus, không thể dùng lẫn embedding
cũ với embedding mới), và schema tool — cần version hoá đồng thời và nhất quán cả bốn. Khi đổi model
hoặc đổi prompt production, nên rollout dần (canary hoặc shadow traffic — cho một phần nhỏ traffic
thực đi qua bản mới trước khi mở rộng) và đo lại toàn bộ evaluation suite (Phần VIII) trên traffic đó
trước khi rollout 100%, không chỉ dựa vào kết quả test offline. Cần có rollback plan rõ ràng và đã
được kiểm thử sẵn khi một bản deploy mới làm giảm chất lượng theo eval hoặc observability — không chỉ
lý thuyết có thể rollback mà phải thực sự thực hiện được nhanh khi cần.

## 7. State và dữ liệu

Version metadata (prompt, model, index, tool schema) cần lưu tập trung, gắn trực tiếp với mỗi trace
(liên hệ file 34 mục 9) để mọi request có thể truy ngược đúng phiên bản đã xử lý nó; cache store (exact
-match, semantic) cần chính sách invalidation rõ ràng khi dữ liệu nguồn hoặc prompt/model thay đổi —
cache cũ dựa trên phiên bản đã lỗi thời có thể trả kết quả sai lệch với hệ thống hiện tại.

## 8. Thuật toán liên quan

Semantic caching dùng similarity search trên embedding (liên hệ Vector index, file 8); model routing
có thể dựa trên classifier đơn giản phân loại độ khó câu hỏi hoặc heuristic (độ dài, có cần reasoning
phức tạp không); canary/shadow rollout là kỹ thuật triển khai phần mềm chuẩn, không phải thuật toán
ML.

## 9. Cách triển khai

1. Triển khai exact-match cache trước (đơn giản nhất, hiệu quả ngay với traffic có phần trùng lặp
   thực sự), sau đó mới đầu tư semantic cache và prompt/prefix caching khi đã xác nhận nhu cầu.
2. Gắn cost tracking vào trace (file 34) ngay từ đầu, không đợi tới khi hoá đơn bất thường mới bắt
   đầu tìm cách theo dõi chi tiết.
3. Version hoá cả bốn thành phần (prompt, model, index, tool schema) từ bản deploy đầu tiên, dù hệ
   thống còn đơn giản — thêm versioning sau khi đã có nhiều bản deploy không version dễ gây nhầm lẫn
   hơn nhiều so với làm đúng từ đầu.
4. Luôn rollout canary/shadow trước khi rollout 100% cho bất kỳ thay đổi model/prompt nào ảnh hưởng
   traffic production, đo lại evaluation suite đầy đủ trên traffic canary trước khi mở rộng.
5. Kiểm thử rollback plan định kỳ (không chỉ viết ra rồi để đó) — đảm bảo khi thực sự cần rollback,
   quy trình đã được xác nhận hoạt động đúng.

## 10. Tham số cần tuning

Ngưỡng similarity cho semantic cache (chặt quá giảm hit rate, lỏng quá tăng rủi ro trả lời sai ngữ
cảnh); ngưỡng độ khó để model routing chuyển sang model mạnh hơn; tỷ lệ traffic canary trước khi mở
rộng toàn bộ; ngưỡng budget/alert theo từng cấp (tenant/agent/task type).

## 11. Failure modes

- **Semantic cache ngưỡng similarity quá lỏng**: trả lời câu hỏi khác ý nghĩa thực tế bằng response
  của một câu hỏi tưởng giống nhưng không giống, gây hallucination gián tiếp qua cache.
- **Bốn thành phần version lệch pha**: đổi embedding model nhưng quên re-index, hoặc đổi tool schema
  nhưng prompt production còn tham chiếu schema cũ — gây lỗi runtime khó truy vết.
- **Rollout 100% ngay không qua canary**: một thay đổi gây giảm chất lượng ảnh hưởng toàn bộ traffic
  ngay lập tức, phát hiện muộn vì không có traffic nhỏ để so sánh trước.
- **Rollback plan chưa từng kiểm thử**: khi thực sự cần rollback khẩn cấp, quy trình không hoạt động
  như kỳ vọng, kéo dài thời gian sự cố.
- **Cost governance chỉ nhìn tổng hoá đơn**: không phát hiện được agent/tenant cụ thể nào đang gây chi
  phí bất thường cho tới khi đã tích luỹ thành vấn đề lớn.

## 12. Security considerations

Semantic cache có thể vô tình trả kết quả của một user cho user khác nếu không phân vùng đúng theo
tenant/quyền truy cập (liên hệ Retrieval guardrail, mục 29.5) — cache key cần bao gồm đúng ngữ cảnh
quyền hạn, không chỉ nội dung câu hỏi; budget/alert theo tenant cũng là công cụ phát hiện sớm hành vi
bất thường có thể liên quan tới lạm dụng hoặc tấn công (một tenant đột ngột tăng vọt chi phí).

## 13. Observability

Cost tracking nên là một loại metadata/span riêng gắn với mỗi trace (liên hệ file 34), cho phép truy
vấn chi phí theo trace, theo tenant, theo agent; theo dõi hit rate của từng tầng cache riêng biệt
(exact-match, semantic, prefix) để đánh giá hiệu quả thực tế của từng loại; theo dõi chỉ số chất lượng
(từ file 31-33) tách riêng theo version trong giai đoạn canary để so sánh trực tiếp với bản hiện hành.

## 14. Evaluation metrics

Cache hit rate theo từng tầng; chi phí trung bình mỗi request (tổng và theo tenant/agent); độ trễ
trung bình có/không có cache; tỷ lệ rollout canary phát hiện được vấn đề trước khi mở rộng toàn bộ
(đo hiệu quả thực tế của quy trình rollout dần); thời gian trung bình thực hiện rollback khi cần.

## 15. Ưu điểm

Kiểm soát được chi phí và độ trễ ở quy mô production lớn; giảm đáng kể rủi ro khi thay đổi hệ thống
nhờ rollout dần và rollback có kiểm thử; version hoá đồng bộ tránh được lớp lỗi runtime khó truy vết
do các thành phần lệch pha nhau.

## 16. Nhược điểm

Thêm đáng kể độ phức tạp hạ tầng (nhiều tầng cache, hệ thống version registry, cơ chế canary) so với
triển khai đơn giản không có các lớp vận hành này; cần đầu tư liên tục để duy trì (invalidate cache
đúng lúc, cập nhật version registry mỗi lần đổi thành phần).

## 17. Khi nên dùng

Hệ thống RAG/agent phục vụ traffic production thực, đặc biệt khi quy mô đủ lớn để chi phí/độ trễ trở
thành mối quan tâm vận hành thực sự, hoặc khi tần suất thay đổi model/prompt đủ cao để rủi ro deploy
cần được kiểm soát có hệ thống.

## 18. Khi không nên dùng

Hệ thống thử nghiệm/demo quy mô nhỏ, traffic thấp, thay đổi hiếm — chi phí đầu tư hạ tầng caching/
versioning/rollout đầy đủ có thể vượt quá lợi ích thực tế mang lại ở quy mô đó; có thể bắt đầu đơn
giản (chỉ exact-match cache, versioning cơ bản) rồi mở rộng khi traffic tăng.

## 19. Pattern liên quan

Cost tracking và version metadata phụ thuộc trực tiếp vào Observability (file 34); rollout dần cần đo
lại Evaluation suite (file 31-33) trước khi mở rộng; semantic cache liên hệ Vector index (file 8) cho
kỹ thuật similarity search; toàn bộ nhóm pattern này là lớp vận hành bao trùm lên mọi kiến trúc đã mô
tả ở các phần trước.

## 20. Ví dụ kiến trúc thực tế

Hệ thống RAG/agent phục vụ hàng nghìn user nội bộ mỗi ngày: exact-match cache bắt các câu hỏi FAQ lặp
lại chính xác; semantic cache (phân vùng theo tenant để tránh rò rỉ chéo) bắt các câu hỏi diễn đạt
khác nhau nhưng cùng ý; prompt/prefix caching tái dùng phần system prompt và tool schema dài không đổi
giữa các request; cost được theo dõi theo từng phòng ban (tenant), có alert khi một phòng ban vượt
ngân sách tháng bất thường; khi đội kỹ thuật đổi sang embedding model mới, toàn bộ index được re-index
đồng bộ, bản deploy mới chạy canary trên 5% traffic trong 3 ngày, đo lại đầy đủ bộ eval Phần VIII trước
khi mở rộng toàn bộ — có rollback plan đã kiểm thử sẵn sàng nếu canary cho thấy chất lượng giảm.

---

*Nguồn tham khảo dùng khi biên soạn: tổng hợp và diễn giải lại các thực hành vận hành production phổ
biến trong ngành cho hệ thống LLM/RAG/agent (caching nhiều tầng, cost governance, canary rollout). Nội
dung không trích dẫn nguyên văn bất kỳ tài liệu cụ thể nào.*
