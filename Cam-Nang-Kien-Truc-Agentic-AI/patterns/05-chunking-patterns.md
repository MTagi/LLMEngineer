# 05. Chunking patterns mở rộng

[← Về mục lục chính](../README.md)

Nhóm lớn nhất trong Phần II: 12 chiến lược chia tài liệu đã parse (file 04) thành các đơn vị
(chunk) đủ nhỏ để embed hiệu quả, đủ lớn để giữ ngữ nghĩa.

## 1. Tên pattern

Fixed-token · Sentence-based · Recursive · Structure-aware · Semantic · Proposition ·
Parent-child · Sliding-window · Small-to-big · Contextual · Late chunking · Agentic chunking.

## 2. Vấn đề cần giải quyết

Chunk là đơn vị nhỏ nhất mà retrieval trả về và model nhìn thấy. Chunk quá nhỏ → mất ngữ cảnh
(model nhận được một câu rời rạc không hiểu "nó" là gì); chunk quá lớn → embedding bị pha loãng
(vector đại diện cho nhiều ý khác nhau, độ chính xác retrieval giảm) và đưa noise không liên quan
vào prompt (tốn token, dễ khiến model bỏ sót phần quan trọng). Không có một kích thước "đúng"
chung cho mọi tài liệu — 12 pattern dưới đây là các chiến lược khác nhau để cân bằng đánh đổi này
theo đặc tính từng loại nội dung.

## 3. Bối cảnh sử dụng

| Pattern | Phù hợp nhất khi |
|---|---|
| Fixed-token | Cần baseline nhanh, không có cấu trúc rõ ràng để dựa vào |
| Sentence-based | Nội dung dạng văn xuôi thông thường |
| Recursive | Tài liệu có phân cấp rõ (chương/mục/đoạn) |
| Structure-aware | Tài liệu có heading/clause/method/table boundary tường minh |
| Semantic | Nội dung chuyển chủ đề không theo cấu trúc hình thức |
| Proposition | Cần khớp chính xác với câu hỏi cụ thể, chấp nhận nhiều chunk hơn |
| Parent-child | Muốn vừa khớp chính xác vừa giữ đủ ngữ cảnh khi trả context |
| Sliding-window | Sợ mất thông tin ở ranh giới chunk |
| Small-to-big | Tương tự parent-child, nhấn vào việc retrieve đơn vị nhỏ rồi mở rộng |
| Contextual | Chunk khi tách rời mất hẳn ý nghĩa nếu không biết nó thuộc tài liệu/section nào |
| Late chunking | Cần chunk giữ được tín hiệu ngữ cảnh toàn tài liệu mà vẫn tách nhỏ được |
| Agentic chunking | Corpus đa dạng, không có một rule cố định nào đúng cho tất cả |

## 4. Kiến trúc

```
Fixed-token:        [.....512 token.....][.....512 token.....][...]  (cắt cơ học theo số token)

Recursive:           Chapter → Section → Paragraph → Sentence → Token
                      (thử chia ở cấp lớn nhất trước, chỉ xuống cấp nhỏ hơn nếu vẫn > giới hạn)

Sliding-window:      [chunk 1        ]
                            [overlap][chunk 2        ]
                                            [overlap][chunk 3        ]

Parent-child /
Small-to-big:         Parent (section, 2000 token)
                        ├─ Child 1 (câu/đoạn nhỏ, được embed & search)
                        ├─ Child 2 (được embed & search)
                        └─ Child 3 (được embed & search)
                       → search trên Child, trả context của Parent

Proposition:          "LangGraph supports checkpoints, which allow workflows to resume after
                       interruption and support human-in-the-loop execution."
                       →  P1: "LangGraph supports checkpoints."
                          P2: "Checkpoints allow workflows to resume after interruption."
                          P3: "Checkpoints support human-in-the-loop execution."

Contextual:           "Document: X | Section: Fault Tolerance > Checkpoint Recovery | Version: 3.2
                       <nội dung chunk gốc>"   (metadata mô tả gắn thẳng vào text trước khi embed)

Late chunking:        Toàn văn bản → embed ở cấp token với full context → SAU ĐÓ mới tách thành
                      các chunk (ngược thứ tự so với 11 pattern còn lại: chunk trước-embed sau)
```

## 5. Thành phần

