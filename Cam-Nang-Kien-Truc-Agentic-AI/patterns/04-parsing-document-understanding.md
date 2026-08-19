# 04. Parsing và document understanding

[← Về mục lục chính](../README.md)

Nhóm 4 pattern xử lý bước **parse tài liệu thô thành cấu trúc có thể chunk/embed đúng cách**:
Layout-aware, Table-aware, Multimodal, Code-aware parsing.

## 1. Tên pattern

Layout-aware parsing · Table-aware parsing · Multimodal parsing · Code-aware parsing.

## 2. Vấn đề cần giải quyết

"Extract plain text" từ PDF/Word/PowerPoint là cái bẫy phổ biến nhất khi mới làm RAG — nó phá vỡ
cấu trúc gốc: ghép nhầm hai cột thành một dòng, chèn header/footer vào giữa câu, tách rời bảng số
liệu khỏi tiêu đề cột, bỏ mất caption của hình. Hậu quả lan xuống toàn bộ pipeline phía sau: chunk
sai ranh giới (file 05), embedding mang ý nghĩa sai, và câu trả lời cuối trích dẫn nhầm ngữ cảnh.

## 3. Bối cảnh sử dụng

Bất kỳ nguồn tài liệu nào không phải plain text thuần: PDF có layout phức tạp (báo cáo, hợp đồng),
tài liệu chứa bảng số liệu, tài liệu chứa hình/sơ đồ, hoặc source code.

## 4. Kiến trúc

```
Layout-aware:  Document → Heading/Paragraph/List/Table/Figure/Caption/Footnote/Header-Footer/
               Page number/Reading order (cấu trúc, không phải chuỗi text phẳng)

Table-aware:   Table → {table_markdown, table_summary, column_names, row_entities, source_page}
               (nhiều representation song song, không chỉ 1 chuỗi text)

Multimodal:    Image → {OCR Text, Caption, Visual Description, Detected Entities,
               Link to Parent Section}

Code-aware:    Repository → Module → Class → {Method, Docstring, Dependencies,
               Call Relationships} + metadata {repo, branch, commit, path, symbol, language,
               imports, caller/callee, related tests}
```

## 5. Thành phần

Layout analysis engine (nhận diện heading/table/figure/reading order — thường dùng document AI/
OCR có layout model, không phải OCR text-only); table extractor giữ được cấu trúc hàng-cột; hoặc
mô-đun sinh mô tả ảnh (image captioning/vision model) cho multimodal; AST/symbol parser theo ngôn
ngữ lập trình cho code.

## 6. Luồng xử lý chi tiết

- **Layout-aware**: phân tích trang thành các khối ngữ nghĩa (heading, paragraph, list, table,
  figure, caption, footnote, header/footer, số trang) và xác định **reading order** đúng (đặc
  biệt quan trọng với layout nhiều cột) trước khi ghép lại thành text tuyến tính để chunk.
- **Table-aware**: một bảng được lưu **song song** dưới nhiều dạng — markdown table (để hiển thị/
  đưa vào prompt), table summary (câu tóm tắt để search theo ý nghĩa), column names + row entities
  (để search có cấu trúc), và tham chiếu trang nguồn. Khi trả lời, hệ thống search trên
  summary/column/entity nhưng **dùng dữ liệu bảng gốc làm evidence** thay vì diễn giải lại.
- **Multimodal**: với mỗi hình/sơ đồ, sinh song song OCR text (chữ trong ảnh), caption, mô tả
  hình ảnh (visual description), thực thể nhận diện được, và liên kết ngược tới section chứa nó.
  Có hai chiến lược thời điểm xử lý: sinh mô tả **lúc indexing** (rẻ hơn ở query time nhưng cố
  định) hoặc **retrieve ảnh gốc và gửi cho multimodal model lúc inference** (linh hoạt hơn nhưng
  tốn chi phí mỗi lần hỏi).
- **Code-aware**: parse theo cấu trúc ngôn ngữ (module → class → method) thay vì cắt file theo số
  dòng cố định; gắn kèm docstring, dependency, quan hệ gọi hàm (caller/callee) và test liên quan
  làm metadata — giúp retrieval trả về đúng đơn vị code có ý nghĩa (một hàm hoàn chỉnh) thay vì
  một đoạn cắt ngang.

## 7. State và dữ liệu

Kết quả parsing là input trực tiếp cho Chunking (file 05); cấu trúc phân cấp (heading → section →
paragraph) cần được giữ lại tường minh (không làm phẳng ngay) để các chiến lược chunking
structure-aware/recursive (mục 5.3-5.4) có thể dùng.

## 8. Thuật toán liên quan

Document layout analysis (mô hình phát hiện khối bố cục trên ảnh trang); OCR; table structure
recognition; AST parsing theo ngôn ngữ lập trình (không phải regex cắt chuỗi).

