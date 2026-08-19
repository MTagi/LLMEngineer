# 33. Multi-agent evaluation

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Multi-agent evaluation — chỉ số bổ sung riêng cho hệ thống nhiều agent (Routing accuracy, Handoff
accuracy, Delegation success, Coordination overhead, Duplicate work, Conflict resolution, Agent
contribution, Termination rate, End-to-end trace completeness), cùng cách tiếp cận đánh giá 3 tầng.

## 2. Vấn đề cần giải quyết

Agent evaluation (file 32) đánh giá tốt hành vi của **một** agent đơn lẻ, nhưng hệ thống multi-agent
có thêm một lớp lỗi hoàn toàn khác: lỗi ở khâu **phối hợp** giữa các agent — routing sai domain,
handoff mất ngữ cảnh, hai agent vô tình làm trùng việc, xung đột quyết định không được giải quyết
đúng cách. Chỉ đánh giá từng agent con riêng lẻ, dù mỗi agent đều "đúng" theo tiêu chuẩn của Agent
evaluation, hệ thống tổng thể vẫn có thể thất bại vì lỗi nằm ở khâu phối hợp — một lớp lỗi hoàn toàn
không xuất hiện khi chỉ nhìn từng agent độc lập.

## 3. Bối cảnh sử dụng

Mọi hệ thống có từ hai agent trở lên phối hợp với nhau (bất kể theo pattern nào ở Phần IV — Federated,
Supervisor, Orchestrator-Worker, Event-driven, Hybrid...) — càng nhiều agent và càng nhiều điểm phối
hợp, càng cần đánh giá riêng lớp này.

## 4. Kiến trúc

```
Tầng 1: Đánh giá độc lập từng agent con
        (agent đó làm đúng phần việc của mình không, tách biệt khỏi phần còn lại hệ thống)
              ↓
Tầng 2: Đánh giá chất lượng phối hợp
        (handoff, delegation, tổng hợp kết quả giữa các agent có đúng không)
              ↓
Tầng 3: Đánh giá kết quả toàn hệ thống
        (task cuối cùng có thành công không)
```

## 5. Thành phần

Trace liên kết xuyên suốt nhiều agent (correlation ID, liên hệ file 19); bộ chỉ số riêng cho khâu
phối hợp; Agent evaluation (file 32) áp dụng cho từng agent con ở tầng 1; end-to-end evaluation
(tương tự file 31 mục End-to-end) ở tầng 3.

## 6. Luồng xử lý chi tiết

**Chín chỉ số bổ sung riêng cho multi-agent:** *Routing accuracy* (Supervisor/router có chọn đúng
agent/domain phù hợp cho một yêu cầu không — liên hệ Federated, file 12); *Handoff accuracy* (khi
chuyển giao hội thoại/task từ agent này sang agent khác, ngữ cảnh có được truyền đầy đủ và chính xác
không, user có phải lặp lại thông tin đã cung cấp không); *Delegation success* (khi một agent giao
việc cho agent/sub-agent khác, phần việc được giao có hoàn thành đúng yêu cầu không); *Coordination
overhead* (chi phí thời gian/token phát sinh riêng cho việc phối hợp — giao tiếp giữa các agent,
tổng hợp kết quả — tách biệt khỏi chi phí xử lý thực chất của từng agent); *Duplicate work* (có agent
nào vô tình làm trùng phần việc agent khác đã làm không, dấu hiệu thiếu phối hợp rõ ràng); *Conflict
resolution* (khi hai agent đưa ra quyết định/kết quả mâu thuẫn nhau, cơ chế giải quyết xung đột có
hoạt động đúng không — liên hệ Committee-of-experts, file 16); *Agent contribution* (đóng góp thực
tế của từng agent vào kết quả cuối, hữu ích để phát hiện agent nào trong hệ thống không mang lại giá
trị tương xứng với chi phí vận hành); *Termination rate* (tỷ lệ luồng multi-agent kết thúc đúng cách
so với bị treo/lặp vô hạn — liên hệ Termination guard, mục 30.9, nhưng ở góc độ toàn hệ thống thay vì
một agent đơn lẻ); *End-to-end trace completeness* (trace có ghi lại đầy đủ, liên tục xuyên suốt mọi
agent tham gia xử lý một request không, hay bị đứt đoạn ở một số điểm chuyển giao).