Text splitter (rule-based cho fixed-token/sentence/recursive/structure-aware); embedding model
dùng để đo similarity cho semantic chunking; LLM cho proposition chunking (tách phát biểu) và
agentic chunking (quyết định ranh giới); cấu trúc dữ liệu 2 cấp (parent/child) cho
parent-child/small-to-big.

## 6. Luồng xử lý chi tiết

Điểm khác biệt cốt lõi giữa các pattern nằm ở **cái gì quyết định ranh giới chunk**:

- **Fixed-token/sentence-based/sliding-window**: ranh giới do quy tắc cơ học (số token, dấu câu,
  độ overlap cố định) — nhanh, dễ đoán, không cần hiểu nội dung.
- **Recursive/structure-aware**: ranh giới do **cấu trúc tài liệu** quyết định (đầu ra của file
  04) — ưu tiên cắt ở ranh giới heading/section/method trước khi phải cắt cơ học bên trong.
- **Semantic**: ranh giới do **độ tương đồng embedding** giữa các câu liền kề quyết định — cắt khi
  similarity giảm dưới ngưỡng, tức là khi nội dung "đổi chủ đề".
- **Proposition/agentic**: ranh giới do **LLM quyết định** dựa trên loại tài liệu, chủ đề, tính
  hoàn chỉnh về ngữ nghĩa — linh hoạt nhất, cũng tốn chi phí nhất (mỗi tài liệu cần 1+ lệnh gọi
  LLM lúc indexing).
- **Parent-child/small-to-big**: **tách 2 giai đoạn** — chunk nhỏ (child) dùng để search vì khớp
  chính xác hơn, nhưng khi trả context cho model thì mở rộng lên chunk lớn hơn (parent) để không
  mất ngữ cảnh xung quanh.
- **Contextual**: **thêm thông tin** vào chunk trước khi embed (title, section path, summary) —
  không đổi ranh giới cắt, chỉ làm giàu nội dung được embed để tăng độ chính xác retrieval, đặc
  biệt hữu ích khi chunk tách rời sẽ vô nghĩa.
- **Late chunking**: **đảo ngược thứ tự** so với tất cả pattern khác — embed toàn văn bản (hoặc
  đoạn dài) với đầy đủ ngữ cảnh trước, rồi mới tách representation thành các chunk nhỏ; nhờ vậy mỗi
  chunk "biết" về phần còn lại của tài liệu dù bản thân nó ngắn.

## 7. State và dữ liệu

Mỗi chunk cần lưu: nội dung, chunk_hash (liên hệ mục incremental indexing ở file 03), vị trí
trong tài liệu gốc (để mở rộng ngược về parent hoặc trích dẫn), và với parent-child cần lưu quan
hệ `child_id → parent_id` tường minh.

## 8. Thuật toán liên quan

Cosine similarity giữa embedding câu liền kề (semantic chunking); recursive splitting theo danh
sách separator ưu tiên giảm dần (đoạn văn → câu → từ); không có thuật toán chuẩn hoá cho
proposition/agentic chunking — phụ thuộc chất lượng prompt điều khiển LLM.

## 9. Cách triển khai

1. Bắt đầu với **structure-aware/recursive** nếu tài liệu có cấu trúc rõ (heading, section) — chi
   phí thấp, hiệu quả cao, nên là baseline đầu tiên trước khi thử pattern phức tạp hơn.
2. Thêm **overlap** (sliding-window) mặc định 15-20% kích thước chunk để giảm rủi ro cắt mất ý ở
   ranh giới, bất kể dùng chiến lược chia nào làm nền.
3. Thêm **contextual chunking** (gắn title/section path) gần như miễn phí về kỹ thuật, lợi ích
   cao — nên áp dụng mặc định trừ khi chunk vốn đã đủ ngữ cảnh tự thân.
4. Chỉ chuyển sang **parent-child/small-to-big** khi baseline cho thấy retrieval chính xác nhưng
   context trả về thiếu — đây là dấu hiệu cần tách search-unit khỏi context-unit.
5. Dùng **proposition/agentic/late chunking** khi đã đánh giá (file 31) và xác định baseline
   không đủ tốt cho use case cụ thể — đây là các pattern tốn chi phí nhất, không nên là lựa chọn
   mặc định.

