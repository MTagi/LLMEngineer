# 17. Red-team and blue-team

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Red-team and blue-team — hai agent với vai trò đối lập có chủ đích: một bên tạo giải pháp, một bên
chủ động tìm lỗi/rủi ro, một judge phân xử.

## 2. Vấn đề cần giải quyết

Một agent tự đánh giá lại chính output của mình (self-critique đơn giản) thường thiên vị — model
có xu hướng "bảo vệ" câu trả lời nó vừa tạo ra thay vì thực sự tìm lỗi một cách khách quan. Red-
team/blue-team giải quyết bằng cách **tách vai trò đối kháng tường minh**: một agent (blue) chuyên
tạo giải pháp, một agent (red) được giao nhiệm vụ rõ ràng là tìm lỗi/rủi ro/đường tấn công trong
giải pháp đó — vai trò đối lập giúp red-agent không bị thiên vị theo hướng "giải pháp đã ổn".

## 3. Bối cảnh sử dụng

Đánh giá chất lượng/an toàn của một giải pháp/output trước khi triển khai thật — đặc biệt hữu ích
cho code review tự động, kiểm tra kế hoạch hành động có rủi ro, hoặc rà soát nội dung trước khi
công bố.

## 4. Kiến trúc

```
Blue Agent → tạo giải pháp
        ↓
Red Agent → chủ động tìm lỗi, rủi ro, hoặc attack path trong giải pháp của Blue
        ↓
Judge → xác định vấn đề Red nêu ra có hợp lệ không (không phải mọi phát hiện của Red đều đúng)
        ↓
Blue Agent → sửa giải pháp dựa trên các vấn đề đã được Judge xác nhận hợp lệ
        ↓ (có thể lặp lại vòng Red → Judge → Blue nhiều lượt)
```

## 5. Thành phần

Blue agent (persona "người xây dựng", tối ưu cho tạo/cải thiện giải pháp); Red agent (persona
"người phá", tối ưu cho tìm lỗi/tấn công — thường cần system prompt khuyến khích tư duy đối kháng
tường minh, không chỉ "kiểm tra lỗi thông thường"); Judge (phân xử tính hợp lệ của phát hiện Red
nêu ra, tránh việc chấp nhận mù quáng mọi cảnh báo của Red).

## 6. Luồng xử lý chi tiết

Blue agent tạo giải pháp ban đầu. Red agent nhận giải pháp đó với nhiệm vụ **chủ động tìm vấn đề**
— khác đánh giá thông thường ("giải pháp này có ổn không"), Red được khuyến khích tư duy như kẻ
tấn công/người phản biện triệt để ("giải pháp này có thể bị khai thác/thất bại như thế nào"). Judge
xem xét từng vấn đề Red nêu ra và xác định vấn đề nào thực sự hợp lệ (không phải mọi thứ Red tìm ra
đều là vấn đề thật — Red có thể quá nhạy, tìm ra rủi ro lý thuyết không thực tế). Blue agent nhận
lại các vấn đề đã được Judge xác nhận, sửa giải pháp tương ứng. Vòng lặp này có thể chạy nhiều lượt
đến khi Red không còn tìm được vấn đề hợp lệ mới, hoặc chạm giới hạn số vòng lặp (liên hệ
Termination guard, mục 30.9).

## 7. State và dữ liệu

Cần lưu lịch sử các vòng lặp (giải pháp phiên bản nào, vấn đề nào được nêu, vấn đề nào được Judge
xác nhận, sửa đổi tương ứng) — hữu ích cho audit và để tránh Red lặp lại đúng một phát hiện đã được
xử lý ở vòng trước.

## 8. Thuật toán liên quan

Không có thuật toán chuẩn — dựa vào việc thiết kế system prompt tạo ra sự phân cực vai trò rõ ràng
giữa hai agent (tương tự tinh thần "adversarial" trong kiểm thử bảo mật truyền thống, áp dụng vào
ngữ cảnh multi-agent LLM).

## 9. Cách triển khai

1. Thiết kế system prompt của Red agent khuyến khích **tư duy đối kháng thực sự**, không chỉ liệt
   kê checklist kiểm tra thông thường — nếu Red hành xử giống một reviewer tiêu chuẩn, lợi ích đặc
   thù của pattern này (tìm ra thứ mà đánh giá thông thường bỏ sót) sẽ mất đi.
2. Bắt buộc có Judge độc lập — không để Blue tự quyết định vấn đề Red nêu có hợp lệ hay không
   (thiên vị tương tự self-critique đơn thuần).
