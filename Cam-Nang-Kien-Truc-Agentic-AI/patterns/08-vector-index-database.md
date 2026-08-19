# 08. Vector index và vector database

[← Về mục lục chính](../README.md)

7 kỹ thuật/lựa chọn liên quan đến việc **lưu và tìm kiếm gần đúng (ANN)** trên embedding: Flat
search, HNSW, IVF, Product Quantization, Scalar/Binary quantization, Disk-oriented ANN, và lựa
chọn vector database.

## 1. Tên pattern

Flat search · HNSW · IVF · Product Quantization (PQ) · Scalar/Binary quantization · Disk-oriented
ANN · Lựa chọn vector database.

## 2. Vấn đề cần giải quyết

Tìm k vector gần nhất trong hàng triệu/tỷ vector không thể làm bằng cách so sánh tuần tự (flat
search) một khi corpus đủ lớn — chi phí gần tuyến tính theo số vector khiến latency không chấp
nhận được. Các kỹ thuật ANN (Approximate Nearest Neighbor) đánh đổi một phần độ chính xác để đạt
tốc độ chấp nhận được ở quy mô lớn; các kỹ thuật quantization đánh đổi một phần độ chính xác để
giảm bộ nhớ. Không có lựa chọn "tốt nhất tuyệt đối" — mỗi kỹ thuật là một điểm khác nhau trên
đường cong đánh đổi tốc độ/bộ nhớ/độ chính xác.

## 3. Bối cảnh sử dụng

| Kỹ thuật | Dùng khi |
|---|---|
| Flat search | Corpus nhỏ (vài nghìn-vài chục nghìn vector), cần độ chính xác tuyệt đối |
| HNSW | Cần recall/latency tốt, có đủ RAM để lưu graph |
| IVF | Corpus lớn, chấp nhận đánh đổi recall lấy tốc độ qua điều chỉnh `nprobe` |
| Product Quantization | Cần giảm mạnh bộ nhớ, chấp nhận khoảng cách xấp xỉ |
| Scalar/Binary quantization | Cần giảm RAM/tăng throughput, có thể rescoring bằng vector gốc |
| Disk-oriented ANN | Corpus rất lớn, không đủ RAM để giữ toàn bộ vector/graph |

## 4. Kiến trúc

```
Flat:              Query vector so sánh trực tiếp với TẤT CẢ vector trong index

HNSW:              Tầng cao  ●───────────────●──────────●   (đường kết nối xa, định vị nhanh)
                   Tầng giữa ●──●──●──●──●──●──●──●──●──●
                   Tầng thấp ●●●●●●●●●●●●●●●●●●●●●●●●●●●●   (neighborhood dày, tìm chính xác)

IVF:               Vector space chia thành nlist cluster; query chỉ so sánh trong nprobe
                   cluster gần nhất (nprobe < nlist → nhanh hơn nhưng có thể bỏ lỡ)

PQ:                Vector gốc → chia thành các subvector → mỗi subvector thay bằng
                   code của centroid gần nhất (nén mạnh, so sánh trên code thay vì vector đầy đủ)
```

## 5. Thành phần

Index structure (graph cho HNSW, cluster centroid cho IVF, codebook cho PQ); cơ chế rescoring
(đọc lại vector gốc để tinh chỉnh kết quả sau khi lọc thô bằng vector đã nén — cần thiết với PQ/
scalar/binary quantization vì các kỹ thuật này chỉ cho khoảng cách xấp xỉ); storage layer (RAM cho
HNSW/IVF thông thường, SSD cho disk-oriented ANN).

## 6. Luồng xử lý chi tiết

- **Flat search**: so sánh tuần tự query với mọi vector, đảm bảo chính xác 100% nhưng chi phí gần
  tuyến tính — chỉ khả thi ở quy mô nhỏ.
- **HNSW**: xây một graph nhiều tầng lúc index — tầng cao có ít điểm nhưng đường kết nối dài (giúp
  "nhảy" nhanh tới vùng đúng), tầng thấp có nhiều điểm với neighborhood dày (giúp tìm chính xác
  trong vùng đã định vị). Lúc query, thuật toán đi từ tầng cao xuống tầng thấp, thu hẹp dần vùng
  tìm kiếm. Tham số `efConstruction` ảnh hưởng chất lượng graph lúc xây, `efSearch` ảnh hưởng độ
  chính xác lúc tìm (đánh đổi với tốc độ).
