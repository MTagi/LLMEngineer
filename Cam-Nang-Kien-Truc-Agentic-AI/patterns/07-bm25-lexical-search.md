# 07. BM25 và lexical search

[← Về mục lục chính](../README.md)

## 1. Tên pattern

BM25 (Okapi BM25) lexical search — nhánh sparse retrieval trong hệ thống Hybrid retrieval
(mục 6.3, file 06).

## 2. Vấn đề cần giải quyết

Dense retrieval (embedding) rất mạnh ở khớp ý nghĩa nhưng yếu ở khớp **chính xác** — mã lỗi, tên
class, đường dẫn API, số phiên bản. Một embedding model có thể coi `ERR_AGENT_042` và
`ERR_AGENT_043` là gần giống nhau về mặt ngữ nghĩa (đều là "mã lỗi agent"), trong khi user hỏi
đích danh một mã cụ thể cần kết quả chính xác tuyệt đối. BM25 giải quyết đúng lớp vấn đề này bằng
thống kê tần suất từ, không qua một mô hình học máy "hiểu" ý nghĩa.

## 3. Bối cảnh sử dụng

Tài liệu kỹ thuật (mã lỗi, tên hàm/class, đường dẫn API, số phiên bản, từ viết tắt sản phẩm) —
gần như luôn nên có mặt trong bất kỳ hệ thống RAG production nào dưới dạng một nhánh của Hybrid
retrieval, hiếm khi dùng đơn độc.

## 4. Kiến trúc

```
Query/Document → Character Filter → Tokenizer → Lowercase → Stop-word Filter →
                  Stemming/Lemmatization → Synonym Filter → Term ready to score

Score(document, query) = Σ IDF(term) × [TF(term, document) × (k1 + 1)] /
                          [TF(term, document) + k1 × (1 - b + b × |document| / avgdl)]
```

Hai tham số quyết định hành vi scoring: `k1` (mức độ bão hoà của term frequency — sau một điểm
nào đó, xuất hiện thêm cùng một từ không làm tăng score nhiều nữa) và `b` (mức độ normalization
theo độ dài tài liệu — tài liệu dài tự nhiên có nhiều từ hơn, `b` bù trừ để không thiên vị tài
liệu dài).

## 5. Thành phần

Analyzer pipeline (character filter → tokenizer → lowercase → stop-word → stemmer → synonym);
inverted index (ánh xạ term → danh sách document chứa term đó); engine tính BM25 score (thường
tích hợp sẵn trong search engine, không tự cài đặt lại).

## 6. Luồng xử lý chi tiết

Trước khi tính điểm, cả document (lúc index) và query (lúc search) đều đi qua cùng một
**analyzer**: character filter (chuẩn hoá ký tự), tokenizer (tách từ), lowercase, stop-word
filter (loại từ phổ biến không mang thông tin như "là", "và"), stemming/lemmatization (đưa từ về
gốc), synonym filter (mở rộng từ đồng nghĩa nếu có từ điển). Sau đó engine tính điểm theo công
thức BM25: mỗi term trong query đóng góp điểm dựa trên tần suất xuất hiện trong document
(term frequency), độ hiếm của term trong toàn corpus (inverse document frequency — term càng
hiếm càng có trọng số cao khi khớp), có điều chỉnh bão hoà (`k1`) và chuẩn hoá theo độ dài
(`b`). OpenSearch dùng Okapi BM25 làm mặc định cho keyword search; Elasticsearch và Solr dựa trên
Lucene, trong đó BM25 là cơ chế scoring lexical phổ biến.

## 7. State và dữ liệu

Inverted index (term → posting list các document chứa term, kèm vị trí/tần suất); thống kê toàn
corpus cần cho IDF (tổng số document, số document chứa mỗi term) — phải cập nhật khi corpus thay
đổi, liên hệ Incremental indexing (mục 3.4).

## 8. Thuật toán liên quan

Okapi BM25 (biến thể phổ biến nhất của họ thuật toán BM); TF-IDF là tiền thân đơn giản hơn (BM25
thêm bão hoà term frequency và chuẩn hoá độ dài mà TF-IDF thuần không có).

## 9. Cách triển khai

1. Chọn search engine có BM25 sẵn (OpenSearch/Elasticsearch/Solr) thay vì tự cài đặt lại thuật
   toán — phần giá trị thực sự nằm ở việc **cấu hình đúng analyzer**, không phải viết lại công
   thức scoring.
2. Xây dựng **danh sách bảo toàn** (không stem/tách) cho các loại token đặc thù: mã lỗi, tên
   class, đường dẫn API, số phiên bản, từ viết tắt sản phẩm — đây là bước dễ bị bỏ qua nhất và
   gây hại nhiều nhất nếu bỏ qua.
