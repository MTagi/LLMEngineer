# 02. Kiến trúc tổng thể của RAG

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Kiến trúc RAG hai pipeline: **Indexing pipeline** và **Query pipeline** — khung tổng thể mà mọi
pattern chi tiết ở các file 03-09 đều là một khối trong đó.

## 2. Vấn đề cần giải quyết

Nhầm lẫn phổ biến nhất khi mới làm RAG là coi nó như "tạo vector rồi tìm top-K" — một hàm đơn.
Thực tế RAG production là **hai hệ thống vận hành độc lập, tốc độ khác nhau, thất bại khác nhau**:
một hệ thống chuẩn bị dữ liệu chạy theo lịch/sự kiện (indexing), và một hệ thống phục vụ truy vấn
chạy realtime theo mỗi request của user (query). Nếu thiết kế gộp chung, việc reindex một
document sẽ chặn luôn cả query đang chạy, hoặc ngược lại một query nặng sẽ làm chậm quá trình
ingest dữ liệu mới.

## 3. Bối cảnh sử dụng

Bất kỳ hệ thống nào cần LLM trả lời dựa trên một tập tài liệu vượt quá khả năng nhét vừa vào một
prompt — tài liệu nội bộ công ty, tài liệu kỹ thuật, hồ sơ pháp lý, codebase, v.v.

## 4. Kiến trúc

