# 22. Memory patterns

[← Về mục lục chính](../README.md)

10 loại memory một agent/hệ thống agent cần quản lý: Working, Conversation, Summary, Semantic,
Episodic, Procedural, Entity, Artifact memory, Memory consolidation, Memory forgetting.

## 1. Tên pattern

Working memory · Conversation memory · Summary memory · Semantic memory · Episodic memory ·
Procedural memory · Entity memory · Artifact memory · Memory consolidation · Memory forgetting.

## 2. Vấn đề cần giải quyết

"Memory" thường bị đơn giản hoá thành "một vector database lưu lịch sử chat" — cách hiểu này bỏ
sót phần lớn các loại thông tin một agent thực sự cần nhớ và quên đi theo những quy luật khác nhau.
Mỗi loại memory có đặc tính khác nhau về thời gian sống, cách truy xuất, và mức độ cần chính xác —
nhầm lẫn giữa chúng (ví dụ nhét tất cả vào một vector store chung) khiến hệ thống vừa tốn kém vừa
kém hiệu quả ở mọi loại truy vấn.

## 3. Bối cảnh sử dụng

| Loại memory | Dùng khi cần nhớ |
|---|---|
| Working | Trạng thái của task đang chạy ngay bây giờ |
| Conversation | Lịch sử message trong phiên hiện tại |
| Summary | Bản tóm tắt lịch sử dài để tiết kiệm token |
| Semantic | Fact/kiến thức tổng quát học được qua thời gian |
| Episodic | Sự kiện/task cụ thể đã xảy ra trong quá khứ |
| Procedural | Cách làm/quy trình/kỹ năng tái sử dụng |
| Entity | Thông tin gắn với một user/customer/product/project cụ thể |
| Artifact | File/report/code/kết quả truy vấn đã tạo ra |

## 4. Kiến trúc

```
Agent
 ├─ Working memory       (plan, tool results, current step, errors — sống trong 1 lượt chạy)
 ├─ Conversation memory  (message user↔agent — sống trong 1 phiên)
 ├─ Summary memory       (bản nén của Conversation khi dài — sống qua nhiều phiên nếu cần)
 ├─ Semantic memory      (fact tổng quát — sống lâu dài, độc lập phiên)
 ├─ Episodic memory      (task/event cụ thể — sống lâu dài, gắn timestamp)
 ├─ Procedural memory    (quy trình/kỹ năng — sống lâu dài, ít thay đổi)
 ├─ Entity memory        (theo user/customer/project — sống lâu dài, theo entity)
 └─ Artifact memory      (file/report/code đã tạo — sống lâu dài, có thể lớn về dung lượng)
```

## 5. Thành phần

Mỗi loại memory có thể dùng hạ tầng lưu trữ khác nhau phù hợp đặc tính của nó (liên hệ file 23 —
State): working/conversation memory thường ở trong bộ nhớ tiến trình hoặc key-value store theo
session; summary/semantic/episodic memory phù hợp với vector store (cần semantic search) kết hợp
metadata store; entity memory phù hợp database quan hệ (theo khoá entity_id); artifact memory phù
hợp object storage (file lớn).

## 6. Luồng xử lý chi tiết

- **Working memory**: chứa plan, kết quả tool call, bước hiện tại, lỗi gặp phải — chỉ tồn tại
  trong phạm vi một lượt chạy task, mất đi khi task kết thúc (trừ khi được consolidate thành loại
  memory khác).
- **Conversation memory**: message qua lại giữa user và agent trong một phiên — là input trực
  tiếp cho model ở mỗi lượt gọi, cần cân bằng giữa giữ đủ ngữ cảnh và không vượt context window
  (liên hệ Compaction, mục 24.1, khi conversation quá dài).
- **Summary memory**: khi conversation/working memory quá dài, tóm tắt lại để giảm token trong khi
  vẫn giữ thông tin quan trọng — khác Compaction (kỹ thuật context engineering) ở chỗ summary
  memory có thể được **lưu lại lâu dài** như một loại memory riêng, không chỉ dùng tạm để nén context
  hiện tại.
- **Semantic memory**: fact và kiến thức tổng quát agent học được qua nhiều tương tác (không gắn
  với một sự kiện cụ thể) — ví dụ "user thường thích câu trả lời ngắn gọn" là một fact semantic,
  khác với "user đã hỏi câu X vào ngày Y" (đó là episodic).