- **IVF**: phân cụm (clustering) toàn bộ vector space thành `nlist` cluster lúc index; lúc query,
  chỉ so sánh trong `nprobe` cluster gần tâm truy vấn nhất thay vì toàn bộ index — `nprobe` càng
  cao, recall càng tốt nhưng càng chậm.
- **Product Quantization**: chia mỗi vector thành nhiều subvector nhỏ, mỗi subvector được thay
  bằng chỉ số (code) của centroid gần nhất trong một codebook đã học trước — giảm bộ nhớ mạnh vì
  lưu code (vài byte) thay vì vector đầy đủ (hàng nghìn byte), đổi lại khoảng cách tính được chỉ
  là xấp xỉ.
- **Scalar/Binary quantization**: đơn giản hơn PQ — giảm độ chính xác số học của từng chiều vector
  (float32 → int8/binary) thay vì nén theo cụm subvector; thường cần bước **rescoring**: lọc thô
  bằng vector đã lượng tử hoá rồi tính lại khoảng cách chính xác bằng vector gốc cho top candidate
  để phục hồi precision.
- **Disk-oriented ANN**: giữ phần lớn vector/graph trên SSD thay vì RAM, chỉ nạp phần cần thiết
  vào bộ nhớ theo yêu cầu truy vấn — mở rộng quy mô corpus vượt giới hạn RAM nhưng cần kiểm soát
  I/O và chiến lược cache để không làm chậm quá mức.

## 7. State và dữ liệu

Index (HNSW graph/IVF cluster assignment/PQ codebook) phải được cập nhật đồng bộ với Incremental
indexing (mục 3.4) — xoá/thêm vector đơn lẻ có chi phí khác nhau tuỳ cấu trúc index (HNSW hỗ trợ
update tốt hơn một số cấu trúc IVF cần rebuild định kỳ).

## 8. Thuật toán liên quan

Hierarchical Navigable Small World graph (HNSW); Inverted File Index với clustering (thường
k-means, IVF); Product Quantization (dựa trên k-means theo từng subvector); scalar/binary
quantization (rời rạc hoá giá trị float).

## 9. Cách triển khai

1. Bắt đầu với **HNSW** làm mặc định cho hầu hết use case vừa/lớn — hỗ trợ tốt nhất bởi các vector
   database phổ biến, cân bằng tốt giữa recall/latency mà không cần tinh chỉnh phức tạp.
2. Chuyển sang **IVF (+PQ nếu cần)** khi quy mô corpus đủ lớn khiến HNSW tốn RAM vượt ngân sách hạ
   tầng.
3. Thêm **scalar/binary quantization + rescoring** khi cần giảm chi phí RAM/tăng throughput nhưng
   vẫn giữ được độ chính xác chấp nhận được nhờ bước rescoring.
4. Chỉ cân nhắc **disk-oriented ANN** khi corpus vượt hẳn khả năng RAM khả thi về chi phí — đây là
   lựa chọn phức tạp nhất, chỉ nên dùng khi các phương án còn lại không đủ.
5. **Lựa chọn database**: đánh giá theo nhu cầu thực tế thay vì chỉ theo benchmark tốc độ ANN —
   FAISS (thư viện cho local/research/custom service), pgvector (khi đã dùng PostgreSQL cho
   metadata/transaction), Qdrant (vector-first, filtering, self-host), Milvus (quy mô lớn,
   distributed), Elasticsearch/OpenSearch (khi cần mạnh cả BM25 + vector + aggregation trong cùng
   hệ thống), Azure AI Search (hệ sinh thái Azure, semantic ranking), Weaviate (object schema +
   hybrid search).

## 10. Tham số cần tuning

HNSW: `M`, `efConstruction`, `efSearch`. IVF: `nlist`, `nprobe`. PQ: số subvector, số bit mỗi code.
Ngưỡng rescoring (top-N candidate được tính lại bằng vector gốc sau khi lọc thô).

