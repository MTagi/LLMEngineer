# 03. Data ingestion patterns

[← Về mục lục chính](../README.md)

Nhóm này gồm 5 pattern kiểm soát **cách dữ liệu đi vào indexing pipeline** (xem file 02): Batch,
Event-driven, Change Data Capture, Incremental indexing, Temporal indexing. Áp dụng chung khung 20
tiêu chí cho cả nhóm, tách riêng phần khác biệt giữa các pattern con.

## 1. Tên pattern

Batch ingestion · Event-driven ingestion · Change Data Capture (CDC) · Incremental indexing ·
Temporal indexing.

## 2. Vấn đề cần giải quyết

Dữ liệu nguồn (tài liệu, database, wiki...) thay đổi liên tục nhưng index không thể re-embed toàn
bộ corpus mỗi lần có 1 thay đổi nhỏ — vừa tốn chi phí embedding, vừa gây downtime/lag không cần
thiết. Cần chọn đúng chiến lược đưa dữ liệu vào index theo đặc tính của nguồn: tĩnh, bán tĩnh, hay
thay đổi liên tục; có cần biết "phiên bản nào đang hiệu lực tại thời điểm nào" hay không.

## 3. Bối cảnh sử dụng

| Pattern | Dùng khi |
|---|---|
| Batch | Nguồn ít thay đổi (user manual, policy, wiki nội bộ) |
| Event-driven | Nguồn thay đổi liên tục, cần freshness cao |
| CDC | Nguồn là database giao dịch (customer, order, contract...) |
| Incremental indexing | Muốn tránh re-embed toàn bộ mỗi lần chạy indexing |
| Temporal indexing | Dữ liệu có phiên bản, cần trả lời đúng theo "thời điểm hiệu lực" |

## 4. Kiến trúc

```
Batch:          Scheduler → Scan Data Source → Detect Changed Documents → Reprocess → Update Index
Event-driven:   Document Changed → Event Bus → Parser → Chunk and Embed → Index Update
CDC:            DB Transaction Log → CDC → Knowledge Projection → Search Index
Incremental:    document_hash / chunk_hash → so sánh với lần trước → chỉ xử lý phần khác
Temporal:       Mỗi record index có {version, valid_from, valid_to, is_current}
```

## 5. Thành phần

Scheduler (batch) hoặc event bus/message queue (event-driven); CDC connector đọc transaction log
của database nguồn; hàm tính checksum (document/chunk hash); trường metadata version + effective
date cho temporal indexing; knowledge projection layer cho CDC (không embed thẳng row database).

## 6. Luồng xử lý chi tiết

- **Batch**: scheduler kích hoạt định kỳ → quét toàn bộ nguồn → so sánh để phát hiện tài liệu đã
  đổi → chỉ reprocess phần đổi → cập nhật index.
- **Event-driven**: mỗi sự kiện thay đổi tài liệu phát lên event bus → consumer parse, chunk,
  embed → ghi/cập nhật index gần như realtime.
- **CDC**: đọc trực tiếp transaction log của database (insert/update/delete) thay vì poll toàn bộ
  bảng → biến đổi thành **knowledge projection** có ý nghĩa nghiệp vụ (ví dụ gộp customer +
  contract + product thành 1 record thay vì embed từng row rời rạc) → ghi vào search index.
- **Incremental indexing**: trước khi xử lý một document/chunk, tính `document_hash`/`chunk_hash`
  và so với hash đã lưu ở lần index trước; chỉ tái xử lý phần có hash khác; đồng thời phải **xoá**
  vector/lexical term/graph edge của phiên bản cũ (dễ bị quên, gây "rác" index tích luỹ).
- **Temporal indexing**: mỗi record index kèm `version`, `valid_from`, `valid_to`, `is_current`;
  câu hỏi "hiện tại" lọc `is_current=true`, câu hỏi "tại thời điểm X" lọc theo khoảng
  `valid_from`/`valid_to` chứa X.

## 7. State và dữ liệu

Bảng/kho lưu `document_hash`, `chunk_hash` (độc lập với chính index — thường là 1 bảng tracking
nhỏ, ví dụ key-value `id → hash → last_indexed_at`); với temporal indexing, version/effective date
phải là **field filter được** trong index, không chỉ nằm trong văn bản chunk.

## 8. Thuật toán liên quan

Hashing (thường dùng hash mật mã như SHA-256, không cần tính bảo mật mà cần tránh va chạm) để phát
hiện thay đổi; không có thuật toán ML nào riêng ở tầng ingestion.

## 9. Cách triển khai

1. Chọn 1 chiến lược nền (batch hoặc event-driven) làm chính theo đặc tính nguồn — nhiều hệ thống
   dùng cả hai: batch cho quét định kỳ đảm bảo không sót, event-driven cho cập nhật nhanh.