**Đánh giá 3 tầng** là cách tiếp cận có cấu trúc thay vì chỉ chấm một điểm số cuối duy nhất: **Tầng
1** đánh giá độc lập từng agent con bằng chính bộ tiêu chí Agent evaluation (file 32), tách biệt hoàn
toàn khỏi phần còn lại của hệ thống — trả lời câu hỏi "agent này tự nó có làm đúng việc của nó
không". **Tầng 2** đánh giá riêng chất lượng phối hợp bằng chín chỉ số vừa liệt kê ở trên — trả lời
câu hỏi "các agent có làm việc cùng nhau đúng cách không", độc lập với việc từng agent con có giỏi
hay không. **Tầng 3** đánh giá kết quả toàn hệ thống (tương tự tầng End-to-end của RAG evaluation,
file 31) — trả lời câu hỏi "cuối cùng task có thành công không" từ góc nhìn user. Cách tách ba tầng
này giúp định vị chính xác lỗi nằm ở agent con cụ thể nào, ở khâu phối hợp, hay ở outcome cuối — thay
vì chỉ biết "hệ thống sai" mà không biết sai ở đâu để sửa đúng chỗ.

## 7. State và dữ liệu

Cần trace xuyên suốt toàn bộ luồng qua nhiều agent (correlation ID nhất quán, liên hệ file 19 mục 7)
để có thể tính được các chỉ số tầng 2 (Handoff accuracy, Duplicate work) vốn đòi hỏi nhìn thấy tương
tác giữa các agent, không chỉ log riêng lẻ của từng agent.

## 8. Thuật toán liên quan

Không có thuật toán ML riêng cho tầng phối hợp — chủ yếu là phân tích trace/log có cấu trúc; tầng 1
và tầng 3 tái sử dụng kỹ thuật của Agent evaluation (LLM-as-judge, Agent-as-a-Judge) và RAG evaluation
tương ứng.

## 9. Cách triển khai

1. Đầu tư trace xuyên suốt (correlation ID nhất quán qua mọi agent) trước khi có thể đo bất kỳ chỉ số
   tầng 2 nào — thiếu trace liên kết, không thể phát hiện Duplicate work hay đo chính xác Coordination
   overhead.
2. Đánh giá theo đúng thứ tự ba tầng khi debug một sự cố: kiểm tra tầng 1 trước (từng agent con có ổn
   không), rồi tầng 2 (phối hợp có ổn không), cuối cùng mới kết luận ở tầng 3 — tránh kết luận vội
   "hệ thống lỗi" mà không biết lỗi nằm ở đâu.
3. Theo dõi Agent contribution định kỳ để phát hiện agent nào trong hệ thống đang tốn chi phí vận
   hành nhưng đóng góp thấp — cân nhắc loại bỏ hoặc gộp lại.
4. Đặt ngưỡng cảnh báo riêng cho Termination rate ở cấp hệ thống — một agent con có termination guard
   tốt (file 30) không đảm bảo toàn bộ luồng multi-agent không bị treo ở một điểm phối hợp nào đó.

## 10. Tham số cần tuning

Ngưỡng chấp nhận cho Coordination overhead (bao nhiêu phần trăm chi phí tổng dành cho phối hợp là hợp
lý); ngưỡng Handoff accuracy tối thiểu trước khi coi là vấn đề cần khắc phục; tần suất lấy mẫu trace
để tính Duplicate work (tính trên toàn bộ traffic có thể tốn kém, cần lấy mẫu hợp lý).

## 11. Failure modes

- **Chỉ đánh giá tầng 1 (từng agent con)**: bỏ lỡ hoàn toàn lỗi phối hợp — hệ thống thất bại dù mọi
  agent con đều "đạt chuẩn" riêng lẻ.
- **Trace không liên kết xuyên suốt**: không thể tính được các chỉ số tầng 2, không thể debug khi sự
  cố nằm ở khâu chuyển giao giữa các agent.
- **Không theo dõi Duplicate work**: lãng phí chi phí vận hành âm thầm qua thời gian mà không ai phát
  hiện, vì mỗi agent riêng lẻ vẫn "hoạt động đúng" theo nghĩa hẹp.
- **Bỏ qua Termination rate ở cấp hệ thống**: luồng multi-agent bị treo ở một điểm phối hợp (ví dụ chờ
  handoff không bao giờ tới) mà termination guard của từng agent con riêng lẻ không phát hiện được.

