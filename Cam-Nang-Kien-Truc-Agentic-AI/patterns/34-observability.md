# 34. Observability

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Observability — kiến trúc tracing phân cấp cho hệ thống agentic (Authentication Span, Router Span,
Retrieval Span với BM25/Vector/Reranker con, Agent Span với Planning/Tool/Context/Evaluation Span
con, Generation Span), phân biệt rõ với Evaluation.

## 2. Vấn đề cần giải quyết

Evaluation (file 31-33) trả lời "hệ thống có tốt không" ở mức tổng hợp, nhưng không tự nó trả lời
được "hệ thống đang làm gì ngay bây giờ, lỗi cụ thể ở đâu, và tại sao" khi có sự cố xảy ra trong
production. Một hệ thống RAG/agent hiện đại có nhiều tầng xử lý độc lập (retrieval, nhiều loại agent
con, generation) — không tách riêng từng tầng trong tracing, khi có lỗi hoặc chậm bất thường, đội vận
hành chỉ thấy "request này chậm/lỗi" mà không biết chính xác tầng nào gây ra vấn đề.

## 3. Bối cảnh sử dụng

Mọi hệ thống RAG/agent chạy production — quan trọng ngang với evaluation, không phải tính năng bổ
sung: evaluation đo chất lượng theo thời gian, observability là công cụ duy nhất để chẩn đoán sự cố
theo thời gian thực.

## 4. Kiến trúc

```
Trace
 ├─ Authentication Span
 ├─ Router Span
 ├─ Retrieval Span
 │   ├─ BM25 Span
 │   ├─ Vector Span
 │   └─ Reranker Span
 ├─ Agent Span
 │   ├─ Planning Span
 │   ├─ Tool Span
 │   ├─ Context Span (độ dài context, số lần compaction — liên hệ mục 24)
 │   └─ Evaluation Span
 └─ Generation Span
```

## 5. Thành phần

Tracing infrastructure (thu thập, lưu trữ, truy vấn span/trace); correlation ID nhất quán xuyên suốt
một request; span metadata (prompt/model/index version, liên hệ mục 9); dashboard/alerting dựa trên
dữ liệu trace.

## 6. Luồng xử lý chi tiết

Mỗi request đi qua hệ thống được ghi lại thành một **trace** duy nhất, bên trong chia thành các
**span** con tương ứng từng tầng xử lý — mỗi span ghi lại thời gian bắt đầu/kết thúc, trạng thái
thành công/lỗi, và metadata liên quan riêng của tầng đó. *Authentication Span* ghi lại quá trình xác
thực identity của request (liên hệ Identity propagation, mục 29.1). *Router Span* ghi lại quyết định
định tuyến (nếu hệ thống có router/Supervisor chọn domain xử lý). *Retrieval Span* tách riêng thành
ba span con — BM25 Span (lexical search, liên hệ file 7), Vector Span (semantic search, liên hệ file
8), Reranker Span (fusion/reranking, liên hệ file 9) — cho phép biết chính xác giai đoạn nào của
retrieval đang chậm hoặc trả kết quả kém. *Agent Span* tách thành bốn span con — Planning Span (quá
trình lập kế hoạch, liên hệ file 10), Tool Span (từng lượt gọi tool, liên hệ file 11), Context Span
(độ dài context hiện tại và số lần compaction đã thực hiện, liên hệ trực tiếp mục 24 Context
Engineering — đây là điểm quan sát trực tiếp hiện tượng context rot mục 24.5 đang diễn ra hay chưa),
Evaluation Span (nếu agent tự đánh giá/reflect giữa chừng, liên hệ Reasoning patterns file 10).
*Generation Span* ghi lại lượt gọi model tạo câu trả lời cuối cùng.

