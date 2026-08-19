# 12. Federated multi-agent

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Federated multi-agent — mô hình liên bang, agent thuộc nhiều domain/hạ tầng khác nhau phối hợp
dưới một supervisor chung.

## 2. Vấn đề cần giải quyết

Trong tổ chức thực tế, khả năng và dữ liệu không nằm trong một hệ thống duy nhất — bộ phận kỹ
thuật, tài chính, đối tác bên ngoài, và dịch vụ cloud đều có agent/hệ thống riêng, với model,
memory, tool và chính sách bảo mật khác nhau. Federated multi-agent giải quyết bài toán điều phối
các agent **không thể/không nên tập trung hoá** vào một hạ tầng chung, mà vẫn cần phối hợp trả lời
một yêu cầu duy nhất từ user.

## 3. Bối cảnh sử dụng

Tổ chức có nhiều đội/phòng ban vận hành hệ thống AI riêng; cần tích hợp agent của đối tác bên
ngoài (không kiểm soát được hạ tầng của họ); cần kết hợp agent chạy local (nội bộ, nhạy cảm dữ
liệu) với agent chạy remote (cloud, mở rộng dễ dàng).

## 4. Kiến trúc

```
Enterprise Supervisor
 ├─ Local Engineering Agent   (hạ tầng nội bộ, model/tool/policy riêng)
 ├─ Remote Finance Agent      (hệ thống tài chính riêng, quyền hạn chế)
 ├─ Partner Agent             (thuộc tổ chức khác, chỉ giao tiếp qua contract công khai)
 └─ Cloud Research Agent      (hạ tầng cloud, mở rộng theo nhu cầu)
```

Khác Hierarchical retrieval (thu hẹp phạm vi trong cùng một hệ thống), federation là **nhiều hệ
thống độc lập về hạ tầng, quyền, và vòng đời** được kết nối qua giao thức chuẩn (thường là A2A —
file 26) thay vì gọi hàm nội bộ.

## 5. Thành phần

Enterprise Supervisor (điều phối, không sở hữu logic nghiệp vụ của từng agent con); secure channel
giữa supervisor và mỗi agent liên bang (mã hoá, xác thực riêng cho từng kết nối vì mỗi agent có
thể ở ngoài biên tin cậy của tổ chức); registry ghi nhận agent nào tồn tại, capability gì, thuộc
domain nào.

## 6. Luồng xử lý chi tiết

Supervisor nhận yêu cầu từ user, phân tích cần domain nào tham gia (có thể cần nhiều domain cho
một yêu cầu), gửi task tới từng agent liên bang tương ứng qua secure channel (A2A), mỗi agent liên
bang tự xử lý trong phạm vi của mình (dùng model/tool/memory riêng — supervisor không nhìn thấy
nội bộ), trả kết quả về cho supervisor, supervisor tổng hợp và trả lời user. Mỗi agent liên bang
giữ toàn quyền tự trị vận hành nội bộ; supervisor chỉ điều phối ở mức **hợp đồng task** (task gì,
input gì, output kỳ vọng gì), không can thiệp cách agent con thực hiện.

## 7. State và dữ liệu

Mỗi agent liên bang giữ state riêng, không chia sẻ trực tiếp; supervisor chỉ giữ state ở mức điều
phối (task nào đã gửi tới agent nào, đang chờ kết quả gì) — không nhân bản state nội bộ của các
agent con.

## 8. Thuật toán liên quan

Không có thuật toán riêng — đây là một mô hình kiến trúc tổ chức (organizational pattern) áp dụng
nguyên lý liên bang (federation) từ hệ thống phân tán truyền thống vào ngữ cảnh multi-agent.

## 9. Cách triển khai

1. Xác định ranh giới liên bang theo **quyền sở hữu thực tế** (đội nào sở hữu hạ tầng/dữ liệu nào)
   — không tạo ranh giới liên bang giả tạo cho các agent thực ra cùng một đội quản lý.
2. Chuẩn hoá giao thức giao tiếp giữa supervisor và các agent liên bang bằng A2A (file 26) — không
   gọi API tuỳ biến riêng cho từng agent, tăng chi phí bảo trì tuyến tính theo số agent liên bang.
3. Thiết lập secure channel và identity propagation (mục 29.1) rõ ràng cho từng kết nối liên bang
   — đặc biệt quan trọng khi Partner Agent thuộc tổ chức khác.
4. Xây registry (liên hệ Agent admission control, mục 29.9) để quản lý danh sách agent liên bang
   hợp lệ, tránh supervisor gửi task tới agent giả mạo/không còn hợp lệ.

## 10. Tham số cần tuning