## 12. Security considerations

Handoff accuracy và trace completeness liên hệ trực tiếp tới Identity propagation (mục 29.1) — cần
xác nhận identity/quyền hạn được truyền đúng qua mỗi lần handoff, không chỉ ngữ cảnh nghiệp vụ; Agent
contribution thấp bất thường của một agent có thể là dấu hiệu agent đó đã bị xâm phạm hoặc hành vi
lệch khỏi thiết kế ban đầu, đáng để điều tra thêm.

## 13. Observability

Dashboard riêng cho tầng 2 (phối hợp) tách biệt khỏi dashboard từng agent con — giúp đội vận hành
nhìn thấy sức khoẻ của "chất keo" giữa các agent, không chỉ sức khoẻ của từng thành phần; liên kết với
End-to-end trace completeness (file 34) để đảm bảo dữ liệu đánh giá tầng 2 luôn đầy đủ.

## 14. Evaluation metrics

(Bản thân file này định nghĩa các chỉ số — xem mục 6.) Chỉ số meta hữu ích: tỷ lệ sự cố được định vị
chính xác vào đúng tầng (1, 2, hay 3) khi debug — đo hiệu quả thực tế của cách tiếp cận đánh giá 3
tầng trong việc rút ngắn thời gian xác định nguyên nhân gốc.

## 15. Ưu điểm

Định vị chính xác nguồn gốc lỗi trong hệ thống nhiều agent — agent con cụ thể, khâu phối hợp, hay
outcome cuối — giúp cải thiện đúng chỗ thay vì tối ưu mù quáng toàn hệ thống; phát hiện được lớp lỗi
(phối hợp) hoàn toàn không xuất hiện nếu chỉ đánh giá từng agent riêng lẻ.

## 16. Nhược điểm

Đòi hỏi hạ tầng trace xuyên suốt phức tạp hơn đáng kể so với đánh giá một agent đơn lẻ; chín chỉ số bổ
sung cộng thêm ba tầng đánh giá làm tăng đáng kể khối lượng công việc xây dựng và duy trì hệ thống
evaluation.

## 17. Khi nên dùng

Mọi hệ thống có từ hai agent trở lên phối hợp với nhau — không có ngoại lệ, vì lớp lỗi phối hợp là
đặc thù riêng của multi-agent, không thể phát hiện bằng cách chỉ áp dụng Agent evaluation (file 32)
cho từng agent.

## 18. Khi không nên dùng

Hệ thống chỉ có một agent duy nhất, không có phối hợp giữa nhiều agent — Agent evaluation (file 32)
đơn thuần là đủ, không cần các chỉ số và tầng đánh giá bổ sung này.

## 19. Pattern liên quan

Mở rộng trực tiếp từ Agent evaluation (file 32, dùng cho tầng 1); tương ứng tầng End-to-end của RAG
evaluation (file 31) ở tầng 3; đo trực tiếp chất lượng của toàn bộ Phần IV (các pattern multi-agent,
file 12-21), đặc biệt Handoff và Hybrid orchestration (file 21).

## 20. Ví dụ kiến trúc thực tế

Hệ thống hỗ trợ khách hàng đa domain (kỹ thuật, tài chính, tài khoản) dùng Supervisor điều phối: tầng
1 đánh giá riêng agent kỹ thuật, agent tài chính, agent tài khoản bằng bộ tiêu chí Agent evaluation;
tầng 2 đo Routing accuracy của Supervisor (có chuyển đúng domain không) và Handoff accuracy khi
chuyển từ agent kỹ thuật sang chuyên viên người thật; tầng 3 đo Task success và User satisfaction
tổng thể — khi User satisfaction giảm, đội vận hành tra ngược qua tầng 2 trước (phát hiện Handoff
accuracy giảm do mất ngữ cảnh khi chuyển giao) thay vì phải đoán mò cả hệ thống, rồi mới xác định
nguyên nhân gốc và sửa đúng khâu chuyển giao.

---

*Nguồn tham khảo dùng khi biên soạn: tổng hợp và diễn giải lại các hướng tiếp cận đánh giá hệ thống
multi-agent phổ biến trong ngành 2025-2026. Nội dung không trích dẫn nguyên văn bất kỳ tài liệu cụ
thể nào.*
