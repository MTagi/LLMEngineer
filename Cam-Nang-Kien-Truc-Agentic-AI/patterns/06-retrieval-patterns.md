# 06. Retrieval patterns mở rộng

[← Về mục lục chính](../README.md)

11 chiến lược **tìm kiếm** trong index đã xây (file 08) để trả về candidate cho bước fusion/
reranking (file 09): Dense, Sparse, Hybrid, Multi-index, Multi-vector, Hierarchical, Federated,
Multi-hop, Adaptive, Negative, Late-interaction.

## 1. Tên pattern

Dense · Sparse · Hybrid · Multi-index · Multi-vector · Hierarchical · Federated · Multi-hop ·
Adaptive · Negative · Late-interaction (ColBERT-style) retrieval.

## 2. Vấn đề cần giải quyết

Không một chiến lược retrieval đơn lẻ nào thắng trong mọi trường hợp: dense retrieval bỏ lỡ khớp
từ khoá chính xác (mã lỗi, tên riêng); sparse bỏ lỡ khớp theo ý nghĩa (từ đồng nghĩa, diễn đạt
khác); một index duy nhất không tối ưu cho mọi loại câu hỏi (câu hỏi cần tên tài liệu khác câu
hỏi cần nội dung chi tiết); câu hỏi cần kết hợp thông tin từ nhiều tài liệu không thể trả lời bằng
một lượt search; và không phải câu hỏi nào cũng cần retrieval. 11 pattern này là các cách kết hợp/
chọn lọc chiến lược search theo đặc tính câu hỏi và corpus.

## 3. Bối cảnh sử dụng

| Pattern | Dùng khi |
|---|---|
| Dense | Câu hỏi diễn đạt khác chữ nhưng cùng ý với tài liệu |
| Sparse (BM25) | Câu hỏi có từ khoá/mã định danh chính xác |
| Hybrid | Mặc định cho hầu hết use case production (kết hợp cả hai) |
| Multi-index | Corpus cần trả lời cả câu hỏi tra tên lẫn câu hỏi nội dung |
| Multi-vector | Một tài liệu có nhiều "mặt" ý nghĩa cần biểu diễn riêng |
| Hierarchical | Corpus lớn, có tổ chức tự nhiên theo sản phẩm/dự án/domain |
| Federated | Dữ liệu nằm ở nhiều search service độc lập theo domain |
| Multi-hop | Câu hỏi cần kết hợp thông tin từ ≥ 2 tài liệu liên quan |
| Adaptive | Muốn hệ thống tự quyết định chiến lược thay vì cố định |
| Negative | Cần chủ động phát hiện mâu thuẫn/phiên bản cũ/ngoại lệ |
| Late-interaction | Cần độ chính xác cao hơn dense đơn thuần, chấp nhận chi phí cao hơn |

## 4. Kiến trúc

```
Hybrid:            BM25 Results ─────┐
                                     ├─ Fusion → Rerank
                   Vector Results ───┘

Multi-index:        Query → {Chunk Index, Summary Index, Title Index, Entity Index,
                    Question Index}  (search song song nhiều index, hợp kết quả)

Hierarchical:        Corpus → Relevant Collection → Relevant Document → Relevant Section →
                    Relevant Chunk   (thu hẹp dần, giảm search space)

Federated:           Enterprise Query → {Engineering Search, HR Search, Product Search,
                    External Search}  (mỗi domain 1 search service riêng, policy riêng)

Multi-hop:           Question → Retrieve Entity A → Extract Relationship → Retrieve Entity B →
                    Compose Evidence  (kết quả bước trước tạo query cho bước sau)

Adaptive:            Query → {cần retrieval? dùng gì (BM25/vector/graph)? top-K bao nhiêu?
                    cần rerank? cần search lại?}  → quyết định động, không cố định
```

## 5. Thành phần

Retriever cho từng loại index (BM25 retriever, vector retriever, graph retriever); router/policy
quyết định chọn retriever nào (multi-index, federated, adaptive); logic thu hẹp dần theo cấp bậc
(hierarchical); vòng lặp truy vấn-trích xuất-truy vấn tiếp (multi-hop).

## 6. Luồng xử lý chi tiết

- **Dense vs Sparse**: dense dùng cosine similarity giữa embedding query và embedding chunk; sparse
  (BM25, chi tiết ở file 07) dùng thống kê tần suất từ. Hybrid chạy **song song cả hai** rồi hợp
  nhất kết quả qua fusion (RRF/weighted, file 09) — không phải chọn một trong hai mà kết hợp.
