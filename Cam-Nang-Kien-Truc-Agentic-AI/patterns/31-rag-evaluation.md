# 31. RAG evaluation

[← Về mục lục chính](../README.md)

## 1. Tên pattern

RAG evaluation — bộ chỉ số đánh giá theo ba tầng: Retrieval (Recall@K, Precision@K, MRR, NDCG, Hit
Rate, Filter accuracy), Generation (Faithfulness, Answer relevance, Completeness, Citation
correctness, Citation completeness, Unsupported claim rate), End-to-end (Task success, User
satisfaction, Latency, Cost, Abstention quality, Security leakage).

## 2. Vấn đề cần giải quyết

Hệ thống RAG có hai tầng xử lý độc lập (retrieval và generation, liên hệ file 2), mỗi tầng có thể
lỗi theo cách khác nhau — retrieval trả về tài liệu không liên quan, hoặc generation diễn giải sai
tài liệu đúng đã truy xuất được. Chỉ đo một chỉ số end-to-end duy nhất (ví dụ "câu trả lời đúng hay
sai") không cho biết lỗi nằm ở tầng nào, khiến việc debug và cải thiện hệ thống mất phương hướng —
cải thiện nhầm tầng không gây ra vấn đề trong khi tầng thực sự lỗi bị bỏ qua.

## 3. Bối cảnh sử dụng

Mọi hệ thống RAG từ giai đoạn phát triển (đánh giá offline trên tập test) tới production (theo dõi
liên tục chất lượng theo thời gian, phát hiện suy giảm khi dữ liệu nguồn hoặc pattern truy vấn thay
đổi).

## 4. Kiến trúc

```
Retrieval evaluation   →  đo chất lượng tập tài liệu truy xuất được (độc lập với generation)
Generation evaluation  →  đo chất lượng câu trả lời DỰA TRÊN tài liệu đã truy xuất (giả định
                           retrieval đã đúng, cô lập lỗi generation)
End-to-end evaluation  →  đo trải nghiệm/kết quả thực tế toàn hệ thống
```

## 5. Thành phần

Tập test có ground truth (câu hỏi + tài liệu liên quan đã biết + câu trả lời tham chiếu, nếu có);
retrieval metric calculator; generation judge (thường LLM-as-judge cho faithfulness/relevance); cơ
chế theo dõi liên tục trên traffic thực (không chỉ tập test tĩnh).

## 6. Luồng xử lý chi tiết

**Tầng Retrieval** đo chất lượng tập tài liệu được truy xuất, độc lập hoàn toàn với việc generation
xử lý chúng ra sao: *Recall@K* (trong K tài liệu top đầu, tỷ lệ tài liệu liên quan thực sự được tìm
thấy so với tổng số tài liệu liên quan có trong corpus); *Precision@K* (trong K tài liệu trả về, tỷ
lệ thực sự liên quan); *MRR* — Mean Reciprocal Rank (đo vị trí của tài liệu liên quan đầu tiên trong
danh sách kết quả, càng cao càng sớm); *NDCG* — Normalized Discounted Cumulative Gain (đo chất lượng
xếp hạng có tính đến mức độ liên quan khác nhau của từng tài liệu, không chỉ liên quan/không liên
quan nhị phân); *Hit Rate* (tỷ lệ truy vấn có ít nhất một tài liệu liên quan trong top-K); *Filter
accuracy* (khi truy vấn có điều kiện lọc metadata — tenant, ngày tháng, loại tài liệu — đo tỷ lệ kết
quả tuân thủ đúng điều kiện lọc).

**Tầng Generation** đo chất lượng câu trả lời với giả định retrieval đã cung cấp đúng tài liệu (cô
lập lỗi generation khỏi lỗi retrieval): *Faithfulness* (câu trả lời có bám sát nội dung tài liệu
được cung cấp không, hay bịa thêm thông tin không có trong nguồn — chỉ số quan trọng nhất chống
hallucination trong RAG); *Answer relevance* (câu trả lời có thực sự trả lời đúng câu hỏi được hỏi
không, tách biệt khỏi việc có bám sát nguồn hay không); *Completeness* (câu trả lời có bao quát đầy
đủ thông tin liên quan có trong tài liệu truy xuất được, hay bỏ sót phần quan trọng); *Citation
correctness* (trích dẫn có trỏ đúng tới tài liệu thực sự chứa thông tin được nêu không); *Citation
completeness* (mọi khẳng định cần trích dẫn có được trích dẫn đầy đủ không, không bỏ sót); *Unsupported
claim rate* (tỷ lệ khẳng định trong câu trả lời không có căn cứ trong tài liệu truy xuất được — liên
hệ trực tiếp ngược với faithfulness).

**Tầng End-to-end** đo kết quả/trải nghiệm thực tế toàn hệ thống: *Task success* (user có đạt được
mục đích ban đầu không); *User satisfaction* (đo trực tiếp qua phản hồi user hoặc gián tiếp qua hành
vi); *Latency* (thời gian phản hồi thực tế, tính cả retrieval lẫn generation); *Cost* (chi phí tính
toán mỗi truy vấn); *Abstention quality* (khi hệ thống không có đủ thông tin để trả lời, có từ chối
đúng cách thay vì bịa câu trả lời không, và có giải thích rõ lý do không); *Security leakage* (câu
trả lời có vô tình lộ thông tin ngoài phạm vi user được phép thấy không — liên hệ Retrieval guardrail
mục 29.5 và Output guardrail mục 29.7).

## 7. State và dữ liệu

Tập test cần ground truth được duy trì và cập nhật theo thời gian (dữ liệu nguồn thay đổi, tập test
cũ có thể không còn phản ánh đúng corpus hiện tại); kết quả evaluation nên lưu lịch sử theo thời gian
để phát hiện xu hướng suy giảm, không chỉ một lần chụp tại một thời điểm.

## 8. Thuật toán liên quan

MRR, NDCG là các công thức xếp hạng chuẩn trong information retrieval; LLM-as-judge (dùng một model
để chấm điểm faithfulness/relevance) là kỹ thuật phổ biến nhất hiện nay cho các chỉ số generation khó
đo bằng công thức đơn giản.

## 9. Cách triển khai

1. Xây tập test có ground truth **tách biệt rõ** cho retrieval (câu hỏi + tài liệu liên quan đã biết)
   và generation (câu hỏi + tài liệu + câu trả lời tham chiếu, nếu cần) — trộn lẫn hai loại làm mất
   khả năng cô lập lỗi theo tầng.
2. Đo retrieval **độc lập** với generation trước — nếu retrieval đã kém, không có ý nghĩa đầu tư sâu
   vào tối ưu generation trên dữ liệu đầu vào sai từ gốc.
3. Dùng LLM-as-judge cho faithfulness/relevance nhưng validate định kỳ judge đó với đánh giá con
   người trên một mẫu nhỏ, tránh judge tự nó bị lệch (bias) mà không ai phát hiện.
4. Theo dõi liên tục trên traffic thực (không chỉ tập test tĩnh một lần) — corpus và pattern truy vấn
   thay đổi theo thời gian, chỉ số đo một lần lúc launch không phản ánh chất lượng hiện tại.
5. Đưa Abstention quality và Security leakage vào bộ chỉ số chuẩn ngay từ đầu — đây là hai chỉ số dễ
   bị bỏ sót nhưng quan trọng cho production thực tế.

## 10. Tham số cần tuning

Giá trị K trong các chỉ số @K (Recall@K, Precision@K) cần khớp với K thực tế hệ thống dùng khi truy
xuất; ngưỡng chấp nhận cho mỗi chỉ số (dưới ngưỡng nào coi là regression cần chặn deploy); tần suất
chạy lại evaluation trên traffic thực.

## 11. Failure modes

- **Chỉ đo end-to-end**: không biết lỗi nằm ở retrieval hay generation khi task success thấp, mất
  phương hướng cải thiện.
- **Tập test lỗi thời**: ground truth không còn khớp với corpus hiện tại (tài liệu đã đổi/xoá), chỉ
  số đo ra không phản ánh đúng chất lượng thực tế.
- **LLM-as-judge không được validate**: judge có bias hệ thống (ví dụ luôn chấm cao câu trả lời dài)
  mà không ai phát hiện, dẫn tới tối ưu sai hướng theo chỉ số judge thay vì chất lượng thực.
- **Bỏ qua Abstention quality**: hệ thống được tối ưu để luôn trả lời (kể cả khi không đủ thông tin),
  tăng hallucination thay vì từ chối đúng cách.

## 12. Security considerations

Security leakage là chỉ số bắt buộc, không tuỳ chọn — cần test riêng biệt kịch bản truy vấn cố tình
khai thác để lộ thông tin ngoài phạm vi quyền truy cập (liên hệ Retrieval guardrail mục 29.5), không
chỉ đo chất lượng câu trả lời thông thường.

## 13. Observability

Liên kết kết quả evaluation với trace cụ thể (liên hệ file 34) để có thể truy ngược từ một điểm chất
lượng thấp về đúng truy vấn, tài liệu truy xuất, và câu trả lời tương ứng — không chỉ có con số tổng
hợp mà không có khả năng drill-down.

## 14. Evaluation metrics

(Bản thân file này định nghĩa bộ chỉ số — xem mục 6.) Chỉ số meta quan trọng: mức độ tương quan giữa
điểm LLM-as-judge và đánh giá con người trên tập mẫu kiểm định, đo độ tin cậy của chính hệ thống
evaluation.

## 15. Ưu điểm

Tách rõ ba tầng cho phép định vị chính xác nguồn gốc lỗi, hướng nỗ lực cải thiện đúng chỗ thay vì đoán
mò; bộ chỉ số toàn diện bao phủ cả chất lượng kỹ thuật (retrieval), chất lượng nội dung (generation),
và trải nghiệm thực tế (end-to-end).

## 16. Nhược điểm

Xây dựng và duy trì tập test có ground truth chất lượng cao tốn công sức đáng kể; nhiều chỉ số (đặc
biệt tầng generation) phụ thuộc LLM-as-judge, tự nó có thể có sai số cần kiểm định định kỳ.

## 17. Khi nên dùng

Mọi hệ thống RAG từ giai đoạn phát triển tới production — nên có bộ chỉ số cả ba tầng ngay từ đầu,
không chờ tới khi phát hiện vấn đề chất lượng mới xây evaluation.

## 18. Khi không nên dùng

Không có ngoại lệ hợp lý — mọi hệ thống RAG production cần ít nhất một tập hợp con cơ bản của các chỉ
số này; có thể giảm nhẹ độ phức tạp (ví dụ bỏ NDCG, chỉ dùng Recall/Precision) cho hệ thống quy mô
nhỏ, nhưng không nên bỏ hoàn toàn evaluation.

## 19. Pattern liên quan

Đo trực tiếp chất lượng của Retrieval patterns (file 6), Fusion/reranking (file 9), và toàn bộ kiến
trúc RAG (file 2); bổ sung bởi Observability (file 34) để liên kết chỉ số với trace cụ thể; liên hệ
Output guardrail (mục 29.7) cho Security leakage.

## 20. Ví dụ kiến trúc thực tế

Hệ thống RAG hỗ trợ tra cứu tài liệu nội bộ doanh nghiệp: pipeline CI chạy bộ test retrieval
(Recall@10, MRR) trên mỗi thay đổi cấu hình chunking/embedding trước khi deploy; production theo dõi
liên tục Faithfulness và Unsupported claim rate qua LLM-as-judge trên mẫu traffic thực hàng ngày,
cảnh báo nếu giảm dưới ngưỡng; đội vận hành có dashboard drill-down từ một điểm User satisfaction thấp
xuống tận trace cụ thể để xác định lỗi nằm ở retrieval (tài liệu sai) hay generation (diễn giải sai
tài liệu đúng).

---

*Nguồn tham khảo dùng khi biên soạn: tổng hợp và diễn giải lại các chỉ số đánh giá RAG phổ biến trong
ngành (information retrieval metrics truyền thống kết hợp LLM-as-judge cho generation quality). Nội
dung không trích dẫn nguyên văn bất kỳ tài liệu cụ thể nào.*
