# 20. Choreography

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Choreography — không có orchestrator trung tâm; mỗi agent tự phản ứng với event và tự phát event
tiếp theo, hành vi tổng thể của hệ thống "nổi lên" (emerge) từ tương tác phân tán.

## 2. Vấn đề cần giải quyết

Orchestration (một agent/orchestrator trung tâm biết toàn bộ luồng và điều khiển từng bước) tạo ra
một điểm phụ thuộc duy nhất (single point of coordination) — mọi logic điều phối tập trung ở một
nơi, khó mở rộng độc lập, và orchestrator trở thành nút thắt cả về hiệu năng lẫn bảo trì khi số
agent tăng. Choreography giải quyết bằng cách loại bỏ hoàn toàn vai trò điều phối trung tâm: mỗi
agent tự biết mình cần phản ứng với event gì và phát ra event gì tiếp theo — không có "nhạc trưởng"
duy nhất, tương tự cách các vũ công trong một buổi biểu diễn *choreography* mỗi người tự biết vai
trò của mình theo kịch bản đã thống nhất trước, không cần người chỉ huy đứng giữa ra hiệu từng
bước.

## 3. Bối cảnh sử dụng

Hệ thống có nhiều đội phát triển độc lập, mỗi đội sở hữu một agent và muốn tự quản lý logic phản
ứng của agent mình mà không phụ thuộc vào một orchestrator chung do đội khác duy trì; hệ thống cần
khả năng mở rộng/thay đổi từng agent độc lập mà không đòi hỏi sửa logic điều phối trung tâm.

## 4. Kiến trúc

```
(Không có Orchestrator ở giữa)

Event A xảy ra → Agent 1 (subscribe A) phản ứng → phát Event B
                                                        ↓
                          Agent 2 (subscribe B) phản ứng → phát Event C
                                                                ↓
                          Agent 3 (subscribe C) phản ứng → phát Event D
```

Về hạ tầng, Choreography dùng chung cơ chế event bus với Event-driven multi-agent (file 19) — khác
biệt nằm ở **triết lý thiết kế**: event-driven đơn thuần vẫn có thể có một agent đóng vai trò điều
phối ẩn (biết toàn bộ chuỗi, dù giao tiếp qua event); choreography chủ động **không có agent nào
biết toàn bộ chuỗi** — mỗi agent chỉ biết event nó nhận và event nó phát, hành vi toàn cục nổi lên
từ tổng hợp các quyết định cục bộ.

## 5. Thành phần

Event bus (giống file 19); mỗi agent tự chứa logic quyết định "khi nhận event X, tôi làm gì và
phát event gì" — không có thành phần điều phối trung tâm riêng biệt.

## 6. Luồng xử lý chi tiết

Không có bước "lập kế hoạch toàn cục" nào — hành vi hệ thống được xác định hoàn toàn bởi tổng hợp
logic cục bộ của từng agent đã được thiết kế trước (thường thông qua thoả thuận/hợp đồng event
schema giữa các đội phát triển, tương tự cách các vũ công thống nhất kịch bản trước buổi diễn).
Khi một event xảy ra, agent liên quan phản ứng theo logic riêng của nó, phát event tiếp theo nếu
cần, agent khác tiếp tục phản ứng — chuỗi phản ứng này tự "chảy" qua hệ thống mà không có điểm điều
khiển trung tâm theo dõi tiến trình tổng thể.

## 7. State và dữ liệu

Không có state điều phối trung tâm (khác Orchestrator giữ state điều phối) — trạng thái "tiến độ
tổng thể" của một luồng xử lý chỉ có thể tái tạo được bằng cách tổng hợp log/trace của tất cả agent
liên quan qua correlation ID (liên hệ file 19, mục 7), không có một nơi duy nhất để tra cứu.

## 8. Thuật toán liên quan

Không có thuật toán riêng — là biến thể triết lý thiết kế của Event-driven multi-agent (file 19),
mượn khái niệm choreography vs orchestration từ kiến trúc microservices truyền thống.

## 9. Cách triển khai

1. Thống nhất rõ **hợp đồng event** (event nào, schema gì, ai subscribe, ai publish) giữa các đội
   sở hữu từng agent — đây là "kịch bản" thay thế cho vai trò orchestrator.
2. Đầu tư mạnh vào distributed tracing (liên hệ file 19, mục 13) — vì không có orchestrator để hỏi
   "tiến độ hiện tại thế nào", khả năng quan sát toàn cục phụ thuộc hoàn toàn vào chất lượng
   tracing.
3. Cân nhắc kỹ trước khi chọn choreography thay vì orchestration thuần — chỉ nên chọn khi có lý do
   tổ chức/kỹ thuật thực sự cần loại bỏ điểm điều phối trung tâm (ví dụ nhiều đội độc lập), không
   chọn "vì nghe có vẻ hiện đại hơn".