- **Multi-index**: cùng một tài liệu được biểu diễn qua nhiều index chuyên biệt (nội dung chunk,
  tóm tắt, tiêu đề, thực thể, câu hỏi mẫu) — mỗi index tối ưu cho một loại truy vấn khác nhau, kết
  quả từ các index được hợp lại trước khi rerank.
- **Multi-vector**: khác multi-index ở chỗ đây là **nhiều vector cho cùng một document/chunk**
  (content embedding, summary embedding, title embedding...) thay vì nhiều index riêng biệt — phù
  hợp khi muốn một chunk "trả lời được" cho nhiều dạng câu hỏi khác nhau về chính nó.
- **Hierarchical**: search theo cấp bậc giảm dần (corpus → collection → document → section →
  chunk), mỗi cấp thu hẹp search space cho cấp sau — giảm chi phí tính toán và tăng độ chính xác
  khi corpus có tổ chức tự nhiên (ví dụ theo sản phẩm/phòng ban).
- **Federated**: mỗi domain (Engineering, HR, Product...) vận hành search service độc lập, có thể
  dùng embedding model/index/policy khác nhau; query gửi song song tới các service liên quan rồi
  tổng hợp — khác hierarchical ở chỗ các nhánh **độc lập về hạ tầng**, không chỉ là thu hẹp phạm
  vi trong cùng một index.
- **Multi-hop**: kết quả của lượt retrieve đầu (ví dụ tìm ra Entity A) được dùng để **tạo truy vấn
  mới** cho lượt tiếp theo (tìm quan hệ của A, từ đó tìm Entity B) — cần thiết khi câu trả lời
  không nằm trọn trong một tài liệu mà phải ghép từ nhiều nguồn.
- **Adaptive**: một lớp quyết định (thường là LLM hoặc classifier nhẹ) đứng trước toàn bộ hệ
  thống retrieval, quyết định có cần search không, dùng retriever nào, top-K bao nhiêu, có cần
  rerank/search lại không — đây là nền tảng kỹ thuật của Agentic RAG.
- **Negative retrieval**: song song với việc tìm bằng chứng ủng hộ, chủ động chạy thêm truy vấn
  tìm tài liệu mâu thuẫn, phiên bản cũ, điều khoản ngoại lệ — để hệ thống trả lời cân bằng, không
  chỉ dựa vào bằng chứng thuận chiều đầu tiên tìm được.
- **Late-interaction**: xem chi tiết thuật toán ở mục 6.11 trong tài liệu tổng quan — giữ embedding
  cấp token, so khớp token-với-token tại thời điểm truy vấn thay vì nén thành một vector duy nhất.

## 7. State và dữ liệu

Retrieval về cơ bản là stateless per-query, trừ multi-hop (cần giữ trạng thái các entity/quan hệ
đã tìm được qua các hop) và adaptive (có thể cần lịch sử quyết định trong cùng phiên để tránh lặp
lại truy vấn giống hệt).

## 8. Thuật toán liên quan

Cosine similarity (dense); BM25 (sparse, file 07); RRF/weighted fusion (file 09); MaxSim
(late-interaction); không có thuật toán chuẩn cho việc "quyết định chiến lược" ở adaptive/negative
— đây là logic điều khiển do LLM hoặc rule-based policy quyết định.

## 9. Cách triển khai

1. Bắt đầu bằng **Hybrid** (dense + sparse + fusion) làm baseline production — đơn giản hơn
   nhiều pattern khác nhưng thường đã đủ tốt cho phần lớn use case.
2. Thêm **Multi-index** nếu đánh giá cho thấy một loại câu hỏi cụ thể (ví dụ tra theo tên tài
   liệu) hoạt động kém với index chunk-only.
3. Thêm **Hierarchical/Federated** khi corpus đủ lớn để search toàn bộ trở nên chậm hoặc tốn kém,
   hoặc khi dữ liệu vốn đã nằm ở nhiều hệ thống tách biệt theo domain.
4. Thêm **Multi-hop** chỉ khi có bằng chứng từ đánh giá rằng nhiều câu hỏi thực tế cần kết hợp
   thông tin từ nhiều tài liệu — đây là pattern phức tạp, không nên dùng mặc định.
5. Chuyển sang **Adaptive** khi đã có agent framework (Phần III) — về bản chất đây là một quyết
   định trong vòng lặp Reasoning/Planning (file 10), không phải một retriever riêng biệt.

