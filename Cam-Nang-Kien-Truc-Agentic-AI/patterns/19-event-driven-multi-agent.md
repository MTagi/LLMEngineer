# 19. Event-driven multi-agent

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Event-driven multi-agent — agent subscribe vào event, phản ứng khi event xảy ra, có thể phát event
mới kích hoạt agent tiếp theo.

## 2. Vấn đề cần giải quyết

Điều phối trực tiếp (một agent/orchestrator gọi trực tiếp agent khác và chờ kết quả) tạo ra
coupling chặt — agent gọi phải biết trước chính xác agent nào cần gọi tiếp theo, và phải chờ đồng
bộ. Với hệ thống có nhiều loại xử lý độc lập, kích hoạt theo các sự kiện không đồng bộ (tài liệu
mới, dữ liệu thay đổi, task hoàn thành ở nơi khác), event-driven multi-agent giải quyết bằng cách
để mỗi agent **subscribe** vào loại event nó quan tâm, phản ứng khi event đó xảy ra, không cần biết
trước ai sẽ phát event hay ai sẽ xử lý event nó phát ra tiếp theo.

## 3. Bối cảnh sử dụng

Pipeline xử lý nhiều giai đoạn không đồng bộ (ví dụ: tài liệu mới → trích xuất → validate → index);
hệ thống cần khả năng mở rộng thêm loại xử lý mới mà không sửa code các agent hiện có; khối lượng
công việc không đều theo thời gian (event-driven tự nhiên xử lý tải không đều tốt hơn polling định
kỳ).

## 4. Kiến trúc

```
DocumentCreated
    ↓ (event bus)
Extraction Agent  (subscribe "DocumentCreated")
    ↓ phát event DocumentExtracted
Validation Agent  (subscribe "DocumentExtracted")
    ↓ phát event DocumentValidated
Indexing Agent    (subscribe "DocumentValidated")
```

Mỗi agent chỉ biết **loại event nó subscribe**, không biết trực tiếp agent nào đã phát event đó hay
agent nào sẽ xử lý event nó phát ra tiếp theo — toàn bộ liên kết được trung gian hoá qua event bus.

## 5. Thành phần

Event bus/message queue (hạ tầng trung gian phân phối event); event schema (định dạng chuẩn cho
mỗi loại event, có versioning); mỗi agent là một consumer subscribe một hoặc nhiều loại event và
có thể đóng vai trò producer phát event mới.

## 6. Luồng xử lý chi tiết

Khi một sự kiện xảy ra (ví dụ tài liệu mới được tạo), một event được phát lên event bus với schema
chuẩn (loại event, payload, correlation ID). Mọi agent đã subscribe loại event đó nhận được và xử
lý độc lập — có thể nhiều agent cùng subscribe một loại event nếu cần xử lý song song các khía cạnh
khác nhau. Sau khi xử lý xong, agent có thể phát event mới (ví dụ `DocumentExtracted`) để kích hoạt
bước tiếp theo trong chuỗi — tạo thành pipeline xử lý không đồng bộ mà không có bất kỳ agent nào
biết toàn bộ chuỗi từ đầu đến cuối.

## 7. State và dữ liệu

Mỗi event cần **correlation ID** xuyên suốt chuỗi xử lý (để trace một tài liệu cụ thể qua toàn bộ
pipeline dù đi qua nhiều agent độc lập); event schema cần versioning tường minh vì các agent có thể
được deploy độc lập và không đồng bộ về thời điểm nâng cấp — agent cũ và mới có thể cùng tồn tại
trong một khoảng thời gian chuyển tiếp.

## 8. Thuật toán liên quan

Không có thuật toán riêng — dựa trên mô hình publish-subscribe kinh điển trong kiến trúc hệ thống
phân tán, áp dụng vào ngữ cảnh multi-agent.

## 9. Cách triển khai

1. Định nghĩa event schema rõ ràng và có versioning ngay từ đầu — thay đổi schema sau này mà không
   có kế hoạch tương thích ngược sẽ phá vỡ agent đang chạy dựa trên schema cũ.
2. Bắt buộc gắn correlation ID vào mọi event trong cùng một chuỗi xử lý logic, xuyên suốt từ event
   gốc đến mọi event phái sinh.
3. Thiết kế mỗi agent **idempotent** với việc xử lý event (liên hệ Idempotent tool pattern, mục
   11.7) — event bus thông thường có thể gửi trùng event trong một số tình huống lỗi mạng, agent
   cần xử lý đúng dù nhận trùng.
