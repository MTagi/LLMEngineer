# 09. Fusion, reranking và context construction

[← Về mục lục chính](../README.md)

7 pattern xử lý **sau khi đã có candidate thô** từ nhiều retriever (file 06-08) cho tới khi có
context sẵn sàng đưa vào prompt: RRF, Weighted fusion, Cross-encoder reranking, LLM reranking,
Diversity reranking, Context compression, Evidence packing.

## 1. Tên pattern

Reciprocal Rank Fusion · Weighted score fusion · Cross-encoder reranking · LLM reranking ·
Diversity reranking (MMR) · Context compression · Evidence packing.

## 2. Vấn đề cần giải quyết

Sau khi Hybrid retrieval (file 06) trả về hai (hoặc nhiều) danh sách kết quả riêng biệt từ các
retriever khác nhau, cần một cách hợp nhất công bằng — điểm BM25 và điểm cosine similarity không
cùng thang đo, cộng trực tiếp là sai về mặt thống kê. Sau khi hợp nhất, top-K thô vẫn có thể chứa
nhiễu, trùng lặp, hoặc thiếu đa dạng. Và ngay cả khi đã có candidate tốt, việc ghép chúng thành
context đưa vào prompt (bao nhiêu token, thứ tự nào, có mâu thuẫn không) là một bước riêng, dễ bị
bỏ qua trong bản demo nhưng quyết định trực tiếp chất lượng câu trả lời cuối.

## 3. Bối cảnh sử dụng

RRF/Weighted fusion: bắt buộc khi dùng Hybrid retrieval. Cross-encoder/LLM reranking: khi top-K
thô từ retrieval còn lẫn nhiều kết quả không thực sự liên quan. Diversity reranking: khi top-K có
xu hướng trùng lặp nội dung gần giống nhau. Context compression/Evidence packing: luôn cần trước
khi gọi model generation.

## 4. Kiến trúc

```
RRF:                RRF(d) = Σ 1 / (k + rank_i(d))   — cộng nghịch đảo THỨ HẠNG, không cộng điểm
                    thô, nên không cần chuẩn hoá thang đo giữa các retriever.

Weighted fusion:    final_score = w1 × normalized_bm25 + w2 × normalized_vector +
                    w3 × metadata_score   — cần chuẩn hoá điểm về cùng thang trước khi cộng.

Pipeline đầy đủ:    BM25 top-K ─┐
                                ├→ Fusion (RRF/Weighted) → Reranking (Cross-encoder/LLM) →
                    Vector top-K┘   Diversity filter (MMR) → Context compression →
                                    Evidence packing → Context sẵn sàng cho prompt
```

## 5. Thành phần

Fusion function (RRF hoặc weighted); cross-encoder model (đọc đồng thời query+document, khác dense
retrieval chỉ đọc riêng từng bên rồi so sánh vector); LLM dùng làm reranker (đắt hơn cross-encoder
nhưng linh hoạt hơn về tiêu chí); MMR (Maximal Marginal Relevance) cân bằng relevance/diversity;
context builder (logic dedupe, group, expand, token budget).

## 6. Luồng xử lý chi tiết

- **RRF**: mỗi retriever tạo một danh sách xếp hạng riêng; RRF cộng nghịch đảo **thứ hạng** (không
  phải điểm thô) của mỗi document qua các danh sách, với hằng số `k` làm giảm ảnh hưởng của các vị
  trí thấp — ưu điểm là không cần chuẩn hoá thang điểm giữa BM25 và cosine similarity (vốn khác
  nhau về đơn vị và phân phối).
- **Weighted fusion**: chuẩn hoá điểm từng retriever về cùng khoảng (ví dụ 0-1) rồi cộng có trọng
  số — cho phép kiểm soát tường minh mức độ ưu tiên retriever nào (ví dụ ưu tiên BM25 hơn khi
  corpus nhiều thuật ngữ kỹ thuật), nhưng nhạy cảm hơn với cách chuẩn hoá và cần tinh chỉnh trọng
  số.