## 10. Tham số cần tuning

Top-K cho mỗi retriever trước khi fusion; trọng số giữa dense/sparse trong hybrid; số cấp trong
hierarchical; số hop tối đa trong multi-hop (cần termination guard, liên hệ mục 30.9); ngưỡng
quyết định "có cần retrieval" trong adaptive.

## 11. Failure modes

- **Hybrid không thực sự hybrid**: chỉ chạy dense rồi lọc theo từ khoá hậu kiểm, không phải chạy
  song song hai retriever thật sự — mất lợi ích của cả hai.
- **Multi-hop lặp vô hạn**: không giới hạn số hop, agent cứ tiếp tục "tìm thêm" mà không có tiêu
  chí dừng rõ ràng.
- **Federated timeout theo service chậm nhất**: nếu không có timeout/fallback riêng cho từng
  domain search, một service chậm/lỗi kéo chậm toàn bộ truy vấn.
- **Adaptive quyết định sai do prompt chọn retriever mơ hồ**: nếu logic/prompt quyết định "cần
  retrieval hay không" không rõ ràng, hệ thống có thể bỏ sót retrieval cần thiết (trả lời sai từ
  kiến thức chung) hoặc retrieval thừa (chi phí không cần thiết).

## 12. Security considerations

Federated retrieval đặc biệt cần chú ý: mỗi domain search có thể có policy quyền truy cập khác
nhau — khi tổng hợp kết quả từ nhiều service, không được để kết quả từ domain user không có quyền
lọt vào response cuối (liên hệ Retrieval guardrail, mục 29.5).

## 13. Observability

Log rõ **retriever nào được chọn** cho mỗi query (đặc biệt với adaptive/multi-index/federated) và
kết quả từng retriever trước khi fusion — nếu không tách log theo retriever, không thể debug khi
kết quả cuối kém (không biết lỗi từ retriever nào).

## 14. Evaluation metrics

Recall@K/Precision@K/MRR/NDCG (file 31) đo riêng cho từng chiến lược retrieval trên cùng bộ query
test, để biết chiến lược nào thực sự cải thiện so với baseline dense/sparse đơn thuần.

## 15. Ưu điểm

Cho phép hệ thống retrieval thích nghi với đặc tính đa dạng của câu hỏi và corpus thay vì ép mọi
truy vấn qua một con đường cứng nhắc — cải thiện trực tiếp độ chính xác và độ liên quan của bằng
chứng đưa vào generation.

## 16. Nhược điểm

Mỗi pattern bổ sung (ngoài Hybrid cơ bản) đều tăng độ phức tạp hạ tầng và chi phí vận hành; cần
đánh giá kỹ trước khi thêm — không phải corpus nào cũng cần multi-hop hay federated.

## 17. Khi nên dùng

Hybrid: gần như luôn nên dùng làm nền. Các pattern còn lại: áp dụng có chọn lọc dựa trên đặc tính
thực tế của corpus và kết quả đánh giá, không thêm "vì nghe có vẻ mạnh hơn".

## 18. Khi không nên dùng

Với corpus rất nhỏ và đồng nhất (vài chục tài liệu, một domain duy nhất) — dense retrieval đơn
giản đã đủ, thêm các pattern phức tạp chỉ tăng chi phí mà không cải thiện đáng kể.

## 19. Pattern liên quan

Nhận input từ Chunking (file 05) và Vector/Lexical index (file 07, 08); output đi vào Fusion/
Reranking (file 09). Adaptive retrieval là nền tảng kỹ thuật trực tiếp của Agentic RAG (liên hệ
file 02, mục 2.3) và của Reasoning/Planning patterns (file 10).

## 20. Ví dụ kiến trúc thực tế

Trợ lý hỗ trợ kỹ thuật đa sản phẩm: hybrid retrieval làm nền cho mọi câu hỏi; hierarchical thu hẹp
theo sản phẩm trước khi search chi tiết; multi-hop kích hoạt khi câu hỏi dạng "lỗi X có liên quan
gì đến cấu hình Y không" (cần nối thông tin từ tài liệu lỗi và tài liệu cấu hình); toàn bộ được
điều phối bởi một lớp adaptive quyết định có cần search không trước khi chạy pipeline.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về information retrieval và các kỹ thuật RAG
phổ biến (hybrid search, RAG-Fusion, multi-hop QA, ColBERT late-interaction). Nội dung là tổng hợp
và diễn giải lại.*