```
┌─────────────────────────── INDEXING PIPELINE (offline/batch/event) ───────────────────────────┐
│ Data Sources → Ingestion → Parsing/OCR/Layout → Cleaning/Normalization → Document               │
│ Classification → Chunking → Metadata & ACL Enrichment → Embedding → Lexical + Vector + Graph    │
│ Index                                                                                            │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────── QUERY PIPELINE (online/realtime) ──────────────────────────────────┐
│ User Query → AuthN/AuthZ → Intent & Query Analysis → Query Transformation → Retriever           │
│ Selection → Candidate Retrieval → Fusion & Reranking → Context Assembly → Answer Generation →   │
│ Groundedness & Citation Validation                                                              │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

Hai pipeline chỉ chạm nhau ở đúng một điểm: **index** (lexical + vector + graph) — indexing pipeline
ghi vào, query pipeline đọc ra. Tách bạch điểm chạm này là nguyên tắc thiết kế quan trọng nhất.

## 5. Thành phần

| Thành phần | Thuộc pipeline | Chi tiết ở file |
|---|---|---|
| Ingestion (batch/event/CDC) | Indexing | 03 |
| Parsing (layout/table/multimodal/code) | Indexing | 04 |
| Chunking | Indexing | 05 |
| Embedding model | Indexing + Query (dùng chung model) | 06, 08 |
| Lexical index (BM25) | Indexing (ghi) / Query (đọc) | 07 |
| Vector index/database | Indexing (ghi) / Query (đọc) | 08 |
| Query transformation, retriever selection | Query | 06 |
| Fusion, reranking, context assembly | Query | 09 |
| Generation model + citation validation | Query | — (nằm ngoài phạm vi RAG thuần, thuộc model layer) |

## 6. Luồng xử lý chi tiết

**Indexing pipeline** — chạy độc lập với user request, theo lịch (mục 3.1), theo sự kiện thay đổi
dữ liệu (mục 3.2), hoặc theo CDC từ database (mục 3.3). Mỗi tài liệu đi qua toàn bộ chuỗi parse →
clean → classify → chunk → enrich metadata/ACL → embed → ghi vào 1-3 loại index tuỳ kiến trúc
(lexical, vector, graph).

**Query pipeline** — chạy mỗi lần user hỏi. Bước quan trọng dễ bị bỏ qua nhất là **Intent and
Query Analysis** trước khi retrieve: không phải câu hỏi nào cũng cần retrieval (liên hệ Agentic
RAG, mục 6.3 trong file 06), và **Groundedness/Citation Validation** sau khi generate: kiểm tra
câu trả lời có thực sự bám vào context đã retrieve hay model đang "bịa" — bước này thường bị bỏ
qua trong bản demo nhưng bắt buộc phải có ở production.

## 7. State và dữ liệu

- Indexing pipeline có state riêng: `document_hash`, `chunk_hash` (mục 3.4), version/effective
  date (mục 3.5) — cần lưu độc lập với index để biết cái gì đã xử lý, cái gì cần xử lý lại.
- Query pipeline about lý tưởng là **stateless** giữa các request (mỗi query độc lập) trừ khi kết
  hợp với agent có memory/state riêng (Phần V) — không nên để logic retrieval phụ thuộc ẩn vào
  state của request trước.

## 8. Thuật toán liên quan

Không có thuật toán riêng ở tầng kiến trúc tổng thể — thuật toán cụ thể nằm ở từng khối (BM25 ở
file 07, HNSW/IVF/PQ ở file 08, RRF/cross-encoder ở file 09).

## 9. Cách triển khai

1. Xây indexing pipeline trước, độc lập, có thể chạy/test mà không cần query pipeline tồn tại
   (input: tài liệu thô, output: index đã điền).
2. Xây query pipeline đọc từ index đã có sẵn (có thể dùng index test nhỏ trong lúc phát triển
   song song).
3. Định nghĩa rõ **hợp đồng dữ liệu** giữa hai pipeline: schema của record trong index (chunk
   text, embedding, metadata, ACL, version) — đây là API nội bộ quan trọng nhất của toàn hệ
   thống RAG.
4. Thêm health check độc lập cho từng pipeline (indexing pipeline có thể "khoẻ" dù query pipeline
   đang lỗi, và ngược lại).

## 10. Tham số cần tuning

- Tần suất chạy indexing pipeline (batch schedule, độ trễ event-driven).
- Timeout của từng bước trong query pipeline (retrieval, rerank, generation) — tổng phải nằm
  trong ngân sách latency chấp nhận được cho user.
- Ngưỡng đồng bộ: index có thể "cũ" bao lâu so với nguồn dữ liệu gốc trước khi coi là vấn đề.

## 11. Failure modes

- **Index lệch pha**: indexing pipeline lỗi âm thầm (một batch job fail nhưng không alert) khiến
  query pipeline trả kết quả dựa trên dữ liệu cũ mà không ai biết.
- **Coupling ngầm**: code viết chung một hàm cho cả indexing và query khiến thay đổi một bên vô
  tình phá bên kia (ví dụ đổi cách chunk ảnh hưởng cả tốc độ ingest lẫn định dạng context đưa vào
  prompt).
- **Bỏ qua bước validate cuối**: generate câu trả lời rồi trả thẳng cho user mà không qua bước
  Groundedness/Citation Validation — model có thể trả lời đúng cấu trúc nhưng sai nội dung so với
  tài liệu nguồn.

## 12. Security considerations

ACL và tenant phải được gắn từ **indexing pipeline** (mục Metadata and ACL Enrichment) và được
**query pipeline** kiểm tra lại ở bước AuthN/AuthZ lẫn Retrieval guardrail (mục 29.5) — không chỉ
lọc kết quả sau khi đã retrieve, vì bản thân bước retrieval đã có thể rò rỉ thông tin qua
similarity score hoặc qua log.

## 13. Observability

Trace hai pipeline **tách biệt** (liên hệ mục 34): indexing pipeline cần log theo document
(thành công/thất bại/thời gian xử lý mỗi tài liệu); query pipeline cần trace theo request, tách
riêng span retrieval khỏi span generation để biết lỗi/chậm nằm ở đâu.

## 14. Evaluation metrics

RAG cần đánh giá tách theo từng khối trong kiến trúc, không chỉ theo câu trả lời cuối cùng: chất
lượng retrieval (Recall@K, Precision@K — file 31), chất lượng reranking, chất lượng context
assembly (có đủ, có dư, có mâu thuẫn không), và chất lượng generation cuối (faithfulness, citation
correctness). Nếu chỉ đo output cuối, không biết lỗi đến từ chunking, retriever hay model.

## 15. Ưu điểm

Tách bạch hai pipeline cho phép scale, deploy và debug độc lập; cho phép đổi công nghệ một bên
(ví dụ đổi vector database) mà không ảnh hưởng bên kia miễn giữ nguyên hợp đồng dữ liệu.

## 16. Nhược điểm

Phức tạp vận hành hơn một script "embed rồi search" đơn giản — cần quản lý đồng bộ, versioning
index, và health check riêng cho từng pipeline. Với dữ liệu rất nhỏ/tĩnh, chi phí này có thể
không đáng.

## 17. Khi nên dùng

Bất kỳ hệ thống RAG nào dự kiến chạy production, có nhiều hơn một người dùng, hoặc dữ liệu nguồn
thay đổi theo thời gian.

## 18. Khi không nên dùng

Với prototype nhỏ, dữ liệu tĩnh, một lần dùng — một script đơn giản "load → embed → search" là đủ,
tách hai pipeline lúc này là over-engineering.

## 19. Pattern liên quan

Toàn bộ file 03-09 là chi tiết của các khối trong kiến trúc này. Liên hệ trực tiếp với Agentic RAG
(mục 6.3 ở file 06) — quyết định "có cần chạy Query Pipeline hay không" chính là quyết định đầu
tiên mà một agent RAG phải đưa ra.

## 20. Ví dụ kiến trúc thực tế

Hệ thống hỏi-đáp tài liệu nội bộ doanh nghiệp: indexing pipeline chạy theo CDC mỗi khi tài liệu
trên hệ thống quản lý tài liệu (SharePoint/Confluence...) được cập nhật; query pipeline phục vụ
qua API cho chatbot nội bộ, retrieval hybrid (BM25 + vector) kết hợp reranking, và mọi câu trả lời
đều kèm trích dẫn nguồn tài liệu + kiểm tra groundedness trước khi trả về.

---

*Nguồn tham khảo dùng khi biên soạn: mô tả kỹ thuật của Microsoft về pipeline chuẩn bị dữ liệu,
chunking, enrichment, embedding, index configuration và retrieval trong RAG production. Nội dung
là tổng hợp và diễn giải lại.*
