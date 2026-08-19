# 21. Hybrid orchestration

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Hybrid orchestration — kết hợp có chủ đích nhiều pattern điều phối (serial, parallel, supervisor,
handoff, event-driven) trong cùng một hệ thống, chọn đúng pattern cho đúng phần việc.

## 2. Vấn đề cần giải quyết

Không hệ thống multi-agent thực tế nào đủ đơn giản để chỉ cần một pattern điều phối duy nhất xuyên
suốt. Một use case thường có: vài bước phải làm tuần tự vì phụ thuộc lẫn nhau, vài bước có thể chạy
song song vì độc lập, một điểm cần chọn domain xử lý (Supervisor), một đoạn hội thoại cần chuyển
giao ngữ cảnh giữa các agent (Handoff), và một phần việc nền không cần phản hồi tức thời
(Event-driven). Cố ép toàn bộ hệ thống theo một pattern duy nhất (ví dụ toàn bộ serial, hoặc toàn
bộ event-driven) tạo ra kiến trúc gượng ép — chỗ cần song song bị làm tuần tự gây chậm không cần
thiết, chỗ cần tức thời bị làm event-driven gây độ trễ không cần thiết.

## 3. Bối cảnh sử dụng

Hầu như mọi hệ thống multi-agent production có độ phức tạp thực tế (không phải demo đơn giản) —
đây là pattern "meta" áp dụng cho việc thiết kế tổng thể, không phải một kỹ thuật hẹp cho một tình
huống cụ thể.

## 4. Kiến trúc

```
User Request
     ↓
[Supervisor] chọn domain cần xử lý
     ↓
[Serial] Bước A → Bước B  (A phải xong trước B mới bắt đầu được, vì B cần output của A)
     ↓
[Parallel] { Kiểm tra X, Kiểm tra Y, Kiểm tra Z } chạy đồng thời (độc lập nhau)
     ↓
[Handoff] chuyển hội thoại sang Agent chuyên biệt nếu user cần hỗ trợ sâu hơn
     ↓
[Event-driven] phát event cho tác vụ nền (ví dụ gửi báo cáo, cập nhật index) không cần user chờ
```

## 5. Thành phần

Không có thành phần hạ tầng riêng biệt mới — Hybrid orchestration tái sử dụng thành phần của các
pattern thành viên (Supervisor logic, cơ chế chạy song song, event bus cho phần event-driven,
context transfer cho handoff). Thành phần "mới" duy nhất là **quyết định thiết kế** về việc dùng
pattern nào cho phần nào của hệ thống.

## 6. Luồng xử lý chi tiết

Không có một luồng xử lý cố định — đặc trưng của pattern này chính là **luồng xử lý khác nhau theo
từng phần** của hệ thống, được quyết định dựa trên đặc tính của từng phần việc: phần việc có phụ
thuộc dữ liệu tuần tự dùng Serial; phần việc độc lập nhau dùng Parallel (chạy đồng thời, tổng hợp
kết quả sau); phần việc cần chọn đúng domain xử lý dùng Supervisor (liên hệ Federated, file 12,
hoặc đơn giản hơn nếu cùng hạ tầng); phần việc là hội thoại kéo dài cần chuyển giao giữa các agent
chuyên biệt dùng Handoff; phần việc nền không cần phản hồi tức thời dùng Event-driven (file 19).

## 7. State và dữ liệu

State được quản lý theo từng đoạn tương ứng với pattern đang áp dụng ở đoạn đó — đoạn Serial/
Parallel dùng working memory thông thường (mục 22.1); đoạn Event-driven dùng correlation ID (liên
hệ file 19); đoạn Handoff cần cơ chế chuyển giao ngữ cảnh rõ ràng giữa agent cũ và agent mới (không
để user phải lặp lại thông tin đã cung cấp).

## 8. Thuật toán liên quan

Không có thuật toán riêng — là sự tổng hợp có chủ đích của các pattern đã mô tả ở file 12-20.

## 9. Cách triển khai

1. Phân tích luồng xử lý tổng thể của use case, đánh dấu rõ từng đoạn: đoạn nào có phụ thuộc tuần
   tự thật sự, đoạn nào độc lập có thể song song, đoạn nào cần điều phối theo domain, đoạn nào là
   nền không cần chờ.
2. Chọn pattern cho **từng đoạn** dựa trên đặc tính đoạn đó (áp dụng đúng tiêu chí "khi nên dùng"
   đã mô tả ở từng file pattern riêng lẻ 12-20), không áp một pattern chung cho toàn hệ thống.
3. Xác định rõ **ranh giới chuyển giao** giữa các đoạn dùng pattern khác nhau — đây là điểm dễ phát
   sinh lỗi nhất (ví dụ dữ liệu từ đoạn Parallel cần được tổng hợp đúng cách trước khi đưa vào đoạn
   Serial tiếp theo).