Production RAG/agent cần tách span retrieval khỏi generation một cách tường minh (không gộp chung
thành một "lượt xử lý" mơ hồ), lưu đầy đủ prompt version, model version, và index version dùng cho
mỗi trace, và liên kết được quality score (từ hệ thống evaluation, file 31-33) ngược về từng trace cụ
thể — cho phép từ một điểm chất lượng thấp trong dashboard evaluation drill-down thẳng xuống đúng
trace và đúng span gây ra vấn đề.

## 7. State và dữ liệu

Trace/span data thường lưu trong hệ thống tracing chuyên dụng (khác state store của agent, file 23)
— cần chính sách retention rõ ràng (giữ bao lâu, vì khối lượng dữ liệu trace lớn theo traffic); mỗi
span nên gắn correlation ID nhất quán với chính trace cha để có thể truy vấn toàn bộ cây span của một
request.

## 8. Thuật toán liên quan

Không có thuật toán ML riêng — đây là hạ tầng kỹ thuật (distributed tracing) áp dụng khái niệm chuẩn
từ observability hệ thống phân tán truyền thống (span, trace, correlation ID) vào ngữ cảnh RAG/agent.

## 9. Cách triển khai

1. Thiết kế cây span (mục 4) khớp đúng kiến trúc thực tế của hệ thống — nếu hệ thống có thêm tầng xử
   lý không có trong cây mẫu (ví dụ multi-agent phối hợp), thêm span tương ứng chứ không gộp vào span
   có sẵn không đúng bản chất.
2. Bắt buộc gắn correlation ID nhất quán xuyên suốt một request, kể cả khi request đi qua nhiều
   service/agent khác nhau (liên hệ trực tiếp Observability của Event-driven multi-agent, file 19 mục
   13, và Choreography, file 20 mục 13 — nơi đây là điều kiện tiên quyết để vận hành được).
3. Lưu version (prompt, model, index) làm metadata bắt buộc trên mỗi trace ngay từ đầu — thiếu thông
   tin này, không thể truy ngược một vấn đề chất lượng về đúng phiên bản gây ra nó khi debug sau này.
4. Xây dựng liên kết hai chiều giữa hệ thống evaluation (file 31-33) và tracing — điểm chất lượng thấp
   phải trỏ được về đúng trace, không chỉ tồn tại như một con số tổng hợp độc lập.
5. Đặc biệt chú ý Context Span cho agent chạy dài — đây là công cụ quan sát trực tiếp để phát hiện
   sớm context rot (mục 24.5) và đánh giá hiệu quả của compaction/note-taking (mục 24.1-24.2) đang áp
   dụng.

## 10. Tham số cần tuning

Chính sách retention dữ liệu trace (bao lâu, cân bằng chi phí lưu trữ với nhu cầu điều tra sự cố cũ);
tỷ lệ sampling nếu traffic quá lớn để trace 100% request (trade-off giữa chi phí tracing và độ đầy đủ
dữ liệu); ngưỡng cảnh báo cho từng loại span (độ trễ bất thường của Retrieval Span, tỷ lệ lỗi bất
thường của Tool Span).

## 11. Failure modes

- **Không tách span retrieval khỏi generation**: khi hệ thống chậm/lỗi, không biết tầng nào gây ra
  vấn đề, mất nhiều thời gian điều tra hơn cần thiết.
- **Thiếu correlation ID nhất quán qua nhiều service**: không thể ghép nối trace hoàn chỉnh khi request
  đi qua nhiều thành phần (đặc biệt nghiêm trọng với Event-driven, file 19, và Choreography, file 20).
- **Không lưu version metadata**: khi phát hiện vấn đề chất lượng, không thể xác định chính xác phiên
  bản prompt/model/index nào gây ra, khó khoanh vùng và khó rollback đúng.
- **Không liên kết evaluation với trace**: điểm chất lượng chỉ tồn tại như con số tổng hợp, không có
  khả năng drill-down xuống nguyên nhân cụ thể.

## 12. Security considerations

