# 23. State

[← Về mục lục chính](../README.md)

## 1. Tên pattern

State — hạ tầng lưu trữ đằng sau memory và tiến trình agent: Session vs Durable state, State store
patterns, State versioning, liên hệ Checkpoint.

## 2. Vấn đề cần giải quyết

Memory (file 22) mô tả **loại nội dung** agent cần nhớ; State giải quyết câu hỏi kỹ thuật khác:
**lưu nội dung đó ở đâu, bằng công nghệ gì, sống được bao lâu**. Không tách bạch hai khái niệm này
dẫn tới lỗi thiết kế phổ biến: coi mọi thứ agent cần nhớ đều nên nằm trong process memory (mất khi
restart), hoặc ngược lại đẩy mọi thứ vào database bền vững một cách không cần thiết (tốn chi phí,
tăng độ trễ cho dữ liệu vốn chỉ cần sống trong một lượt chạy).

## 3. Bối cảnh sử dụng

Bất kỳ agent nào chạy quá một lượt gọi model đơn lẻ — cần quyết định rõ phần state nào sống trong
session, phần nào cần bền vững qua restart/deploy/failover.

## 4. Kiến trúc

```
Session state:    Process memory / in-memory cache
                   → mất khi process restart, KHÔNG SAO nếu là working memory tạm thời

Durable state:     Key-value store / Document store / Event-sourced store / Graph-based store
                   → phải sống sót qua restart, deploy, failover

State record:      { schema_version, task_id/session_id, payload, updated_at }
```

## 5. Thành phần

State store (công nghệ lưu trữ cụ thể — xem mục 6 dưới); schema versioning mechanism (trường
`schema_version` gắn với mỗi record); migration logic (đọc state cũ bằng code mới một cách an
toàn).

## 6. Luồng xử lý chi tiết

- **Session state vs Durable state**: câu hỏi đầu tiên khi thiết kế state cho bất kỳ phần dữ liệu
  nào của agent là "nếu process restart ngay bây giờ, mất phần này có sao không?". Nếu không sao
  (ví dụ biến tạm trong một lượt xử lý tool call) → session state, có thể để trong bộ nhớ tiến
  trình. Nếu có sao (ví dụ tiến độ của một task dài đang chạy, cần resume được) → durable state,
  bắt buộc persist ra ngoài process.
- **State store patterns**:
  - *Key-value/document store*: lưu state theo khoá `task_id`/`session_id`, đơn giản, dễ scale
    ngang — phù hợp phần lớn use case cơ bản.
  - *Event-sourced state*: thay vì lưu snapshot trạng thái hiện tại, lưu **chuỗi sự kiện** dẫn tới
    trạng thái đó; trạng thái hiện tại = replay toàn bộ event từ đầu — cho phép audit trail đầy đủ
    (biết chính xác state đã thay đổi thế nào qua thời gian) và "time-travel debugging" (xem lại
    state tại bất kỳ thời điểm nào trong quá khứ), đổi lại đọc trạng thái hiện tại phức tạp hơn
    (cần replay, hoặc cache snapshot định kỳ).
  - *Graph-based state*: phù hợp agent có luồng xử lý nhiều bước phụ thuộc phi tuyến (rẽ nhánh,
    hợp nhánh) — mỗi node trong luồng đọc/ghi vào một state object dùng chung, tự nhiên khớp với
    các framework điều phối agent theo mô hình đồ thị trạng thái.
- **State versioning và migration**: khi agent được nâng cấp, cấu trúc state (plan format, tool
  result schema, metadata) thường thay đổi theo. Mỗi state record cần gắn `schema_version` tường
  minh; cần migration path rõ ràng để code mới đọc được state cũ một cách an toàn (không giả định
  ngầm rằng mọi state đang tồn tại đều theo schema mới nhất).
- **State và checkpoint**: state store là nơi checkpoint (mục 30.1) thực sự ghi dữ liệu vào —
  checkpoint/resume, retry, và saga (mục 30.10) đều phụ thuộc trực tiếp vào việc state được thiết
  kế đủ chi tiết để tái tạo lại chính xác điểm dừng; nếu state thiếu thông tin cần thiết, agent
  không thể resume đúng dù cơ chế checkpoint có hoạt động.

## 7. State và dữ liệu

(Mục này tự tham chiếu — bản thân file đang mô tả kiến trúc lưu trữ dữ liệu.) Điểm quan trọng cần
nhấn mạnh: state record nên luôn có `schema_version`, khoá định danh rõ ràng (`task_id`/
`session_id`/`entity_id`), và `updated_at` để hỗ trợ cả migration lẫn debug.

## 8. Thuật toán liên quan

Event sourcing (mô hình lưu trữ dựa trên chuỗi sự kiện thay vì snapshot); không có thuật toán ML
liên quan — đây thuần tuý là vấn đề kiến trúc dữ liệu.

## 9. Cách triển khai

1. Với mỗi phần dữ liệu agent cần lưu, đặt câu hỏi session hay durable **trước khi** chọn công
   nghệ lưu trữ — quyết định sai ở bước này (ví dụ coi nhầm dữ liệu cần durable là session) gây sự
   cố mất dữ liệu khó phát hiện cho tới khi có sự cố restart thực tế.