4. Tránh over-engineering: không thêm một pattern phức tạp (ví dụ Event-driven) cho một đoạn đơn
   giản chỉ vì "để nhất quán với phần khác của hệ thống".

## 10. Tham số cần tuning

Không có tham số runtime chung — mỗi đoạn kế thừa tham số của pattern nó áp dụng (timeout của
Serial/Parallel, retry policy của Event-driven, v.v.).

## 11. Failure modes

- **Ranh giới mờ giữa các pattern**: không rõ đoạn nào dùng pattern gì, dẫn tới code lẫn lộn logic
  điều phối, khó bảo trì và khó debug khi có lỗi.
- **Ép một pattern cho cả hệ thống "vì đơn giản"**: chọn một pattern duy nhất (ví dụ toàn Serial)
  cho toàn hệ thống dù một số đoạn rõ ràng có thể song song — mất hiệu năng không cần thiết.
- **Chuyển giao state sai giữa các đoạn**: dữ liệu/context không được truyền đúng khi chuyển từ
  đoạn dùng pattern này sang đoạn dùng pattern khác (ví dụ mất context khi Handoff từ agent A sang
  agent B).

## 12. Security considerations

Mỗi đoạn kế thừa yêu cầu bảo mật của pattern nó áp dụng (Identity propagation cho Federated/
Supervisor, quyền publish/subscribe cho Event-driven) — cần đảm bảo identity/quyền được truyền
nhất quán **xuyên suốt** dù đi qua nhiều đoạn dùng pattern khác nhau, không bị "rơi rớt" ở điểm
chuyển giao.

## 13. Observability

Trace tổng thể cần liên kết được qua **mọi đoạn**, dù mỗi đoạn có cơ chế log khác nhau (Serial/
Parallel dễ trace tuyến tính, Event-driven cần correlation ID) — nên dùng một correlation ID/trace
ID nhất quán xuyên suốt toàn bộ hệ thống bất kể đoạn nào đang xử lý.

## 14. Evaluation metrics

Đánh giá theo từng đoạn bằng metric tương ứng pattern đó (file 31-33), cộng với metric tổng thể
end-to-end (latency toàn luồng, tỷ lệ thành công toàn luồng) để đảm bảo việc kết hợp nhiều pattern
không tạo ra điểm nghẽn ẩn ở ranh giới chuyển giao.

## 15. Ưu điểm

Tối ưu đúng đắn cho từng phần của hệ thống thay vì đánh đổi hiệu năng/độ phức tạp một cách gượng ép
để giữ "một pattern duy nhất cho tất cả" — phản ánh đúng thực tế rằng các phần việc khác nhau có
đặc tính khác nhau.

## 16. Nhược điểm

Đòi hỏi hiểu biết đầy đủ về nhiều pattern (12-20) để áp dụng đúng chỗ; hệ thống có nhiều "loại
logic điều phối" khác nhau cùng tồn tại có thể khó onboarding cho thành viên mới nếu không tài
liệu hoá rõ ràng ranh giới và lý do chọn từng pattern.

## 17. Khi nên dùng

Gần như mọi hệ thống multi-agent production có độ phức tạp thực tế nên áp dụng tư duy này ngay từ
giai đoạn thiết kế, thay vì chọn một pattern đơn lẻ rồi cố ép mọi phần việc theo nó.

## 18. Khi không nên dùng

Với hệ thống multi-agent rất đơn giản (2-3 agent, luồng cố định, không có nhu cầu song song hay
nền) — một pattern đơn (ví dụ Supervisor thuần) là đủ, không cần tư duy hybrid phức tạp.

## 19. Pattern liên quan

Là pattern "tổng hợp" của toàn bộ Phần IV (file 12-20) — hiểu rõ từng pattern thành viên là điều
kiện tiên quyết để áp dụng Hybrid orchestration đúng cách.

## 20. Ví dụ kiến trúc thực tế

Hệ thống hỗ trợ khách hàng doanh nghiệp: Supervisor nhận yêu cầu và chọn domain (kỹ thuật/tài
chính/tài khoản); trong domain kỹ thuật, chạy Parallel một số bước chẩn đoán độc lập (kiểm tra log,
kiểm tra cấu hình, kiểm tra phiên bản) rồi Serial bước đề xuất giải pháp (phụ thuộc kết quả chẩn
đoán); nếu vấn đề phức tạp cần chuyên viên người thật, Handoff hội thoại kèm đầy đủ ngữ cảnh đã thu
thập; sau khi giải quyết xong, phát Event-driven cho tác vụ nền (cập nhật knowledge base, gửi khảo
sát hài lòng) không cần user chờ.

---

*Nguồn tham khảo dùng khi biên soạn: tổng hợp và diễn giải lại các khuyến nghị chung về kết hợp
nhiều mô hình orchestration (serial, concurrent, supervisor-based) trong kiến trúc multi-agent
doanh nghiệp phức tạp. Nội dung không trích dẫn nguyên văn.*