- **Episodic memory**: lưu các sự kiện/task cụ thể đã xảy ra — gồm task, hành động đã thực hiện,
  kết quả, feedback nhận được, và "bài học" rút ra — hữu ích để agent tham khảo lại các tình huống
  tương tự trong quá khứ.
- **Procedural memory**: lưu cách làm/quy trình/kỹ năng có thể tái sử dụng (liên hệ Agent Skills,
  file 27, ở cấp độ chuẩn hoá cao hơn) — khác semantic memory ở chỗ đây là "biết làm" chứ không chỉ
  "biết sự thật".
- **Entity memory**: thông tin tổ chức theo một thực thể cụ thể (một khách hàng, một dự án) — cho
  phép agent nhớ ngữ cảnh riêng của từng entity qua nhiều lần tương tác khác nhau, không lẫn giữa
  các entity.
- **Artifact memory**: file, report, đoạn code, kết quả truy vấn mà agent đã tạo ra trong quá trình
  làm việc — cần lưu trữ (liên hệ Shared artifact workspace, file 18, ở cấp multi-agent) để tham
  chiếu lại hoặc để agent khác/con người dùng tiếp.
- **Memory consolidation**: quá trình tổng hợp nhiều memory ngắn hạn/rời rạc (ví dụ nhiều episodic
  memory tương tự nhau) thành một memory ổn định hơn, tổng quát hơn (có thể trở thành một semantic
  memory mới) — tương tự cách trí nhớ con người củng cố từ trải nghiệm lặp lại thành hiểu biết
  chung.
- **Memory forgetting**: chủ động xoá hoặc giảm trọng số memory theo tiêu chí: thời gian (memory
  cũ ít giá trị hơn), độ liên quan (không còn phù hợp ngữ cảnh hiện tại), chính sách riêng tư,
  yêu cầu xoá của user, quy định lưu trữ dữ liệu, hoặc khi có thông tin mới mâu thuẫn với memory cũ.

## 7. State và dữ liệu

Xem chi tiết ở file 23 (State) — memory là **nội dung**, state store là **hạ tầng lưu trữ** memory
đó. Mỗi loại memory nên có schema riêng phù hợp đặc tính (working memory: cấu trúc tạm thời; entity
memory: khoá theo entity_id; episodic: có timestamp bắt buộc).

## 8. Thuật toán liên quan

Semantic search (cho semantic/episodic memory dạng vector); không có thuật toán chuẩn cho memory
consolidation/forgetting — thường dựa vào LLM để tóm tắt/đánh giá độ liên quan, hoặc rule-based
theo thời gian/tần suất truy cập.

## 9. Cách triển khai

1. Phân loại rõ ràng loại memory nào cần cho use case cụ thể trước khi thiết kế hạ tầng — không
   phải mọi hệ thống cần đủ cả 8 loại; working + conversation memory là tối thiểu cho hầu hết
   agent.
2. Chọn hạ tầng lưu trữ phù hợp từng loại (không dùng một vector store chung cho tất cả — entity
   memory cần truy vấn chính xác theo khoá, không phải similarity search).
3. Thiết kế chính sách memory forgetting **ngay từ đầu**, không phải thêm sau khi phát hiện vấn đề
   dữ liệu — đặc biệt quan trọng cho tuân thủ quy định về quyền riêng tư/lưu trữ dữ liệu.
4. Cân nhắc memory consolidation định kỳ (batch job) cho episodic memory tích luỹ nhiều theo thời
   gian, tránh để agent phải quét qua toàn bộ lịch sử episodic thô mỗi lần cần tham khảo.

## 10. Tham số cần tuning

TTL cho từng loại memory (working: hết ngay sau task; conversation: hết phiên hoặc theo TTL cấu
hình; semantic/entity: dài hạn); ngưỡng độ dài trigger summary memory; tần suất chạy memory
consolidation; ngưỡng độ liên quan để giữ/xoá trong memory forgetting.

## 11. Failure modes

- **Nhầm working memory với memory dài hạn**: giữ thông tin chỉ có ý nghĩa trong 1 task như thể là
  kiến thức lâu dài, gây nhiễu cho các task sau.
