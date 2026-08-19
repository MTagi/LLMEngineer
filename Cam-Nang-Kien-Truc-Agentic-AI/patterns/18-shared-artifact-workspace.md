# 18. Shared artifact workspace

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Shared artifact workspace — nhiều agent cộng tác qua file/artifact dùng chung thay vì chỉ trao đổi
qua message trực tiếp.

## 2. Vấn đề cần giải quyết

Giao tiếp thuần qua message giữa các agent (A gửi message cho B, B trả lời A) không phù hợp với
công việc có tính "xây dựng dần" — nhiều agent cùng đóng góp vào một tài liệu thiết kế, một
codebase, một báo cáo — nơi trạng thái công việc là **sản phẩm đang hình thành**, không phải một
chuỗi hội thoại. Shared artifact workspace giải quyết bằng cách để các agent đọc/ghi vào một không
gian file dùng chung, giống cách một nhóm người thật cộng tác qua repository hoặc thư mục tài liệu
chia sẻ.

## 3. Bối cảnh sử dụng

Task tạo ra sản phẩm cụ thể cần nhiều agent đóng góp các phần khác nhau theo thời gian (không phải
tất cả cùng lúc) — phát triển phần mềm nhiều giai đoạn, viết tài liệu kỹ thuật nhiều phần, review
và sửa lặp lại.

## 4. Kiến trúc

```
Shared Workspace
 ├─ requirements.md          (agent Planning tạo/cập nhật)
 ├─ architecture.md          (agent Architecture tạo, tham chiếu requirements.md)
 ├─ implementation/          (agent Coding tạo, tuân theo architecture.md)
 ├─ test-results.json        (agent Testing tạo, chạy trên implementation/)
 └─ review-comments.json     (agent Review tạo, tham chiếu toàn bộ các file trên)
```

Mỗi agent đọc các artifact liên quan đã có, thực hiện phần việc của mình, ghi/cập nhật artifact
tương ứng — không cần agent khác "biết" trực tiếp agent nào đã làm gì qua message, mà thông qua
chính trạng thái của workspace.

## 5. Thành phần

Workspace lưu trữ (file system, object storage, hoặc git repository); quy ước đặt tên/cấu trúc file
rõ ràng (để agent biết tìm gì ở đâu); (tuỳ chọn) cơ chế khoá/versioning để tránh hai agent ghi đè
lẫn nhau khi làm việc gần như đồng thời.

## 6. Luồng xử lý chi tiết

Khác giao tiếp qua message (nơi thông tin trao đổi trực tiếp, có tính tạm thời), shared artifact
workspace coi **artifact chính là trạng thái công việc bền vững**. Một agent bắt đầu công việc bằng
cách đọc các artifact liên quan hiện có trong workspace (ví dụ agent Architecture đọc
`requirements.md`), thực hiện phần việc của mình, rồi ghi kết quả thành artifact mới hoặc cập nhật
artifact hiện có (`architecture.md`). Agent tiếp theo trong chuỗi (ví dụ agent Coding) không cần
biết trực tiếp agent Architecture đã "nói" gì — chỉ cần đọc `architecture.md` đã hoàn thiện. Cách
này tự nhiên tạo ra một dạng lịch sử/audit trail (mỗi artifact là một điểm dừng có thể xem lại),
khác hẳn hội thoại message có thể trôi qua và khó tra cứu lại về sau.

## 7. State và dữ liệu

Bản thân workspace **là** state chính của hệ thống multi-agent — không cần một state store riêng
biệt song song, vì trạng thái công việc hiện tại thể hiện trực tiếp qua nội dung các file trong
workspace tại một thời điểm.

## 8. Thuật toán liên quan

Không có thuật toán riêng — về bản chất mượn mô hình cộng tác qua version control/shared document
(tương tự cách con người dùng git repository hoặc thư mục tài liệu chia sẻ) áp dụng cho multi-
agent.

## 9. Cách triển khai

1. Định nghĩa rõ quy ước cấu trúc workspace (tên file, thư mục, định dạng) trước khi các agent bắt
   đầu — quy ước lỏng lẻo khiến agent khó biết đọc/ghi ở đâu.
2. Cân nhắc dùng git (hoặc hệ thống versioning tương tự) làm nền cho workspace nếu cần lịch sử thay
   đổi đầy đủ và khả năng agent xem diff giữa các phiên bản.
3. Xử lý rõ ràng trường hợp ghi đồng thời (concurrent write) — khoá file, hoặc quy ước mỗi agent
   chỉ ghi vào phần workspace thuộc phạm vi của mình để tránh xung đột.
