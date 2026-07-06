# Format output chung cho dữ liệu extract từ tài liệu

Schema chung (superset) cho **mọi phương pháp extract** trong `Document_Parsing/`
(PDFParse, DocParse, HTMLParse, ...). Mỗi phương pháp có một adapter nhỏ
normalize output thô về schema này: điền được field nào thì điền, không thì bỏ qua.
Nhờ đó downstream (chunking, vector DB, citation, benchmark giữa các extractor)
chỉ viết một lần.

- **File output**: JSONL (`chunks.jsonl`) — mỗi dòng một record ứng với một block/chunk nội dung.
- **Có thể xuất kèm** bản human-readable (`document.md`) để review, nhưng JSONL là source of truth.
- **Validate** bằng [`output_schema.json`](./output_schema.json) (JSON Schema draft 2020-12).

## Tầng 1 — Field bắt buộc (mọi extractor phải điền được)

| Field | Kiểu | Mô tả |
|---|---|---|
| `chunk_id` | string | ID duy nhất, đặt theo quy luật `doc__p<trang>__<index>` để trace ngược và cho LLM cite |
| `doc_id` | string | Tên/ID tài liệu gốc (vd. `bao_cao_2025.pdf`) |
| `content` | string | Nội dung block. Bảng giữ dạng Markdown/HTML table, không ép về text phẳng |
| `content_format` | `"plain"` \| `"markdown"` | pypdf/OCR xuất `plain`; MinerU/PyMuPDF4LLM/Docling/VLM xuất `markdown` |
| `page_start` | integer ≥ 1 | Trang bắt đầu (đánh từ 1) |
| `page_end` | integer ≥ 1 | Trang kết thúc (chunk có thể vắt qua 2 trang; nếu 1 trang thì bằng `page_start`) |
| `extractor` | string | Phương pháp + version (vd. `pypdf-5.x`, `mineru-2.x`) — bắt buộc để benchmark/so sánh |

## Tầng 2 — Field optional (extractor đủ khả năng mới điền)

| Field | Kiểu | Mô tả |
|---|---|---|
| `type` | `"text"` \| `"title"` \| `"table"` \| `"image"` \| `"formula"` \| `"list"` | Loại block; mặc định coi là `text`. Dùng để xử lý khác nhau khi chunking (vd. không cắt đôi bảng) |
| `section_path` | array of string | Chuỗi heading từ gốc xuống (vd. `["3. Kết quả", "3.2 Kết quả kinh doanh"]`) — cho citation kiểu "mục 3.2, trang 12" |
| `bbox` | array of 4 number | Tọa độ block trên trang `[x0, y0, x1, y1]` — để highlight vùng trong PDF viewer khi bấm citation |
| `ocr_confidence` | number 0–1 | Độ tin cậy OCR, chỉ có khi block đi qua OCR |
| `extracted_at` | string (date) | Ngày extract, ISO `YYYY-MM-DD` |

**Quy ước nghiêm ngặt**: consumer KHÔNG được giả định field optional tồn tại —
luôn xử lý được record chỉ có tầng 1 (tức mức pypdf).

## Ma trận khả năng theo phương pháp (PDF)

| Field | pypdf | PyMuPDF | pdfplumber | PyMuPDF4LLM | MinerU/Docling | OCR (Tesseract) | VLM/Claude |
|---|---|---|---|---|---|---|---|
| `content` + `page` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `content_format` | plain | plain | plain | markdown | markdown | plain | markdown |
| `type` | ❌ | một phần | table ✅ | ✅ | ✅ | ❌ | ✅ |
| `section_path` | ❌* | ❌* | ❌ | ✅ | ✅ | ❌ | ✅ |
| `bbox` | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `ocr_confidence` | — | — | — | — | ✅ (khi OCR) | ✅ | — |

*\* điền được nếu PDF có bookmark/outline (đọc qua `reader.outline` rồi map bookmark → trang).*

## Ví dụ

### Record tối thiểu (pypdf — mỗi trang một record)

```json
{"chunk_id": "bao_cao__p12", "doc_id": "bao_cao.pdf", "content": "Doanh thu quý 4 đạt 120 tỷ đồng...", "content_format": "plain", "page_start": 12, "page_end": 12, "extractor": "pypdf-5.x"}
```

Adapter tương ứng:

```python
for i, page in enumerate(reader.pages, start=1):
    records.append({
        "chunk_id": f"{doc_id}__p{i}",
        "doc_id": doc_id,
        "content": page.extract_text() or "",
        "content_format": "plain",
        "page_start": i,
        "page_end": i,
        "extractor": "pypdf",
    })
```

### Record đầy đủ (MinerU/Docling)

```json
{
  "chunk_id": "bao_cao__p12__003",
  "doc_id": "bao_cao.pdf",
  "content": "## 3.2 Kết quả kinh doanh\n\nDoanh thu quý 4 đạt **120 tỷ đồng**, tăng 15% so với cùng kỳ...",
  "content_format": "markdown",
  "page_start": 12,
  "page_end": 12,
  "extractor": "mineru-2.x",
  "type": "text",
  "section_path": ["3. Kết quả", "3.2 Kết quả kinh doanh"],
  "bbox": [72.0, 340.5, 523.2, 610.8],
  "extracted_at": "2026-07-06"
}
```

## Cách dùng để trích nguồn trong RAG

Khi đưa context vào prompt, render kèm nguồn từ metadata:

```text
[bao_cao.pdf | trang 12 | mục 3.2]
Doanh thu quý 4 đạt 120 tỷ đồng...
```

và yêu cầu model cite theo format `(nguồn: <file>, tr.<trang>)`. Vì trang/section
nằm trong metadata chứ không trộn vào `content`, có thể đổi cách hiển thị
citation bất kỳ lúc nào mà không phải re-extract.