Timeout cho mỗi domain (một domain chậm không nên chặn toàn bộ response — liên hệ Graceful
degradation, mục 30.7); số lượng domain tối đa được gọi song song cho một yêu cầu; retry policy
riêng cho từng loại kết nối (nội bộ vs partner thường cần policy khác nhau).

## 11. Failure modes

- **Domain chậm nhất quyết định tốc độ toàn hệ thống**: không có timeout/fallback riêng theo
  domain, một agent liên bang phản hồi chậm kéo chậm toàn bộ response.
- **Rò rỉ quyền qua supervisor**: supervisor vô tình trở thành điểm có quyền truy cập rộng hơn
  tổng các agent con cộng lại, phá vỡ nguyên tắc phân quyền theo domain.
- **Registry lỗi thời**: agent liên bang đã ngừng hoạt động/đổi endpoint nhưng registry chưa cập
  nhật, supervisor gửi task tới đích không còn hợp lệ.

## 12. Security considerations

Đây là pattern có bề mặt rủi ro bảo mật lớn nhất trong nhóm Multi-Agent vì vượt ra ngoài biên tổ
chức (Partner Agent). Bắt buộc: secure channel cho mọi kết nối liên bang, identity propagation
đúng đắn (mục 29.1 — supervisor không dùng một service account "vạn năng" thay cho toàn bộ
domain), và traceability đầy đủ (biết chính xác agent liên bang nào đã xử lý phần nào của một yêu
cầu, phục vụ audit).

## 13. Observability

Trace phải đi xuyên qua ranh giới liên bang (distributed tracing với correlation ID nhất quán) —
nếu mỗi domain chỉ log nội bộ mà không liên kết được với trace tổng, không thể tái tạo được toàn
bộ luồng xử lý một yêu cầu khi debug sự cố liên domain.

## 14. Evaluation metrics

Liên hệ Multi-agent evaluation (file 33): tỷ lệ thành công theo từng domain liên bang riêng biệt;
latency đóng góp bởi từng domain; tỷ lệ yêu cầu cần phối hợp nhiều hơn 1 domain liên bang.

## 15. Ưu điểm

Cho phép các đội/tổ chức khác nhau phát triển, vận hành, và chịu trách nhiệm cho agent của mình
độc lập, đồng thời vẫn phối hợp được để phục vụ yêu cầu phức tạp cần nhiều domain — không ép buộc
tập trung hoá hạ tầng.

## 16. Nhược điểm

Chi phí điều phối và bảo mật cao hơn đáng kể so với multi-agent trong cùng một hạ tầng; độ trễ
tăng do giao tiếp qua network/protocol chuẩn thay vì gọi hàm nội bộ; khó debug hơn do trace phải
xuyên qua nhiều hệ thống độc lập.

## 17. Khi nên dùng

Khi các agent thực sự thuộc quyền sở hữu/vận hành khác nhau (đội khác, tổ chức khác, hạ tầng khác)
và việc tập trung hoá không khả thi hoặc không mong muốn về mặt tổ chức/bảo mật.

## 18. Khi không nên dùng

Khi tất cả agent thuộc cùng một đội, cùng hạ tầng, cùng chính sách bảo mật — dùng Hybrid
orchestration (mục 21) trong cùng một hệ thống đơn giản và hiệu quả hơn nhiều so với dựng kiến
trúc liên bang không cần thiết.

## 19. Pattern liên quan

Dựa trên A2A (file 26) làm giao thức giao tiếp chuẩn; liên hệ Identity propagation (mục 29.1) và
Agent admission control (mục 29.9) cho khía cạnh bảo mật; có thể kết hợp với Hybrid orchestration
(mục 21) khi bên trong một domain liên bang lại có nhiều agent con cần điều phối theo pattern
khác.

## 20. Ví dụ kiến trúc thực tế

Tập đoàn đa quốc gia với agent kỹ thuật chạy on-premise (dữ liệu nhạy cảm không rời khỏi nội bộ),
agent tài chính chạy trên hạ tầng tuân thủ quy định tài chính riêng, và tích hợp agent của một nhà
cung cấp logistics bên ngoài qua A2A — Enterprise Supervisor điều phối một yêu cầu "kiểm tra tình
trạng đơn hàng và ảnh hưởng ngân sách" bằng cách gọi song song cả ba agent liên bang, mỗi agent xử
lý trong phạm vi và quyền hạn riêng của mình.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về mô hình liên bang (federation) trong hệ
thống phân tán, áp dụng vào kiến trúc multi-agent doanh nghiệp. Nội dung là tổng hợp và diễn giải
lại.*