4. Kết hợp với một cơ chế điều phối (Hybrid orchestration, mục 21, hoặc Event-driven, file 19) để
   quyết định agent nào chạy khi nào — bản thân shared workspace chỉ giải quyết vấn đề *lưu trữ và
   chia sẻ trạng thái*, không tự động điều phối trình tự.

## 10. Tham số cần tuning

Cơ chế khoá file (pessimistic locking vs optimistic với version check); tần suất agent poll
workspace để kiểm tra artifact mới (nếu không dùng event-driven trigger); giới hạn kích thước/thời
gian giữ lại lịch sử phiên bản.

## 11. Failure modes

- **Ghi đè xung đột**: hai agent cùng cập nhật một artifact gần như đồng thời, một bản ghi đè mất
  bản kia mà không có cảnh báo.
- **Artifact không đồng bộ**: agent B đọc artifact của agent A ở trạng thái cũ (do cache hoặc do
  chưa kiểm tra lại) trong khi A đã cập nhật phiên bản mới — dẫn tới B làm việc dựa trên thông tin
  lỗi thời.
- **Workspace phình to không kiểm soát**: không có quy ước dọn dẹp artifact tạm/trung gian, workspace
  tích luỹ ngày càng nhiều file gây khó khăn cho agent mới tham gia hiểu trạng thái hiện tại.

## 12. Security considerations

Nếu các agent thuộc các domain quyền khác nhau (liên hệ Federated multi-agent, file 12), cần kiểm
soát quyền đọc/ghi theo từng phần của workspace — không phải mọi agent nên có quyền ghi vào mọi
file, ngay cả khi tất cả cùng chia sẻ một workspace.

## 13. Observability

Log mọi thao tác đọc/ghi vào workspace (agent nào, file nào, thời điểm nào) — kết hợp với
versioning, đây chính là nguồn trace tự nhiên cho toàn bộ quá trình cộng tác, thường dễ theo dõi
hơn so với chuỗi message trong pattern giao tiếp trực tiếp.

## 14. Evaluation metrics

Tỷ lệ xung đột ghi đè xảy ra; thời gian trung bình từ khi một artifact được cập nhật đến khi agent
phụ thuộc phát hiện và xử lý; độ đầy đủ/nhất quán của workspace ở trạng thái cuối so với yêu cầu ban
đầu.

## 15. Ưu điểm

Tạo ra trạng thái công việc bền vững, dễ audit, dễ cho con người xem xét giữa chừng (khác hội thoại
message trôi qua khó tra cứu); phù hợp tự nhiên với công việc có tính xây dựng dần theo thời gian.

## 16. Nhược điểm

Cần xử lý đúng vấn đề đồng bộ/xung đột ghi vốn không phát sinh trong giao tiếp message thuần; agent
cần chủ động kiểm tra workspace thay vì được "đẩy" thông tin trực tiếp (trừ khi kết hợp thêm cơ chế
event-driven).

## 17. Khi nên dùng

Task tạo ra sản phẩm cụ thể (tài liệu, code, báo cáo) cần nhiều agent đóng góp qua nhiều giai đoạn,
đặc biệt khi cần con người xem xét được trạng thái trung gian.

## 18. Khi không nên dùng

Với tương tác hội thoại thuần (không tạo sản phẩm cụ thể cần lưu trữ), giao tiếp qua message trực
tiếp đơn giản và tự nhiên hơn.

## 19. Pattern liên quan

Thường kết hợp với Event-driven multi-agent (file 19, để trigger agent khi artifact mới xuất hiện)
hoặc Hybrid orchestration (mục 21); artifact ở đây liên hệ với Artifact memory (mục 22.8) nhưng ở
cấp độ multi-agent thay vì một agent đơn.

## 20. Ví dụ kiến trúc thực tế

Hệ thống phát triển tính năng phần mềm tự động: agent Planning tạo `requirements.md`, agent
Architecture đọc và tạo `architecture.md`, agent Coding đọc `architecture.md` và tạo code trong
`implementation/`, agent Testing chạy test và ghi `test-results.json`, agent Review đọc toàn bộ
workspace và ghi `review-comments.json` — con người có thể xem workspace ở bất kỳ thời điểm nào để
theo dõi tiến độ, thay vì phải đọc lại toàn bộ lịch sử hội thoại giữa các agent.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về mô hình cộng tác qua tài liệu/artifact chia
sẻ (tương tự version control) áp dụng vào kiến trúc multi-agent. Nội dung là tổng hợp và diễn giải
lại.*
