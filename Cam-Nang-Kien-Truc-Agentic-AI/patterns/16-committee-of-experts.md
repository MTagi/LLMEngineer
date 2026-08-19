# 16. Committee-of-experts

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Committee-of-experts — router chọn một nhóm agent chuyên môn khác nhau cùng đánh giá một vấn đề,
judge/synthesizer tổng hợp thành một kết luận chung.

## 2. Vấn đề cần giải quyết

Với câu hỏi/vấn đề phức tạp có nhiều góc nhìn chuyên môn hợp lệ (kỹ thuật, pháp lý, tài chính,
vận hành...), một agent đơn — dù có tool tốt — vẫn thiên lệch theo một hướng suy luận nếu chỉ dùng
một system prompt/persona duy nhất. Committee-of-experts giải quyết bằng cách cho **nhiều agent
chuyên môn khác nhau đánh giá độc lập cùng một vấn đề**, rồi tổng hợp — giảm rủi ro bỏ sót góc nhìn
quan trọng mà một agent đơn dễ mắc phải.

## 3. Bối cảnh sử dụng

Quyết định/đánh giá cần cân nhắc nhiều khía cạnh chuyên môn khác nhau cùng lúc — ví dụ đánh giá một
đề xuất dự án cần góc nhìn kỹ thuật, chi phí, và rủi ro pháp lý đồng thời.

## 4. Kiến trúc

```
Vấn đề cần đánh giá
        ↓
Router chọn các "expert" liên quan
        ↓
   ├─ Expert A (chuyên môn kỹ thuật) → đánh giá độc lập
   ├─ Expert B (chuyên môn chi phí) → đánh giá độc lập
   └─ Expert C (chuyên môn pháp lý) → đánh giá độc lập
        ↓
Judge/Synthesizer tổng hợp các đánh giá thành 1 kết luận chung
```

## 5. Thành phần

Router (chọn expert nào liên quan tới vấn đề, không nhất thiết luôn gọi toàn bộ expert có sẵn);
các expert agent — mỗi expert có system prompt/persona/kiến thức chuyên biệt (khác Committee ở tổ
chức: đây không phải nhiều agent làm các phần việc khác nhau như Orchestrator-Worker, mà **cùng
đánh giá một vấn đề** từ góc nhìn khác nhau); judge/synthesizer tổng hợp.

## 6. Luồng xử lý chi tiết

Router phân tích vấn đề và xác định expert nào liên quan (không cần gọi mọi expert cho mọi vấn đề
— ví dụ vấn đề thuần kỹ thuật không cần expert pháp lý tham gia). Mỗi expert được chọn nhận **cùng
một vấn đề** nhưng đánh giá **độc lập**, không thấy đánh giá của expert khác (tránh hiệu ứng
"đồng thuận sớm" — expert sau bị ảnh hưởng bởi ý kiến expert trước thay vì đưa ra đánh giá độc lập
thực sự). Sau khi tất cả expert hoàn thành, judge/synthesizer nhận toàn bộ đánh giá, tổng hợp thành
kết luận chung — có thể là đồng thuận (nếu các expert nhất trí), hoặc trình bày rõ các quan điểm
khác biệt nếu expert bất đồng (không nên "làm phẳng" bất đồng quan trọng thành một câu trả lời giả
đồng thuận).

## 7. State và dữ liệu

Mỗi expert giữ context riêng khi đánh giá (liên hệ Sub-agent context isolation, mục 24.3); judge
cần nhận đủ đánh giá từ tất cả expert đã được router chọn trước khi tổng hợp — cần cơ chế chờ/
timeout rõ ràng cho trường hợp một expert không phản hồi.

## 8. Thuật toán liên quan

Không có thuật toán chuẩn — tương đồng về mặt ý tưởng với "ensemble" trong machine learning cổ
điển (nhiều model độc lập đánh giá, tổng hợp kết quả), áp dụng vào ngữ cảnh agent với các persona/
system prompt khác nhau thay vì các model được train khác nhau.

## 9. Cách triển khai

1. Xác định rõ tập expert cố định với persona/phạm vi chuyên môn tách biệt rõ ràng — tránh chồng
   chéo phạm vi giữa các expert (nếu hai expert cùng đánh giá y hệt một khía cạnh, mất lợi ích đa
   dạng góc nhìn).
2. Đảm bảo mỗi expert đánh giá **thực sự độc lập** (không thấy đánh giá của expert khác trước khi
   đưa ra ý kiến riêng) — vi phạm nguyên tắc này khiến committee suy biến thành một agent đơn với
   nhiều bước rườm rà.
3. Thiết kế judge/synthesizer để **giữ lại bất đồng quan trọng** thay vì luôn cố tạo một câu trả
   lời đồng thuận giả tạo khi các expert thực sự bất đồng về một điểm quan trọng.