4. Thiết kế mỗi agent với khả năng phản ứng đúng dù nhận event theo thứ tự không hoàn toàn dự đoán
   được (network có thể làm event đến không đúng thứ tự phát ra).

## 10. Tham số cần tuning

Giống Event-driven multi-agent (file 19): retry policy, TTL event, dead-letter handling — cộng
thêm thoả thuận rõ ràng về version của event schema giữa các đội độc lập sở hữu từng agent.

## 11. Failure modes

- **Hành vi toàn cục khó dự đoán**: vì không ai "thấy" toàn bộ luồng trước khi chạy, các tương tác
  phức tạp giữa nhiều agent có thể tạo ra hành vi ngoài dự kiến mà không đội nào chủ động thiết kế
  — đặc biệt khi số agent tăng lên.
  Cần
  test tích hợp (integration test) mô phỏng toàn bộ chuỗi, không chỉ test từng agent riêng lẻ.
- **Không ai chịu trách nhiệm khi luồng "kẹt"**: nếu một agent không phản ứng đúng (bug, down), 
  không có orchestrator nào phát hiện "luồng đang chờ ở đâu" — cần alerting chủ động dựa trên
  timeout theo correlation ID.
- **Thay đổi một agent ảnh hưởng ngầm tới agent khác**: vì phụ thuộc là ngầm định qua event schema
  thay vì tường minh qua lời gọi trực tiếp, một đội đổi logic agent của mình có thể vô tình phá vỡ
  giả định của agent khác mà không nhận ra ngay.

## 12. Security considerations

Giống Event-driven multi-agent (file 19) — cộng thêm: khi nhiều đội độc lập sở hữu agent, cần chính
sách rõ ràng ai được publish/subscribe loại event nào, tránh một đội vô tình (hoặc cố ý) subscribe
event chứa dữ liệu ngoài phạm vi họ nên thấy.

## 13. Observability

Bắt buộc hơn cả Event-driven multi-agent thuần — vì thiếu điểm điều phối trung tâm, distributed
tracing với correlation ID không phải "nên có" mà là **điều kiện tiên quyết để vận hành được**, nếu
không sẽ không thể trả lời câu hỏi cơ bản "một yêu cầu cụ thể hiện đang ở đâu trong hệ thống".

## 14. Evaluation metrics

Giống file 19 (độ trễ, tỷ lệ dead-letter); thêm chỉ số riêng: tỷ lệ luồng xử lý hoàn thành đúng dự
kiến khi test tích hợp toàn chuỗi (đo mức độ "hành vi nổi lên" có khớp với thiết kế mong muốn hay
không).

## 15. Ưu điểm

Loại bỏ hoàn toàn điểm phụ thuộc/nút thắt duy nhất; cho phép các đội phát triển độc lập thực sự,
không phụ thuộc vào việc sửa đổi một orchestrator chung do đội khác sở hữu.

## 16. Nhược điểm

Khó hiểu, khó debug, khó dự đoán hành vi toàn cục hơn đáng kể so với orchestration — đánh đổi tính
độc lập của từng đội lấy độ khó trong việc nắm bắt/kiểm soát hành vi hệ thống tổng thể.

## 17. Khi nên dùng

Tổ chức có nhiều đội thực sự độc lập, mỗi đội sở hữu agent riêng, và lợi ích của việc loại bỏ điều
phối trung tâm (giảm coupling tổ chức) lớn hơn chi phí về khả năng quan sát/kiểm soát.

## 18. Khi không nên dùng

Với hệ thống do một đội duy nhất phát triển, hoặc khi cần khả năng kiểm soát/audit chặt chẽ luồng
xử lý — Hybrid orchestration (mục 21) với một điểm điều phối tường minh dễ vận hành và debug hơn
nhiều.

## 19. Pattern liên quan

Là biến thể triết lý của Event-driven multi-agent (file 19); đối lập trực tiếp với các pattern có
orchestrator tường minh (Federated, file 12; Hybrid orchestration, mục 21) — nhiều hệ thống thực tế
kết hợp cả hai: choreography giữa các domain lớn, orchestration bên trong mỗi domain.

## 20. Ví dụ kiến trúc thực tế

Nền tảng thương mại điện tử với nhiều đội độc lập: đội Đơn hàng phát event `OrderPlaced`; đội Kho
vận subscribe, tự động kiểm tra tồn kho và phát `InventoryReserved` hoặc `InventoryUnavailable`;
đội Thanh toán subscribe `InventoryReserved`, xử lý thanh toán và phát `PaymentCompleted`; đội
Vận chuyển subscribe `PaymentCompleted` để bắt đầu quy trình giao hàng — không đội nào sở hữu một
"orchestrator đơn hàng" trung tâm, toàn bộ quy trình nổi lên từ các agent phản ứng độc lập theo hợp
đồng event đã thống nhất.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về mô hình choreography vs orchestration trong
kiến trúc microservices, áp dụng vào ngữ cảnh multi-agent. Nội dung là tổng hợp và diễn giải lại.*