- **Cross-encoder reranking**: khác dense retrieval (embed query và document riêng rồi so sánh
  vector — "bi-encoder"), cross-encoder đưa **cả query và document vào cùng một lần forward pass**
  của model, cho phép model học được tương tác sâu giữa hai bên — chính xác hơn nhưng chậm hơn
  nhiều, nên chỉ áp dụng cho top-N nhỏ sau khi đã lọc thô bằng retrieval.
- **LLM reranking**: dùng LLM đánh giá từng candidate theo tiêu chí ngữ nghĩa phức tạp hơn
  similarity thuần — chunk có trả lời trực tiếp câu hỏi không, có đúng phiên bản/sản phẩm không,
  có phải nguồn chính thức không, có chứa ngoại lệ quan trọng không. Linh hoạt nhất nhưng chi phí
  cao nhất (một lệnh gọi LLM cho mỗi candidate, hoặc một lệnh gọi đánh giá cả danh sách).
- **Diversity reranking (MMR)**: sau khi có danh sách xếp hạng theo relevance, MMR chọn dần từng
  kết quả sao cho vừa liên quan tới query vừa **khác biệt** với các kết quả đã chọn trước đó —
  tránh top-K bị chiếm hết bởi các chunk gần như trùng lặp nội dung.
- **Context compression**: trích riêng phần liên quan từ một chunk lớn (thay vì đưa nguyên chunk)
  trước khi đưa vào prompt — giảm token, giảm nhiễu, nhưng cần cẩn thận không cắt mất ngữ cảnh cần
  thiết để hiểu phần được trích.
- **Evidence packing**: bước cuối cùng lắp ráp context thực tế đưa vào prompt — deduplicate (loại
  candidate trùng/gần trùng), group theo document (giữ các chunk cùng nguồn gần nhau để model dễ
  theo mạch), mở rộng sang chunk lân cận nếu cần thêm ngữ cảnh, giữ source ID cho citation, ưu tiên
  phiên bản hiện hành (liên hệ Temporal indexing, mục 3.5), tuân thủ token budget, đảm bảo ACL, và
  phát hiện mâu thuẫn giữa các nguồn được đưa vào cùng context.

## 7. State và dữ liệu

Fusion/reranking về cơ bản stateless theo từng request; context builder cần giữ metadata đủ để
truy vết ngược (source ID, version, vị trí trong tài liệu gốc) cho bước citation validation ở
generation.

## 8. Thuật toán liên quan

Reciprocal Rank Fusion; cross-encoder (mô hình phân loại cặp query-document); Maximal Marginal
Relevance (kết hợp similarity với query và dissimilarity với các kết quả đã chọn).

## 9. Cách triển khai

1. Luôn có bước fusion (RRF là lựa chọn an toàn mặc định, ít cần tinh chỉnh) ngay khi dùng nhiều
   hơn một retriever.
2. Thêm cross-encoder reranking cho top-N sau fusion (thường N = 20-50) trước khi cắt xuống top-K
   cuối (thường K = 3-10) đưa vào prompt — đây là cải thiện chi phí thấp, lợi ích cao phổ biến
   nhất trong RAG production.
3. Chỉ thêm LLM reranking khi cross-encoder không đủ để đánh giá các tiêu chí phức tạp (ví dụ
   phân biệt phiên bản tài liệu) — chi phí cao hơn đáng kể.
4. Luôn có bước Evidence packing tường minh — không nối thẳng các chunk lấy được vào prompt mà
   không qua dedupe/group/token budget/ACL check.

## 10. Tham số cần tuning

Hằng số `k` trong RRF (thường giá trị nhỏ như 60 theo thực nghiệm phổ biến, nhưng cần test lại
theo dữ liệu thực tế); trọng số `w1/w2/w3` trong weighted fusion; N (số candidate đưa vào
reranking) và K (số candidate cuối cùng sau reranking); hệ số cân bằng relevance/diversity trong
MMR; token budget cho context cuối cùng.

## 11. Failure modes