- **Không consolidate, memory phình vô hạn**: episodic memory tích luỹ không giới hạn theo thời
  gian, làm chậm truy xuất và tăng chi phí lưu trữ không cần thiết.
- **Forgetting quá tay**: xoá memory quan trọng chỉ vì "cũ" mà không xét độ liên quan thực tế, khiến
  agent mất kiến thức hữu ích.
- **Trộn lẫn entity**: entity memory không tách bạch đúng theo entity_id, dẫn tới rò rỉ ngữ cảnh
  của entity này sang entity khác (ví dụ thông tin khách hàng A xuất hiện khi phục vụ khách hàng B).

## 12. Security considerations

Entity memory và episodic memory thường chứa dữ liệu cá nhân — cần áp dụng đúng chính sách quyền
riêng tư (liên hệ Memory và context poisoning, mục 29.10, cho khía cạnh nội dung độc hại; ở đây là
khía cạnh dữ liệu cá nhân hợp lệ nhưng cần bảo vệ) và tôn trọng yêu cầu xoá dữ liệu của user qua cơ
chế Memory forgetting.

## 13. Observability

Theo dõi kích thước từng loại memory theo thời gian (phát hiện sớm memory phình bất thường); log
các lần memory forgetting được kích hoạt (cái gì bị xoá, lý do gì) để có thể giải trình khi cần.

## 14. Evaluation metrics

Độ chính xác truy xuất semantic/episodic memory (tương tự Recall@K của RAG, file 31); tỷ lệ entity
memory bị trộn lẫn sai entity (kiểm tra qua test case cụ thể); hiệu quả memory consolidation (giảm
được bao nhiêu dung lượng/số lượng record mà vẫn giữ đủ thông tin hữu ích).

## 15. Ưu điểm

Phân loại đúng memory theo đặc tính giúp mỗi loại được lưu trữ/truy xuất/quản lý vòng đời một cách
tối ưu, tránh tình trạng "một kho lưu trữ chung" vừa chậm vừa khó quản lý.

## 16. Nhược điểm

Độ phức tạp hạ tầng tăng đáng kể so với "một vector store chung" — cần nhiều loại storage, nhiều
chính sách vòng đời khác nhau; chi phí thiết kế ban đầu cao hơn.

## 17. Khi nên dùng

Working + Conversation memory gần như luôn cần cho mọi agent. Các loại còn lại áp dụng theo nhu
cầu thực tế: agent phục vụ nhiều user/customer riêng biệt cần Entity memory; agent học hỏi qua thời
gian cần Episodic + Semantic + Consolidation; agent tạo sản phẩm cụ thể cần Artifact memory.

## 18. Khi không nên dùng

Agent xử lý task đơn lẻ, không có phiên làm việc kéo dài, không cần nhớ gì giữa các lần gọi — chỉ
cần Working memory trong phạm vi một lượt chạy, không cần đầu tư vào các loại memory dài hạn khác.

## 19. Pattern liên quan

Liên hệ trực tiếp State (file 23, hạ tầng lưu trữ) và Context Engineering (file 24, cách chọn phần
memory nào đưa vào context tại một thời điểm); Memory và context poisoning (mục 29.10) là rủi ro
bảo mật trực tiếp liên quan đến nhóm pattern này.

## 20. Ví dụ kiến trúc thực tế

Trợ lý chăm sóc khách hàng dài hạn: Working memory cho từng yêu cầu đang xử lý; Conversation memory
cho phiên chat hiện tại; Entity memory lưu hồ sơ từng khách hàng (lịch sử mua hàng, sở thích);
Episodic memory lưu các lần hỗ trợ trước đó của khách hàng đó kèm kết quả; Semantic memory tổng hợp
từ nhiều episodic (ví dụ "khách hàng này thường gặp vấn đề với tính năng X") qua Memory
consolidation định kỳ; chính sách Memory forgetting xoá Conversation memory sau 90 ngày theo quy
định lưu trữ dữ liệu, trong khi Entity memory được giữ lâu hơn theo thoả thuận dịch vụ.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về phân loại bộ nhớ (working/episodic/
semantic/procedural memory — mượn khái niệm từ khoa học nhận thức) áp dụng vào kiến trúc AI agent.
Nội dung là tổng hợp và diễn giải lại.*