2. Bắt đầu với key-value/document store đơn giản cho durable state trừ khi có yêu cầu rõ ràng cần
   audit trail đầy đủ (event-sourced) hoặc luồng phi tuyến phức tạp (graph-based).
3. Gắn `schema_version` ngay từ bản đầu tiên, dù chưa cần migration — thêm sau khi đã có dữ liệu
   production không có version sẽ khó xử lý hơn nhiều.
4. Thiết kế migration path đơn giản nhất có thể (ví dụ default value cho field mới) trước khi cần
   migration phức tạp.

## 10. Tham số cần tuning

TTL cho session state (nếu dùng cache có TTL); tần suất snapshot cho event-sourced state (để tránh
phải replay quá dài mỗi lần đọc); chiến lược sharding/partition cho state store ở quy mô lớn.

## 11. Failure modes

- **Coi session state như durable**: mất dữ liệu quan trọng khi process restart ngoài kế hoạch
  (deploy, crash, scale down) vì dữ liệu chỉ được lưu trong bộ nhớ tiến trình.
- **Thiếu schema_version**: khi cấu trúc state đổi, code mới đọc nhầm state cũ theo schema mới,
  gây lỗi runtime khó truy vết (thường không lỗi rõ ràng ngay mà gây hành vi sai lệch âm thầm).
- **Event-sourced không có snapshot**: đọc trạng thái hiện tại phải replay toàn bộ lịch sử event
  từ đầu, càng lâu càng chậm nếu không có cơ chế snapshot định kỳ.
- **State không đủ chi tiết để checkpoint/resume đúng**: agent "tưởng" đã lưu đủ để resume nhưng
  thực ra thiếu một phần thông tin quan trọng (ví dụ thiếu bước hiện tại trong plan), dẫn tới resume
  sai vị trí.

## 12. Security considerations

State store thường chứa dữ liệu nhạy cảm tương đương hoặc nhiều hơn chính memory nó lưu (bao gồm cả
working memory tạm thời có thể chứa dữ liệu nhạy cảm của task đang xử lý) — cần áp dụng kiểm soát
truy cập tương đương các kho dữ liệu nhạy cảm khác trong tổ chức, không coi nhẹ vì "chỉ là state kỹ
thuật".

## 13. Observability

Theo dõi kích thước state store theo thời gian, độ trễ đọc/ghi, và tỷ lệ lỗi migration (khi state
cũ không đọc được bằng code mới) như các chỉ số vận hành riêng biệt với observability của chính
logic agent.

## 14. Evaluation metrics

Thời gian resume trung bình từ checkpoint (đo hiệu quả thực tế của state design cho reliability);
tỷ lệ mất dữ liệu khi có sự cố restart (nên bằng 0 cho phần đã xác định là durable); độ trễ đọc state
ở quy mô dữ liệu thực tế.

## 15. Ưu điểm

Tách bạch rõ ràng session/durable state và chọn đúng công nghệ lưu trữ theo đặc tính giúp hệ thống
vừa hiệu quả (không lưu thừa những gì không cần bền vững) vừa đáng tin cậy (không mất những gì cần
bền vững).

## 16. Nhược điểm

Thêm một lớp quyết định thiết kế cần cân nhắc kỹ cho mỗi loại dữ liệu; event-sourced/graph-based
state đặc biệt tăng độ phức tạp vận hành so với key-value store đơn giản.

## 17. Khi nên dùng

Mọi agent chạy quá một lượt gọi model đơn lẻ cần suy nghĩ rõ ràng về state ngay từ thiết kế ban
đầu — không để "tự nhiên trở thành session state" chỉ vì đó là lựa chọn dễ nhất lúc viết code đầu
tiên.

## 18. Khi không nên dùng

Agent xử lý một request độc lập, không cần nhớ gì giữa các lần gọi, không cần resume — không cần
đầu tư vào durable state, session state (hoặc thậm chí không cần state nào ngoài input/output của
chính lượt gọi đó) là đủ.

## 19. Pattern liên quan

Là hạ tầng trực tiếp cho Memory patterns (file 22) và Checkpoint and resume (mục 30.1); liên hệ
chặt với Context Engineering (file 24) — quyết định phần state nào cần đưa vào context tại một thời
điểm cụ thể là một bài toán context engineering, tách biệt với việc lưu trữ state đó ở đâu.

## 20. Ví dụ kiến trúc thực tế

Agent xử lý quy trình phê duyệt nhiều bước kéo dài vài ngày (chờ phê duyệt từ nhiều cấp): toàn bộ
tiến độ quy trình là durable state, lưu trong document store theo `task_id`, có `schema_version` để
xử lý an toàn khi logic quy trình được cập nhật giữa chừng vòng đời một số task đang chạy dở; mỗi
lần một cấp phê duyệt xong, checkpoint được ghi vào state store để nếu hệ thống restart, quy trình
resume đúng từ bước đang chờ, không phải bắt đầu lại từ đầu.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về event sourcing và state management trong
hệ thống phân tán, áp dụng vào kiến trúc AI agent. Nội dung là tổng hợp và diễn giải lại.*