4. Dùng Router để chỉ gọi expert liên quan, tránh gọi toàn bộ expert cho mọi vấn đề (tốn chi phí
   không cần thiết).

## 10. Tham số cần tuning

Số expert tối đa tham gia một đánh giá; timeout chờ đánh giá từ mỗi expert; ngưỡng để judge coi là
"đồng thuận" hay "bất đồng đáng kể cần trình bày rõ".

## 11. Failure modes

- **Đồng thuận sớm giả tạo**: nếu vô tình để expert thấy đánh giá của nhau trước khi hoàn thành
  đánh giá riêng, các expert có xu hướng hội tụ về ý kiến giống nhau, mất lợi ích đa dạng góc nhìn.
- **Judge làm phẳng bất đồng quan trọng**: tổng hợp thành một kết luận "trung bình" khi thực ra có
  một expert cảnh báo rủi ro nghiêm trọng mà các expert khác không thấy — cần thiết kế judge ưu
  tiên nêu rõ cảnh báo thiểu số quan trọng, không chỉ lấy đa số.
- **Chi phí tuyến tính theo số expert**: mỗi expert thêm vào là một lệnh gọi model đầy đủ — cần
  cân nhắc số lượng expert hợp lý, không thêm expert "cho chắc" mà không có giá trị phân biệt rõ
  ràng.

## 12. Security considerations

Nếu các expert cần truy cập dữ liệu nhạy cảm khác nhau theo chuyên môn (ví dụ expert tài chính cần
dữ liệu tài chính, expert pháp lý cần hồ sơ pháp lý), áp dụng Least-privilege agent (mục 29.2)
riêng cho từng expert theo đúng phạm vi chuyên môn, không cấp quyền chung cho tất cả expert.

## 13. Observability

Log đầy đủ đánh giá riêng của từng expert (không chỉ kết luận tổng hợp cuối) — cần thiết để hiểu
judge đã cân nhắc gì khi có bất đồng, và để audit khi kết luận cuối bị chất vấn.

## 14. Evaluation metrics

Liên hệ Multi-agent evaluation (file 33): mức độ đa dạng ý kiến giữa các expert (quá đồng nhất có
thể chỉ ra expert chưa thực sự độc lập); tỷ lệ judge giữ lại đúng cảnh báo quan trọng khi có bất
đồng (đánh giá qua case thử nghiệm có "cài" một rủi ro chỉ một expert phát hiện được).

## 15. Ưu điểm

Giảm rủi ro bỏ sót góc nhìn chuyên môn quan trọng so với agent đơn; kết luận có căn cứ rõ ràng từ
nhiều chuyên môn, dễ giải trình hơn quyết định "hộp đen" của một agent đơn.

## 16. Nhược điểm

Chi phí tăng tuyến tính theo số expert; cần thiết kế cẩn thận để tránh đồng thuận giả tạo hoặc làm
phẳng bất đồng quan trọng — nếu làm sai, lợi ích đa góc nhìn biến mất trong khi vẫn gánh chi phí.

## 17. Khi nên dùng

Vấn đề/quyết định thực sự cần nhiều góc nhìn chuyên môn khác biệt, đặc biệt khi hậu quả sai sót
cao (quyết định kinh doanh quan trọng, đánh giá rủi ro).

## 18. Khi không nên dùng

Với vấn đề thuộc một lĩnh vực chuyên môn duy nhất, hoặc khi cần phản hồi nhanh — overhead nhiều
expert không tương xứng lợi ích.

## 19. Pattern liên quan

Khác Orchestrator-Worker (file 15) ở chỗ các "worker" ở đây cùng đánh giá một vấn đề thay vì chia
việc theo hướng khác nhau; liên hệ Red-team and blue-team (file 17) như một dạng đặc biệt hoá của
việc dùng nhiều agent với vai trò đối lập thay vì bổ trợ.

## 20. Ví dụ kiến trúc thực tế

Hệ thống đánh giá đề xuất đầu tư nội bộ: router xác định đề xuất cần expert kỹ thuật, expert tài
chính, và expert rủi ro pháp lý cùng đánh giá độc lập; mỗi expert đưa ra đánh giá riêng kèm mức độ
tự tin; synthesizer tổng hợp thành báo cáo trình bày rõ điểm đồng thuận và điểm bất đồng (nếu expert
rủi ro pháp lý cảnh báo vấn đề mà hai expert khác không đề cập, cảnh báo đó vẫn được nêu rõ trong
kết luận cuối thay vì bị lược bỏ).

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về mô hình đánh giá đa chuyên gia (ensemble/
mixture-of-experts về mặt tổ chức) áp dụng vào kiến trúc multi-agent. Nội dung là tổng hợp và diễn
giải lại.*