4. Kết hợp cơ chế dead-letter (mục 30.5) cho event xử lý thất bại nhiều lần — không để event "biến
   mất" âm thầm khi agent xử lý gặp lỗi liên tục.

## 10. Tham số cần tuning

Retry policy khi agent xử lý event thất bại; TTL của event trong queue; số lượng consumer song song
cho mỗi loại event (throughput vs độ phức tạp điều phối).

## 11. Failure modes

- **Event bị mất**: không có cơ chế retry/dead-letter, một số event không bao giờ được xử lý mà
  không ai biết — cần giám sát chủ động (không chỉ tin hệ thống "chắc đã xử lý hết").
- **Event trùng lặp không được xử lý idempotent**: agent tạo hiệu ứng phụ (ghi dữ liệu, gọi API)
  nhiều lần cho cùng một sự kiện logic do nhận event bị gửi trùng.
- **Thiếu correlation ID**: khi debug một tài liệu cụ thể bị xử lý sai, không thể tra cứu lại toàn
  bộ chuỗi event liên quan vì các event không liên kết được với nhau.
- **Schema drift không kiểm soát**: agent mới deploy với schema mới trong khi agent cũ (subscribe
  cùng loại event) chưa cập nhật, gây lỗi parse hoặc xử lý sai payload.

## 12. Security considerations

Event bus cần kiểm soát quyền publish/subscribe theo domain — không phải mọi agent nên có quyền
phát mọi loại event hoặc đọc mọi topic, đặc biệt khi event chứa dữ liệu nhạy cảm.

## 13. Observability

Distributed tracing dựa trên correlation ID là bắt buộc — vì không có một luồng thực thi tuyến tính
duy nhất để log tuần tự, cần khả năng tái tạo lại toàn bộ chuỗi event của một tài liệu/task cụ thể
từ log phân tán của nhiều agent độc lập.

## 14. Evaluation metrics

Độ trễ trung bình/tối đa từ event gốc đến khi toàn bộ chuỗi xử lý hoàn thành; tỷ lệ event vào
dead-letter queue; tỷ lệ event được xử lý idempotent đúng (không tạo hiệu ứng phụ trùng lặp khi test
gửi lại event).

## 15. Ưu điểm

Giảm coupling mạnh giữa các agent — có thể thêm agent xử lý loại event mới mà không cần sửa code
agent hiện có; xử lý tải không đều tự nhiên tốt hơn polling định kỳ; mở rộng quy mô dễ dàng bằng
cách thêm consumer song song.

## 16. Nhược điểm

Khó debug hơn luồng gọi trực tiếp (cần distributed tracing thay vì đọc log tuần tự); cần hạ tầng
event bus/message queue riêng; độ phức tạp vận hành cao hơn (schema versioning, dead-letter
handling, idempotency).

## 17. Khi nên dùng

Pipeline nhiều giai đoạn không đồng bộ, cần khả năng mở rộng linh hoạt, hoặc khối lượng công việc
không đều theo thời gian.

## 18. Khi không nên dùng

Với tương tác cần phản hồi đồng bộ tức thời (user đang chờ trực tiếp) — event-driven thêm độ trễ
và độ phức tạp không cần thiết so với gọi trực tiếp.

## 19. Pattern liên quan

Liên hệ Event-driven ingestion (mục 3.2) ở tầng dữ liệu; là nền tảng cho Choreography (file 20, khi
không có orchestrator trung tâm); thường kết hợp với Shared artifact workspace (file 18) — event
báo hiệu artifact mới đã sẵn sàng để agent tiếp theo xử lý.

## 20. Ví dụ kiến trúc thực tế

Pipeline xử lý tài liệu doanh nghiệp: webhook từ hệ thống quản lý tài liệu phát event
`DocumentCreated`; Extraction Agent subscribe, trích xuất nội dung, phát `DocumentExtracted`;
Validation Agent subscribe, kiểm tra chất lượng trích xuất, phát `DocumentValidated`; Indexing Agent
subscribe, đưa vào index RAG (liên hệ file 03) — toàn bộ chuỗi chạy không đồng bộ, mỗi agent có thể
được scale/deploy độc lập theo tải riêng của từng giai đoạn.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về kiến trúc hướng sự kiện (event-driven
architecture) và mô hình publish-subscribe trong hệ thống phân tán, áp dụng vào multi-agent. Nội
dung là tổng hợp và diễn giải lại.*
