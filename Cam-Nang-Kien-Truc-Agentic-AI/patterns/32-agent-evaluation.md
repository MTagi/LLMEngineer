# 32. Agent evaluation

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Agent evaluation — đánh giá cả trajectory (không chỉ final answer): chọn tool, tham số tool, gọi
thừa tool, dừng đúng lúc, tính hợp lệ của plan, khả năng phục hồi sau lỗi, tuân thủ policy; cùng
cách tiếp cận Agent-as-a-Judge.

## 2. Vấn đề cần giải quyết

Đánh giá agent chỉ dựa trên câu trả lời cuối cùng bỏ lỡ toàn bộ thông tin về **cách** agent đi tới
kết quả đó — hai agent có thể cùng cho ra câu trả lời đúng nhưng một agent đi đường vòng tốn kém
(gọi thừa tool, plan không tối ưu), hoặc tệ hơn, "đúng nhờ may mắn" (một lỗi ở giữa được một lỗi khác
bù lại một cách tình cờ). Không quan sát trajectory, không thể phát hiện các vấn đề này cho tới khi
chúng gây hậu quả thực sự (chi phí cao, hành vi không nhất quán trên input tương tự).

## 3. Bối cảnh sử dụng

Mọi agent có nhiều hơn một bước ra quyết định (chọn tool, lập plan, thực hiện nhiều lượt suy luận) —
đặc biệt quan trọng khi agent có khả năng gây tác động thực (tool call thay đổi dữ liệu) nơi hành vi
sai giữa chừng có thể gây hậu quả dù kết quả cuối trông có vẻ ổn.

## 4. Kiến trúc

```
Task đầu vào
    ↓
[Trajectory]  Bước 1 → Bước 2 → ... → Bước N   (mỗi bước: chọn tool, tham số, kết quả)
    ↓
Final Answer

Đánh giá SONG SONG hai lớp:
  (1) Rubric chấm từng bước   — tool đúng/sai, tham số đúng/sai, có gọi thừa không
  (2) Judge chấm toàn trajectory — chiến lược tổng thể có hợp lý không, có phục hồi tốt sau lỗi không
```

## 5. Thành phần

Trajectory logger (ghi lại đầy đủ từng bước: tool được gọi, tham số, kết quả, quyết định tiếp theo);
rubric chấm từng bước (rule-based hoặc LLM-based); Agent-as-a-Judge (một hệ thống agentic riêng dùng
để đánh giá, có khả năng multi-step reasoning); benchmark trajectory (nếu dùng bộ có sẵn).

## 6. Luồng xử lý chi tiết

Đánh giá agent không dừng ở final answer mà mở rộng ra bảy khía cạnh của trajectory: *agent có chọn
đúng tool không* (trong số các tool khả dụng, có chọn tool phù hợp nhất cho bước đó không); *tool
parameter có đúng không* (tham số truyền vào tool có chính xác, đầy đủ không); *có gọi thừa tool
không* (thực hiện các bước không cần thiết, tốn chi phí/thời gian không mang lại giá trị); *có dừng
đúng lúc không* (biết khi nào đã đủ thông tin để kết luận, không tiếp tục lặp vô ích, cũng không dừng
quá sớm khi chưa đủ căn cứ); *plan có hợp lệ không* (kế hoạch tổng thể agent đề ra có khả thi và hợp
lý không, liên hệ Reasoning/Planning patterns file 10); *có phục hồi sau lỗi không* (khi một bước gặp
lỗi, agent có điều chỉnh chiến lược hợp lý thay vì lặp lại y hệt hành động đã thất bại hoặc bỏ cuộc
không cần thiết); *có tuân thủ policy không* (các ràng buộc nghiệp vụ/an ninh đặt ra cho agent có
được tôn trọng xuyên suốt trajectory không, liên hệ file 29).