3. Test analyzer bằng cách chạy thử qua các query mẫu chứa token kỹ thuật thực tế, kiểm tra token
   hoá ra có đúng như mong đợi không (không bị tách vỡ, không bị stem sai).
4. Tinh chỉnh `k1`/`b` sau khi có dữ liệu đánh giá thực tế — không đoán mò trước.

## 10. Tham số cần tuning

`k1` (thường 1.2-2.0, giá trị mặc định phổ biến 1.2); `b` (0-1, mặc định phổ biến 0.75); danh sách
stop-word (có thể cần tuỳ biến theo domain — một số "stop word" thông thường lại mang nghĩa quan
trọng trong domain kỹ thuật); danh sách bảo toàn token không stem.

## 11. Failure modes

- **Stem sai token kỹ thuật**: `CheckpointManager` bị tách/stem thành `checkpoint` + `manag`, mất
  khả năng khớp chính xác tên class.
- **Tách vỡ đường dẫn API**: `/v1/tasks/cancel` bị tokenizer mặc định tách thành các từ riêng lẻ
  mất cấu trúc đường dẫn, khớp nhầm với các API path khác chứa từ tương tự.
- **Mã lỗi bị coi là stop-word hoặc số thường**: `ERR_AGENT_042` bị xử lý sai nếu tokenizer không
  nhận diện đây là một định danh nguyên khối.
- **Dùng chung analyzer cho mọi loại nội dung**: áp cùng một cấu hình phân tích cho cả văn bản tự
  nhiên lẫn nội dung kỹ thuật, khiến một trong hai loại bị xử lý kém.

## 12. Security considerations

BM25 tự nó không có rủi ro bảo mật đặc thù, nhưng inverted index cần tuân theo cùng ACL/tenant
filter như vector index — không nên coi lexical search là "kênh phụ" ít cần kiểm soát quyền hơn
vector search.

## 13. Observability

Log riêng điểm BM25 (tách khỏi điểm vector) trong mỗi kết quả để debug khi hybrid fusion (file 09)
cho kết quả không như mong đợi — cần biết một kết quả được xếp hạng cao vì khớp từ khoá hay vì
khớp ngữ nghĩa.

## 14. Evaluation metrics

So sánh Recall@K/Precision@K của BM25 đơn thuần với dense đơn thuần trên cùng bộ query — đặc biệt
quan trọng trên tập con query chứa mã định danh/thuật ngữ kỹ thuật chính xác, nơi BM25 thường vượt
trội rõ rệt so với dense.

## 15. Ưu điểm

Nhanh, rẻ, dễ giải thích (có thể chỉ ra chính xác term nào đóng góp bao nhiêu điểm); không cần
GPU/embedding model; khớp chính xác tuyệt đối cho định danh kỹ thuật mà dense retrieval hay bỏ
lỡ.

## 16. Nhược điểm

Không hiểu ý nghĩa — bỏ lỡ hoàn toàn các trường hợp diễn đạt khác chữ nhưng cùng ý; chất lượng phụ
thuộc mạnh vào việc cấu hình analyzer đúng theo domain, cấu hình sai gây hại thầm lặng (không lỗi
rõ ràng, chỉ recall thấp hơn mà khó nhận ra ngay).

## 17. Khi nên dùng

Gần như luôn nên có mặt như một nhánh trong Hybrid retrieval (file 06) cho bất kỳ corpus kỹ thuật
nào; đặc biệt quan trọng khi user thường hỏi bằng mã lỗi/tên hàm/thuật ngữ chính xác.

## 18. Khi không nên dùng

Với corpus thuần văn bản tự nhiên, không có định danh kỹ thuật, và câu hỏi luôn diễn đạt khác chữ
so với tài liệu — dense retrieval đơn thuần có thể đủ, BM25 đóng góp ít giá trị.

## 19. Pattern liên quan

Là một nhánh bắt buộc trong Hybrid retrieval (mục 6.3, file 06); kết quả BM25 và dense được hợp
nhất qua RRF/weighted fusion (mục 9.1-9.2, file 09).

## 20. Ví dụ kiến trúc thực tế

Hệ thống tìm kiếm tài liệu API nội bộ: BM25 với danh sách bảo toàn gồm toàn bộ tên endpoint, tên
class, và mã lỗi trích xuất tự động từ codebase (không để analyzer mặc định xử lý các token này);
chạy song song với dense retrieval, hợp nhất qua RRF — đảm bảo câu hỏi "lỗi ERR_AGENT_042 là gì"
luôn tìm đúng tài liệu chứa chính xác mã đó, đồng thời câu hỏi diễn đạt tự nhiên vẫn được dense
retrieval xử lý tốt.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về Okapi BM25 và cách các search engine phổ
biến (OpenSearch, Elasticsearch/Lucene, Solr) triển khai lexical scoring. Nội dung là tổng hợp và
diễn giải lại.*