## 11. Failure modes

- **HNSW hết RAM khi corpus tăng**: graph phình theo số vector, không có cảnh báo sớm dẫn đến sự
  cố bất ngờ khi corpus vượt ngưỡng dự kiến ban đầu.
- **`nprobe` quá thấp**: IVF bỏ lỡ kết quả đúng nằm ở cluster không được search, recall giảm âm
  thầm mà không có lỗi rõ ràng.
- **Quantization không rescoring**: dùng thẳng khoảng cách xấp xỉ từ PQ/scalar/binary làm kết quả
  cuối mà không rescoring bằng vector gốc — độ chính xác giảm đáng kể so với kỳ vọng.
- **Chọn database chỉ theo benchmark**: bỏ qua các yêu cầu vận hành thực tế (filtering theo
  metadata, update/delete, backup, replication, multi-tenancy) — một số database nhanh nhất theo
  benchmark ANN lại yếu ở các khả năng vận hành này.

## 12. Security considerations

Metadata filtering (ACL, tenant) cần được **vector database hỗ trợ hiệu quả ở tầng index**, không
phải lọc hậu kiểm sau khi lấy kết quả — lọc hậu kiểm có thể khiến top-K trả về rỗng nếu tất cả kết
quả gần nhất đều bị lọc bỏ vì quyền truy cập, trong khi index vốn có thể tìm được kết quả phù hợp
hơn nếu biết trước filter.

## 13. Observability

Theo dõi recall thực tế (so với kết quả flat search làm ground truth trên một tập mẫu) định kỳ,
không chỉ tin vào tham số cấu hình — recall thực tế có thể suy giảm theo thời gian khi phân phối
dữ liệu thay đổi mà tham số cũ không còn phù hợp.

## 14. Evaluation metrics

Recall@K so với flat search ground truth; latency p50/p95/p99; RAM/disk footprint; QPS
(query per second) ở các mức tải khác nhau.

## 15. Ưu điểm

Cho phép retrieval ở quy mô lớn với latency chấp nhận được — không có các kỹ thuật này, RAG không
thể mở rộng vượt quá corpus rất nhỏ.

## 16. Nhược điểm

Mọi kỹ thuật ANN đều đánh đổi độ chính xác lấy tốc độ/bộ nhớ ở một mức độ nào đó; việc chọn sai
tham số có thể gây suy giảm chất lượng retrieval âm thầm, khó phát hiện nếu không đo recall định
kỳ.

## 17. Khi nên dùng

HNSW là lựa chọn mặc định hợp lý cho hầu hết hệ thống RAG production quy mô vừa. Các kỹ thuật còn
lại áp dụng khi có ràng buộc cụ thể về quy mô/chi phí đã được xác nhận qua đo lường thực tế.

## 18. Khi không nên dùng

Với corpus rất nhỏ (vài nghìn vector), flat search đơn giản, chính xác tuyệt đối, và đủ nhanh —
không cần bất kỳ kỹ thuật ANN phức tạp nào.

## 19. Pattern liên quan

Nhận input từ Chunking (file 05) và Embedding; là hạ tầng cho mọi pattern trong Retrieval (file
06); recall của tầng này giới hạn trực tiếp trần chất lượng của Fusion/Reranking (file 09) — không
thể rerank tốt một candidate set vốn đã thiếu kết quả đúng.

## 20. Ví dụ kiến trúc thực tế

Hệ thống RAG doanh nghiệp với ~5 triệu chunk: dùng HNSW làm index chính trên một vector database
hỗ trợ metadata filtering hiệu quả (ví dụ Qdrant hoặc Azure AI Search) để lọc ACL/tenant ngay ở
tầng index; theo dõi recall hàng tuần trên bộ query mẫu so với flat search ground truth trên một
tập con nhỏ để phát hiện sớm suy giảm chất lượng khi corpus tăng trưởng.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về các thuật toán ANN phổ biến (HNSW, IVF,
Product Quantization) và tài liệu công khai của các vector database phổ biến (FAISS, pgvector,
Qdrant, Milvus, Elasticsearch/OpenSearch, Azure AI Search, Weaviate). Nội dung là tổng hợp và diễn
giải lại.*