2. Thêm CDC nếu một phần nguồn là database giao dịch — không dùng batch/poll toàn bảng cho
   trường hợp này vì tốn tài nguyên và độ trễ cao.
3. Bắt buộc thêm incremental indexing (hash-based) bất kể chọn batch hay event-driven, để tránh
   re-embed lãng phí.
4. Thêm temporal indexing nếu nghiệp vụ có khái niệm "phiên bản"/"hiệu lực theo thời gian" (chính
   sách, hợp đồng, quy định).

## 10. Tham số cần tuning

Chu kỳ batch (giờ/ngày); độ trễ chấp nhận được của event-driven (near-realtime vs vài giây); TTL
của bảng tracking hash; cách xử lý document bị xoá ở nguồn (soft-delete vs xoá cứng khỏi index).

## 11. Failure modes

- **Batch job fail âm thầm**: không alert, index "đóng băng" mà không ai biết cho tới khi user
  báo dữ liệu cũ.
- **Event bị mất**: event-driven không có cơ chế retry/dead-letter (mục 30.5) khiến một số thay
  đổi không bao giờ vào index — cần kết hợp batch định kỳ làm lưới an toàn (reconciliation).
- **Quên xoá index cũ**: incremental indexing chỉ thêm bản mới mà không xoá vector/term cũ →
  index phình to, kết quả tìm kiếm trả về cả bản đã lỗi thời.
- **CDC embed thẳng row**: bỏ qua bước xây knowledge projection, embed nguyên văn từng dòng
  database → context vô nghĩa khi retrieve (thiếu liên kết giữa các bảng liên quan).

## 12. Security considerations

CDC đọc trực tiếp transaction log có thể vô tình lộ dữ liệu nhạy cảm không nên đưa vào index tìm
kiếm (ví dụ cột lương, thông tin cá nhân) — cần whitelist rõ field nào được đưa vào knowledge
projection, không mặc định lấy hết.

## 13. Observability

Theo dõi độ trễ giữa "thời điểm thay đổi ở nguồn" và "thời điểm phản ánh trong index" (freshness
lag) như một metric vận hành chính; alert khi batch job không chạy đúng lịch hoặc event consumer
bị tụt lại (consumer lag).

## 14. Evaluation metrics

Freshness lag (thời gian trễ trung bình/tối đa); tỷ lệ document xử lý thành công/thất bại mỗi
lần chạy; hiệu quả incremental (% tài liệu được skip nhờ hash không đổi, càng cao càng tiết kiệm
chi phí).

## 15. Ưu điểm

Tránh lãng phí chi phí embedding cho dữ liệu không đổi; giữ index luôn phản ánh đúng nguồn theo
mức độ tươi mới cần thiết; temporal indexing cho phép hệ thống trả lời chính xác cả câu hỏi về
lịch sử, không chỉ hiện tại.

## 16. Nhược điểm

Tăng độ phức tạp vận hành (cần bảng tracking hash, event infrastructure, hoặc CDC connector); CDC
đặc biệt đòi hỏi hạ tầng và kỹ năng vận hành riêng, không phải database nào cũng hỗ trợ tốt.

## 17. Khi nên dùng

Batch: mặc định an toàn cho hầu hết nguồn tài liệu tĩnh/bán tĩnh. Event-driven: khi có sẵn hạ tầng
message queue và cần độ trễ thấp. CDC: khi nguồn chính là OLTP database. Incremental: gần như luôn
nên có, chi phí cài đặt thấp so với lợi ích. Temporal: khi nghiệp vụ có khái niệm hiệu lực theo
thời gian.

## 18. Khi không nên dùng

Với corpus rất nhỏ, tĩnh hoàn toàn (không đổi sau khi tạo) — re-index toàn bộ theo yêu cầu thủ
công đơn giản hơn nhiều so với xây cả 5 pattern này.

## 19. Pattern liên quan

Kết quả của indexing patterns là input cho Chunking (file 05) và Vector index (file 08). Temporal
indexing liên hệ trực tiếp tới Retrieval guardrail (mục 29.5) — phải lọc đúng version khi trả lời.

## 20. Ví dụ kiến trúc thực tế

Hệ thống RAG cho tài liệu chính sách nội bộ: batch job quét toàn bộ kho tài liệu mỗi đêm làm lưới
an toàn; đồng thời webhook từ hệ thống quản lý tài liệu bắn event mỗi khi có tài liệu mới/sửa để
cập nhật gần-realtime; mỗi tài liệu và chunk có hash để tránh re-embed; mỗi chính sách có
`version`/`valid_from`/`valid_to` để trả lời đúng "chính sách hiện hành" khi được hỏi, và đúng
"chính sách tại thời điểm tháng 1/2025" khi được hỏi ngược lại.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về data engineering pipeline, CDC và
event-driven architecture. Nội dung là tổng hợp và diễn giải lại.*