3. Giới hạn số vòng lặp Red-Judge-Blue rõ ràng để tránh chi phí không kiểm soát.
4. Log toàn bộ phát hiện của Red (kể cả những phát hiện bị Judge bác bỏ) — hữu ích để đánh giá chất
   lượng của chính Red agent theo thời gian.

## 10. Tham số cần tuning

Số vòng lặp tối đa; ngưỡng để Judge coi một phát hiện là "đủ nghiêm trọng" cần Blue sửa ngay vs có
thể bỏ qua; mức độ "hung hăng" của Red (điều chỉnh qua prompt — quá nhẹ thì bỏ sót vấn đề thật, quá
mạnh thì tạo nhiều cảnh báo giả tốn công Judge xử lý).

## 11. Failure modes

- **Red quá yếu**: chỉ lặp lại các kiểm tra hời hợt giống review thông thường, không mang lại giá
  trị vượt trội so với self-critique đơn giản.
- **Red quá nhạy**: tạo ra quá nhiều cảnh báo giả (false positive), làm Judge/Blue tốn công xử lý
  vấn đề không thực sự quan trọng.
- **Judge thiên vị theo Blue**: nếu Judge dùng cùng model/persona gần giống Blue, có xu hướng bác
  bỏ phát hiện của Red dễ dàng hơn mức cần thiết — nên cân nhắc Judge độc lập rõ ràng về persona.
- **Vòng lặp không hội tụ**: Blue sửa vấn đề này lại tạo vấn đề khác, Red liên tục tìm ra vấn đề
  mới không dừng — cần termination guard chặt.

## 12. Security considerations

Nếu dùng pattern này cho việc rà soát bảo mật thực sự (ví dụ Red agent thử tìm lỗ hổng trong code),
cần chạy trong Tool sandbox (mục 11.5) nghiêm ngặt — Red agent về bản chất được khuyến khích "thử
các cách khai thác", cần giới hạn phạm vi hành động thực tế của nó dù chỉ đang mô phỏng.

## 13. Observability

Log đầy đủ mỗi vòng lặp: giải pháp phiên bản nào, phát hiện của Red, quyết định của Judge, sửa đổi
của Blue — cho phép truy vết toàn bộ quá trình cải thiện giải pháp qua các vòng đối kháng.

## 14. Evaluation metrics

Tỷ lệ phát hiện của Red được Judge xác nhận hợp lệ (đo chất lượng Red agent); số vòng lặp trung
bình đến khi hội tụ; so sánh chất lượng giải pháp cuối cùng với/không có vòng Red-team (đo lợi ích
thực sự của pattern so với chi phí thêm vào).

## 15. Ưu điểm

Phát hiện được lớp vấn đề mà self-critique đơn giản hoặc review thông thường dễ bỏ sót, nhờ vai trò
đối kháng tường minh loại bỏ thiên vị "tự bảo vệ giải pháp của chính mình".

## 16. Nhược điểm

Chi phí cao hơn đáng kể so với tạo giải pháp một lần (cần ít nhất 3 agent: Blue, Red, Judge, cộng
nhiều vòng lặp); cần thiết kế prompt cẩn thận để Red thực sự hiệu quả, không chỉ là một reviewer
"nhẹ nhàng" khoác tên gọi khác.

## 17. Khi nên dùng

Giải pháp/kế hoạch có rủi ro thực sự nếu sai sót không bị phát hiện — code trước khi triển khai
production, kế hoạch hành động có tác động lớn, nội dung trước khi công bố công khai.

## 18. Khi không nên dùng

Với output có rủi ro thấp, hoặc khi cần phản hồi nhanh — chi phí nhiều vòng đối kháng không tương
xứng với mức độ rủi ro thực tế của use case.

## 19. Pattern liên quan

Là dạng đặc biệt hoá của Committee-of-experts (file 16) với vai trò đối lập thay vì bổ trợ; liên hệ
Reflection trong Reasoning patterns (file 10, khái niệm self-critique cơ bản) — Red-team/blue-team
là phiên bản mạnh hơn của self-critique nhờ tách vai trò thành các agent riêng biệt.

## 20. Ví dụ kiến trúc thực tế

Quy trình review code tự động trước khi merge: Blue agent (hoặc chính developer) tạo pull request;
Red agent chủ động tìm lỗi bảo mật, edge case chưa xử lý, và khả năng regression; Judge (có thể là
một agent riêng hoặc con người) xác định phát hiện nào hợp lệ cần sửa trước khi merge; Blue agent
cập nhật code theo phản hồi đã xác nhận.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về mô hình đối kháng (adversarial review, red
team/blue team) trong kiểm thử bảo mật truyền thống, áp dụng vào kiến trúc multi-agent LLM. Nội
dung là tổng hợp và diễn giải lại.*