**Agent-as-a-Judge** khác với LLM-as-judge truyền thống (chỉ chấm output cuối bằng một lượt gọi
model) ở chỗ dùng chính một hệ thống agentic để đánh giá — có khả năng multi-step reasoning và quan
sát toàn bộ quá trình, không chỉ kết quả, nhờ đó đánh giá được cả cách agent đi tới câu trả lời chứ
không chỉ câu trả lời cuối. Một số bộ benchmark public theo hướng đánh giá trajectory này gồm
AgentRewardBench (chấm khả năng đánh giá plan/thực thi), TRACE (đánh giá dựa trên evidence bank), và
TRAJECT-Bench (đánh giá chi tiết việc dùng tool). Khi tự xây eval nội bộ, nên tách rõ hai lớp: (1)
rubric chấm từng bước (tool đúng/sai, tham số đúng/sai — có thể rule-based cho phần xác định rõ ràng),
và (2) judge chấm toàn trajectory để bắt các lỗi chỉ lộ ra khi nhìn tổng thể (ví dụ mỗi bước riêng lẻ
đều hợp lý nhưng chiến lược tổng thể sai — lỗi loại này không thể phát hiện chỉ bằng rubric từng
bước).

## 7. State và dữ liệu

Trajectory cần được log đầy đủ và có cấu trúc (không chỉ text tự do) — mỗi bước ghi rõ tool được gọi,
tham số, kết quả trả về, và lý do agent quyết định bước tiếp theo (nếu có thể trích xuất) — dữ liệu
này vừa phục vụ evaluation vừa phục vụ debug khi có sự cố.

## 8. Thuật toán liên quan

LLM-as-judge/Agent-as-a-Judge (dùng model để chấm điểm định tính); rubric-based scoring (quy tắc rõ
ràng cho phần có thể xác định đúng/sai một cách khách quan, ví dụ tham số tool có đúng kiểu dữ liệu
không).

## 9. Cách triển khai

1. Đầu tư vào trajectory logging đầy đủ trước khi xây bất kỳ hệ thống evaluation nào — không có
   trajectory chi tiết, không thể đánh giá được bảy khía cạnh đã liệt kê ở mục 6.
2. Tách rõ hai lớp đánh giá: rubric cho phần xác định được rõ ràng (tool đúng/sai, tham số hợp lệ),
   Agent-as-a-Judge cho phần cần đánh giá tổng thể/ngữ nghĩa (chiến lược có hợp lý không).
3. Validate Agent-as-a-Judge định kỳ bằng đánh giá con người trên một tập mẫu, tương tự nguyên tắc
   validate LLM-as-judge trong RAG evaluation (file 31).
4. Cân nhắc dùng benchmark public sẵn có (AgentRewardBench, TRACE, TRAJECT-Bench) làm điểm tham chiếu
   trước khi đầu tư xây benchmark nội bộ riêng từ đầu.
5. Theo dõi riêng chỉ số "gọi thừa tool" và "không dừng đúng lúc" như tín hiệu chi phí, không chỉ tín
   hiệu chất lượng — hai vấn đề này ảnh hưởng trực tiếp tới chi phí vận hành dù không luôn làm sai kết
   quả cuối.

## 10. Tham số cần tuning

Ngưỡng chấp nhận cho từng khía cạnh trajectory (ví dụ số lần gọi thừa tool tối đa chấp nhận được);
trọng số tương đối giữa rubric từng bước và judge toàn trajectory khi tổng hợp một điểm số chung (nếu
cần một chỉ số tổng hợp).

## 11. Failure modes

- **Chỉ đánh giá final answer**: bỏ lỡ hoàn toàn thông tin về cách agent đi tới kết quả, không phát
  hiện được agent "đúng nhờ may mắn" hoặc đi đường vòng tốn kém.
- **Rubric quá cứng nhắc**: chấm sai khi agent tìm ra một cách giải quyết hợp lý nhưng khác với đường
  đi "chuẩn" được định nghĩa trước trong rubric.
- **Agent-as-a-Judge không được validate**: judge tự nó có lỗi hệ thống trong đánh giá mà không ai
  phát hiện, dẫn tới tối ưu sai hướng.
- **Trajectory logging thiếu chi tiết**: không đủ dữ liệu để đánh giá đầy đủ bảy khía cạnh, hoặc
  không đủ để debug khi cần điều tra một trường hợp cụ thể.