- **Cộng điểm thô không chuẩn hoá**: cộng trực tiếp BM25 score (có thể là số bất kỳ, không giới
  hạn) với cosine similarity (0-1) mà không chuẩn hoá — kết quả bị chi phối hoàn toàn bởi retriever
  có thang điểm lớn hơn.
- **Reranking toàn bộ candidate**: chạy cross-encoder/LLM reranking trên hàng trăm candidate thay
  vì top-N đã lọc — chi phí và latency tăng vọt không cần thiết.
- **Context trùng lặp**: không dedupe, đưa nhiều chunk gần như giống hệt nhau vào cùng prompt,
  lãng phí token budget mà không thêm thông tin mới.
- **Bỏ qua phát hiện mâu thuẫn**: đưa cả tài liệu phiên bản cũ và mới vào cùng context mà không
  đánh dấu, khiến model có thể trích dẫn nhầm thông tin đã lỗi thời.

## 12. Security considerations

Evidence packing là điểm kiểm tra ACL **cuối cùng** trước khi nội dung vào prompt — dù retrieval
đã lọc ACL, vẫn cần double-check ở bước này vì fusion/reranking có thể vô tình đưa vào candidate
từ nguồn khác (ví dụ multi-index/federated) chưa qua đúng bộ lọc quyền.

## 13. Observability

Log riêng: kết quả trước fusion (từng retriever), sau fusion, sau reranking, và context cuối cùng
— cho phép xác định chính xác bước nào loại bỏ một kết quả đáng lẽ nên giữ lại khi debug một câu
trả lời sai.

## 14. Evaluation metrics

So sánh Recall@K/NDCG trước và sau reranking (đo được cải thiện thực tế của reranking, không chỉ
giả định nó luôn tốt hơn); tỷ lệ context bị cắt do vượt token budget; tỷ lệ phát hiện mâu thuẫn
đúng trong evidence packing.

## 15. Ưu điểm

Fusion + reranking + evidence packing là chuỗi cải thiện chất lượng có chi phí/lợi ích tốt nhất
trong toàn bộ pipeline RAG — thường mang lại cải thiện lớn hơn nhiều so với việc tinh chỉnh
chunking hay đổi embedding model, với chi phí kỹ thuật tương đối thấp (đặc biệt RRF + cross-encoder
reranking).

## 16. Nhược điểm

Mỗi bước thêm vào (đặc biệt reranking) tăng latency của query pipeline; LLM reranking đặc biệt có
thể trở thành nút thắt cổ chai chi phí nếu áp dụng cho mọi request mà không giới hạn N.

## 17. Khi nên dùng

RRF/Weighted fusion: bắt buộc với Hybrid retrieval. Cross-encoder reranking: nên có mặc định trong
hầu hết hệ thống production. LLM reranking/Diversity reranking: áp dụng có chọn lọc theo nhu cầu
cụ thể. Evidence packing: luôn luôn cần, không có ngoại lệ.

## 18. Khi không nên dùng

Nếu chỉ dùng một retriever duy nhất (không hybrid), bước fusion không cần thiết — nhưng reranking
và evidence packing vẫn nên giữ.

## 19. Pattern liên quan

Nhận input từ Retrieval (file 06); output là context cho bước generation (nằm ngoài phạm vi RAG
thuần túy). Evidence packing liên hệ trực tiếp tới Citation validation trong Query pipeline (file
02) và Temporal indexing (mục 3.5) để ưu tiên phiên bản hiện hành.

## 20. Ví dụ kiến trúc thực tế

Hệ thống hỏi-đáp tài liệu pháp lý: BM25 + vector retrieval trả về top-30 mỗi bên → RRF hợp nhất
thành top-40 → cross-encoder rerank xuống top-8 → evidence packing dedupe, ưu tiên điều khoản còn
hiệu lực (loại điều khoản đã bị thay thế trừ khi user hỏi về lịch sử), gắn source ID cho từng đoạn
để model bắt buộc trích dẫn điều khoản cụ thể trong câu trả lời.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về Reciprocal Rank Fusion, cross-encoder
reranking và Maximal Marginal Relevance trong hệ thống information retrieval/RAG. Nội dung là
tổng hợp và diễn giải lại.*