Span data có thể chứa nội dung nhạy cảm (prompt, kết quả tool call, dữ liệu truy xuất) — cần áp dụng
kiểm soát truy cập tương đương dữ liệu production, không coi trace như dữ liệu debug ít nhạy cảm; cần
cân nhắc redact hoặc mã hoá các trường chứa PII/secret trước khi lưu trữ dài hạn (liên hệ Output
guardrail, mục 29.7).

## 13. Observability

(Bản thân file này là nội dung observability.) Điểm nhấn cần nhắc lại: observability trả lời "đang
xảy ra gì, ở đâu, tại sao" — khác vai trò với evaluation (file 31-33) trả lời "có tốt không" — hai hệ
thống bổ sung cho nhau, không thay thế nhau, và nên được thiết kế liên kết chặt chẽ với nhau ngay từ
đầu.

## 14. Evaluation metrics

Độ trễ trung bình/phân vị (p50, p95, p99) theo từng loại span; tỷ lệ lỗi theo từng loại span; tỷ lệ
trace có đầy đủ metadata bắt buộc (version, correlation ID); thời gian trung bình đội vận hành cần để
xác định nguyên nhân gốc một sự cố (đo hiệu quả thực tế của kiến trúc tracing).

## 15. Ưu điểm

Cho phép chẩn đoán chính xác vấn đề theo thời gian thực, tách biệt rõ tầng nào trong hệ thống gây ra
sự cố; liên kết chặt với evaluation tạo thành vòng phản hồi đầy đủ từ phát hiện chất lượng thấp tới
điều tra nguyên nhân cụ thể.

## 16. Nhược điểm

Thêm chi phí hạ tầng và lưu trữ đáng kể, đặc biệt ở traffic lớn nếu trace 100% request; thiết kế cây
span không khớp đúng kiến trúc thực tế làm giảm giá trị của toàn bộ hệ thống tracing.

## 17. Khi nên dùng

Mọi hệ thống RAG/agent chạy production — không có ngoại lệ, observability là hạ tầng vận hành cơ bản
tương đương logging/monitoring trong bất kỳ hệ thống phần mềm production nào khác, đặc thù thêm ở chỗ
cần tách theo đúng cấu trúc các tầng RAG/agent.

## 18. Khi không nên dùng

Không có trường hợp hợp lý để bỏ hoàn toàn observability trong production; có thể giảm nhẹ độ chi
tiết (ví dụ gộp bớt span con) cho hệ thống thử nghiệm/demo nội bộ chưa phục vụ traffic thực.

## 19. Pattern liên quan

Bổ sung trực tiếp cho Evaluation (file 31-33) — hai hệ thống nên liên kết chặt; Context Span liên hệ
trực tiếp Context Engineering (file 24) và Context rot mitigation (mục 30.11); correlation ID xuyên
suốt là điều kiện tiên quyết cho Event-driven multi-agent (file 19) và Choreography (file 20).

## 20. Ví dụ kiến trúc thực tế

Hệ thống RAG/agent hỗ trợ tra cứu và xử lý yêu cầu nội bộ doanh nghiệp: mỗi request có một trace duy
nhất từ Authentication Span tới Generation Span; khi user báo cáo câu trả lời chậm bất thường, đội
vận hành mở trace tương ứng, thấy Retrieval Span bình thường nhưng Agent Span có Context Span cho
thấy độ dài context đã vượt ngưỡng và không có lần compaction nào được thực hiện — xác định chính xác
nguyên nhân là context rot (mục 24.5) chưa được xử lý đúng, thay vì phải đoán mò giữa hàng chục nguyên
nhân có thể khác.

---

*Nguồn tham khảo dùng khi biên soạn: tổng hợp và diễn giải lại các nguyên tắc distributed tracing
chuẩn trong hệ thống phần mềm, áp dụng vào kiến trúc RAG/agent production. Nội dung không trích dẫn
nguyên văn bất kỳ tài liệu cụ thể nào.*