## 12. Security considerations

Đánh giá "có tuân thủ policy không" trong trajectory nên bao phủ cả các ràng buộc an ninh (liên hệ
file 29) — ví dụ agent có gọi tool ngoài phạm vi capability được cấp không, có bỏ qua approval
boundary khi lẽ ra cần dừng lại không — không chỉ đánh giá tuân thủ policy nghiệp vụ thông thường.

## 13. Observability

Trajectory evaluation nên liên kết trực tiếp với Agent Span trong tracing (liên hệ file 34) — cho
phép từ một điểm chất lượng thấp trong dashboard evaluation drill-down thẳng xuống đúng trace ghi lại
trajectory gây ra vấn đề đó.

## 14. Evaluation metrics

(Bản thân file này định nghĩa các chỉ số trajectory — xem mục 6.) Chỉ số meta: mức độ tương quan giữa
điểm Agent-as-a-Judge và đánh giá con người; tỷ lệ trajectory được rubric và judge đồng thuận (khi hai
lớp bất đồng, thường là dấu hiệu cần xem xét kỹ hơn).

## 15. Ưu điểm

Phát hiện được vấn đề mà đánh giá chỉ final answer bỏ lỡ hoàn toàn — đi đường vòng tốn kém, "đúng nhờ
may mắn", vi phạm policy giữa chừng dù kết quả cuối vẫn hợp lệ; cung cấp tín hiệu debug chi tiết hơn
nhiều khi agent hoạt động sai.

## 16. Nhược điểm

Tốn công đầu tư trajectory logging đầy đủ và xây dựng cả hai lớp đánh giá (rubric + judge); Agent-as-
a-Judge dùng chính một hệ thống agentic để đánh giá nên có chi phí tính toán cao hơn LLM-as-judge đơn
lượt thông thường.

## 17. Khi nên dùng

Mọi agent có nhiều hơn một bước ra quyết định, đặc biệt agent có khả năng gây tác động thực qua tool
call — đánh giá trajectory nên là chuẩn mực, không phải bổ sung tuỳ chọn, cho agent production.

## 18. Khi không nên dùng

Agent thực hiện đúng một lượt gọi model, không có tool call hay nhiều bước suy luận — trong trường
hợp đó không có "trajectory" thực sự để đánh giá, LLM-as-judge chấm output cuối là đủ.

## 19. Pattern liên quan

Bổ sung trực tiếp cho RAG evaluation (file 31, ở tầng generation) khi RAG được bọc trong một agent có
nhiều bước; là một trong ba tầng của Multi-agent evaluation (file 33, ở tầng "đánh giá độc lập từng
agent con"); liên hệ Reasoning/Planning patterns (file 10) cho khía cạnh "plan có hợp lệ không".

## 20. Ví dụ kiến trúc thực tế

Agent xử lý yêu cầu phân tích dữ liệu tài chính (truy vấn nhiều nguồn, tính toán, tổng hợp báo cáo):
mỗi trajectory được log đầy đủ (nguồn nào được truy vấn, công thức tính toán nào được dùng, có bước
nào lặp lại không cần thiết); rubric tự động kiểm tra tham số truy vấn có đúng định dạng không, có gọi
đúng API tính toán không; Agent-as-a-Judge đánh giá tổng thể xem chiến lược phân tích có hợp lý về mặt
tài chính không, có bỏ sót góc nhìn quan trọng nào không — phát hiện được trường hợp agent tính đúng
từng con số riêng lẻ nhưng chọn sai chỉ số để trả lời đúng câu hỏi user thực sự hỏi.

---

*Nguồn tham khảo dùng khi biên soạn: tổng hợp và diễn giải lại các hướng tiếp cận đánh giá agent
trajectory phổ biến trong nghiên cứu và ngành 2025-2026 (bao gồm khái niệm Agent-as-a-Judge và các bộ
benchmark công khai được nêu tên). Nội dung không trích dẫn nguyên văn bất kỳ tài liệu cụ thể nào.*
