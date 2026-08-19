# 14. Contract-net pattern

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Contract-net pattern — manager công bố task, agent gửi đề xuất, manager trao "hợp đồng" cho agent
phù hợp nhất, rồi xác nhận kết quả.

## 2. Vấn đề cần giải quyết

Cần một giao thức phối hợp rõ ràng, có cấu trúc cho việc "giao việc — nhận việc — xác nhận hoàn
thành" giữa các agent không có quan hệ cố định sẵn (khác Supervisor cố định gán việc cho agent con
đã biết trước) — đặc biệt khi tập agent khả dụng có thể thay đổi hoặc manager không biết trước
chính xác agent nào phù hợp nhất cho đến khi nhận được đề xuất.

## 3. Bối cảnh sử dụng

Hệ thống cần cơ chế giao việc động, minh bạch, có thể audit (biết rõ ai đã đề xuất gì, ai được
chọn và vì sao) — phù hợp khi tập worker thay đổi theo thời gian hoặc cần ghi log quyết định giao
việc cho mục đích tuân thủ/audit.

## 4. Kiến trúc

```
Manager announces task
        ↓
Agents submit proposals   (mỗi agent quan tâm gửi đề xuất: cách làm, chi phí, thời gian)
        ↓
Manager awards contract   (chọn 1 đề xuất, thông báo trao "hợp đồng" cho agent đó)
        ↓
Selected agent executes
        ↓
Manager validates result  (xác nhận kết quả có đúng như đề xuất/hợp đồng không)
```

## 5. Thành phần

Announcement channel (nơi manager công bố task, có thể broadcast tới tất cả hoặc multicast tới
nhóm agent đủ điều kiện); proposal format chuẩn (mỗi agent trình bày đề xuất theo cùng cấu trúc để
so sánh được); award/reject mechanism; validation step sau khi agent được chọn hoàn thành việc.

## 6. Luồng xử lý chi tiết

Bốn giai đoạn tuần tự, khác Market-based allocation (file 13, tập trung vào cạnh tranh nhiều tiêu
chí) ở việc Contract-net nhấn mạnh vào **quy trình có cấu trúc rõ ràng và bước xác nhận cuối**:
(1) manager công bố task kèm mô tả yêu cầu, (2) các agent quan tâm và đủ điều kiện gửi đề xuất
(proposal) mô tả cách họ dự định thực hiện cùng ước tính chi phí/thời gian, (3) manager đánh giá
các đề xuất và trao hợp đồng (award) cho một agent — các agent không được chọn nhận thông báo từ
chối rõ ràng, (4) agent được chọn thực hiện task, và **bắt buộc có bước manager validate kết quả**
trước khi coi task hoàn thành — khác với gán việc trực tiếp không có bước xác nhận tường minh.

## 7. State và dữ liệu

Manager cần lưu trạng thái task qua các giai đoạn (announced → proposals received → awarded →
executing → validated) để biết task nào đang ở bước nào, đặc biệt quan trọng khi có nhiều task
được công bố song song.

## 8. Thuật toán liên quan

Không có thuật toán cố định cho việc chọn đề xuất — có thể đơn giản (đề xuất có chi phí thấp nhất
đạt ngưỡng chất lượng) hoặc phức tạp hơn tuỳ ngữ cảnh, tương tự hàm utility ở Market-based
allocation.

## 9. Cách triển khai

1. Chuẩn hoá format proposal (mô tả cách làm, chi phí ước tính, thời gian ước tính) để manager so
   sánh công bằng.
2. Đặt deadline rõ ràng cho giai đoạn nhận đề xuất — không chờ vô thời hạn.
3. Luôn có bước validate kết quả tường minh sau khi agent được trao hợp đồng hoàn thành — đây là
   điểm khác biệt cốt lõi so với gán việc trực tiếp, không nên bỏ qua.
4. Ghi log đầy đủ các giai đoạn cho mục đích audit, đặc biệt hữu ích trong ngữ cảnh cần minh bạch
   quyết định (ví dụ hệ thống có yêu cầu tuân thủ).

## 10. Failure modes

- **Không agent nào đề xuất**: task công bố nhưng không nhận được proposal nào phù hợp — cần cơ
  chế fallback (mở rộng phạm vi công bố, hoặc escalate).
- **Agent được trao hợp đồng thất bại giữa chừng**: cần cơ chế tái công bố task (re-announce) thay
  vì để task treo.
- **Bỏ qua bước validate**: coi task hoàn thành ngay khi agent báo "xong" mà không kiểm tra kết
  quả có đúng như đề xuất — mất đi lợi ích chính của pattern này so với gán việc trực tiếp.

## 11. Security considerations

Announcement không nên chứa thông tin nhạy cảm hơn mức cần thiết để agent đánh giá có nên đề xuất
hay không — chi tiết đầy đủ chỉ nên cung cấp cho agent đã được trao hợp đồng.

## 12. Observability

Log toàn bộ vòng đời mỗi task (announce → proposals → award → execute → validate) với timestamp
từng giai đoạn — phục vụ cả debug lẫn audit quyết định giao việc.

## 13. Evaluation metrics

Tỷ lệ task được validate thành công ngay lần đầu (không cần re-announce); thời gian trung bình từ
announce đến validate; số lượng đề xuất trung bình nhận được mỗi task (quá thấp có thể chỉ ra
phạm vi công bố quá hẹp).

## 14. Ưu điểm

Quy trình minh bạch, có cấu trúc, dễ audit — phù hợp với ngữ cảnh cần giải trình quyết định giao
việc; bước validate tường minh giảm rủi ro chấp nhận kết quả sai.

## 15. Nhược điểm

Overhead giao tiếp qua 4 giai đoạn cao hơn gán việc trực tiếp; không phù hợp với task cần phản hồi
tức thời (toàn bộ vòng đời announce-propose-award-validate cần thời gian).

## 16. Khi nên dùng

Khi cần tính minh bạch/audit cao trong việc giao task, hoặc khi tập agent khả dụng thay đổi động
và manager không có đủ thông tin trước để gán việc trực tiếp.

## 17. Khi không nên dùng

Với hệ thống có quan hệ agent cố định, đã biết rõ agent nào phù hợp với loại task nào — gán trực
tiếp (như Supervisor pattern) đơn giản và nhanh hơn.

## 18. Pattern liên quan

Gần với Market-based task allocation (file 13); có thể kết hợp với Dry-run và approval (mục 11.9)
khi bước validate cần con người xác nhận thêm trước khi coi hợp đồng hoàn tất.

## 19. Ví dụ kiến trúc thực tế

Hệ thống quản lý dự án nội bộ dùng agent: khi có task kỹ thuật mới, manager agent công bố tới các
agent chuyên môn khả dụng (frontend, backend, QA); mỗi agent quan tâm gửi đề xuất kèm ước tính thời
gian; manager trao hợp đồng cho agent phù hợp nhất, và sau khi agent báo hoàn thành, manager chạy
bước validate (kiểm tra output có khớp yêu cầu ban đầu) trước khi đóng task.

## 20. Tên gọi khác/liên hệ lịch sử

Contract Net Protocol là một trong những giao thức phối hợp multi-agent kinh điển trong lĩnh vực
distributed AI — được đưa vào cẩm nang này như một pattern nền tảng, không phải phát minh mới của
tài liệu này.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về Contract Net Protocol trong lĩnh vực hệ
thống multi-agent. Nội dung là tổng hợp và diễn giải lại.*