## 9. Cách triển khai

1. Chọn công cụ parsing theo loại tài liệu chiếm đa số trong corpus (PDF layout phức tạp cần công
   cụ khác hẳn plain markdown hay source code).
2. Không xử lý mọi loại tài liệu bằng một hàm "extract text" chung — tách pipeline riêng cho
   tài liệu văn phòng, bảng, hình, và code.
3. Giữ metadata liên kết ngược (source page, parent section) xuyên suốt để phục vụ citation
   (mục 9.7 ở file 09) sau này.
4. Với multimodal, quyết định sớm chiến lược "mô tả lúc index" hay "gửi ảnh gốc lúc query" —
   ảnh hưởng trực tiếp tới chi phí và kiến trúc runtime.

## 10. Tham số cần tuning

Độ chi tiết của table summary (ngắn cho search, đủ dài để hữu ích); có OCR hay không cho tài liệu
scan; ngưỡng chất lượng OCR để quyết định có cần review thủ công; với code-aware, mức độ chi tiết
symbol cần trích (chỉ public API hay cả private method).

## 11. Failure modes

- **Ghép sai cột**: layout 2 cột bị đọc lẫn thành 1 dòng liên tục, phá hỏng ngữ nghĩa toàn đoạn.
- **Bảng bị "làm phẳng"**: chuyển bảng thành văn bản tuyến tính, mất quan hệ hàng-cột, model
  không còn hiểu được số liệu thuộc dòng/cột nào.
- **Caption tách khỏi hình**: chunk cắt ngang giữa hình và caption, khiến ngữ cảnh mô tả hình bị
  mất khi retrieve.
- **Code bị cắt giữa hàm**: parser không nhận diện ranh giới hàm/class, chunk cắt ngang giữa thân
  hàm, phá vỡ tính đầy đủ về mặt cú pháp lẫn ngữ nghĩa.

## 12. Security considerations

Tài liệu scan/ảnh có thể chứa nội dung nhạy cảm không hiện trong text layer gốc (chỉ lộ ra sau
OCR) — cần áp guardrail nội dung **sau** OCR, không chỉ trước; source code parsing cần loại bỏ
secret/credential (API key, password hardcode) trước khi đưa vào index tìm kiếm được.

## 13. Observability

Theo dõi tỷ lệ tài liệu parse thất bại hoặc parse "đáng ngờ" (page trống bất thường, bảng không
nhận diện được cột nào, OCR confidence thấp) như một hàng đợi cần review, không để âm thầm vào
index với chất lượng kém.

## 14. Evaluation metrics

Độ chính xác reading order (đánh giá thủ công trên mẫu); độ chính xác table extraction (so khớp
với bảng gốc); với multimodal, độ liên quan của mô tả ảnh sinh ra so với nội dung ảnh thật; với
code-aware, tỷ lệ symbol được parse đúng ranh giới cú pháp.

## 15. Ưu điểm

Giữ được cấu trúc và ngữ nghĩa gốc của tài liệu, là nền tảng bắt buộc để các bước sau (chunking,
retrieval, citation) hoạt động đúng — sai từ bước này lan truyền và khuếch đại ở mọi bước sau.

## 16. Nhược điểm

Chi phí kỹ thuật cao hơn đáng kể so với "extract text" đơn giản; cần công cụ/mô hình chuyên biệt
theo loại tài liệu, tăng độ phức tạp vận hành pipeline.

## 17. Khi nên dùng

Bắt buộc với bất kỳ corpus nào có tài liệu định dạng phức tạp hơn plain text — gần như mọi hệ
thống RAG doanh nghiệp thực tế.

## 18. Khi không nên dùng

Với nguồn dữ liệu vốn đã là plain text/markdown sạch (ví dụ export sẵn từ CMS dạng text), bước
parsing phức tạp này không cần thiết.

## 19. Pattern liên quan

Đầu ra trực tiếp làm input cho Chunking (file 05, đặc biệt Structure-aware và Contextual
chunking); metadata sinh ra ở đây (source page, parent section) phục vụ Evidence packing/citation
(mục 9.7 ở file 09).

## 20. Ví dụ kiến trúc thực tế

Hệ thống RAG cho báo cáo tài chính dạng PDF nhiều cột, chứa bảng số liệu quý và biểu đồ: layout
parser tách đúng thứ tự đọc 2 cột, table parser giữ bảng dưới dạng markdown + summary + tên cột
để cả search theo ý nghĩa lẫn trích dẫn số liệu chính xác, multimodal parser sinh mô tả cho biểu
đồ kèm liên kết về đoạn văn bản giải thích biểu đồ đó.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về document AI, layout analysis và table/
code parsing trong hệ thống RAG production. Nội dung là tổng hợp và diễn giải lại.*