## 10. Tham số cần tuning

Kích thước chunk (điểm khởi đầu thực dụng: 500 ký tự/token cho hầu hết use case, điều chỉnh theo
kết quả đánh giá thực tế); tỷ lệ overlap (~20% kích thước chunk); ngưỡng similarity cho semantic
chunking; giới hạn kích thước parent trong parent-child.

## 11. Failure modes

- **Cắt giữa ý**: fixed-token thuần không quan tâm ranh giới ngữ nghĩa, dễ cắt ngang câu/ý quan
  trọng.
- **Chunk quá nhỏ mất chủ ngữ**: sentence-based/proposition tách quá nhỏ khiến đại từ ("nó", "hệ
  thống này") mất tham chiếu — cần contextual chunking hoặc parent-child để bù.
- **Bùng nổ số lượng chunk**: proposition chunking làm tăng mạnh số chunk cần lưu/search, tăng chi
  phí index và có thể làm chậm truy vấn nếu không tối ưu index.
- **Semantic chunking không ổn định**: ngưỡng similarity nhạy với nội dung, cùng một tham số có
  thể hoạt động tốt với tài liệu này nhưng tệ với tài liệu khác — cần đánh giá theo loại tài liệu,
  không dùng một ngưỡng chung cho toàn corpus không đồng nhất.

## 12. Security considerations

Contextual chunking nhúng metadata (title, section) vào nội dung được embed — cần đảm bảo
metadata đó không vô tình lộ thông tin nhạy cảm (ví dụ tên khách hàng trong section path) ra
ngoài phạm vi ACL của chunk gốc.

## 13. Observability

Theo dõi phân phối kích thước chunk thực tế (không chỉ tham số cấu hình — nhiều bộ chia thực tế
tạo ra chunk lệch xa kích thước mục tiêu); theo dõi tỷ lệ chunk "mồ côi" (retrieve được nhưng
không đủ ngữ cảnh để model dùng, phải bỏ qua ở bước generation).

## 14. Evaluation metrics

Recall@K khi thay đổi chiến lược chunking (file 31) là thước đo quyết định — không có metric nội
tại "chunking tốt" nào tách rời khỏi hiệu năng retrieval cuối cùng; nên A/B test từng chiến lược
trên cùng bộ câu hỏi vàng.

## 15. Ưu điểm

Chọn đúng chiến lược chunking theo đặc tính tài liệu cải thiện trực tiếp và mạnh nhất tới chất
lượng RAG — thường tác động lớn hơn cả việc đổi embedding model.

## 16. Nhược điểm

Không có công thức phổ quát; nhiều pattern (proposition, agentic, late chunking) đòi hỏi thêm chi
phí tính toán/LLM ở thời điểm indexing, tăng đáng kể thời gian và chi phí xây index cho corpus
lớn.

## 17. Khi nên dùng

Structure-aware/recursive + contextual + overlap nên là bộ ba mặc định cho hầu hết hệ thống RAG
production. Các pattern còn lại áp dụng có chọn lọc sau khi đánh giá cho thấy cần thiết.

## 18. Khi không nên dùng

Với tài liệu cực ngắn (đã nhỏ hơn giới hạn cần chunk) — chunking không cần thiết, embed nguyên
tài liệu.

## 19. Pattern liên quan

Nhận input từ Parsing (file 04); output là input trực tiếp cho Retrieval (file 06) và Vector index
(file 08). Parent-child chunking liên hệ chặt với Hierarchical retrieval (mục 6.6 ở file 06).

## 20. Ví dụ kiến trúc thực tế

Hệ thống RAG cho tài liệu kỹ thuật sản phẩm: dùng structure-aware chunking theo heading (mỗi mục
hướng dẫn là 1 đơn vị chunking cơ sở), thêm contextual chunking gắn tên sản phẩm + phiên bản +
đường dẫn section vào mỗi chunk trước khi embed, và dùng parent-child cho các mục dài — search
trên đoạn nhỏ nhưng trả về cả mục hướng dẫn đầy đủ làm context cho model.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung và các bài viết kỹ thuật phổ biến về chiến
lược chunking trong RAG (bao gồm khái niệm late chunking, proposition-based chunking). Nội dung
là tổng hợp và diễn giải lại, không trích dẫn nguyên văn.*
