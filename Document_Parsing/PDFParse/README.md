# PDFParse — Các phương pháp đọc & parse file PDF

Tổng hợp các phương pháp đọc PDF cho pipeline LLM/RAG, chia theo 7 nhóm.
Chọn phương pháp phụ thuộc vào: PDF **digital** (có text sẵn) hay **scan** (ảnh),
và có cần giữ **cấu trúc** (heading, bảng, công thức) hay không.

## 1. Direct text extraction (PDF digital, nhanh, miễn phí)

| Tool | Điểm mạnh | Ghi chú |
|---|---|---|
| **PyMuPDF (fitz)** | Nhanh nhất nhóm, trích text + tọa độ + ảnh | License AGPL |
| **pdfplumber** | Mạnh về bảng biểu, truy cập từng ký tự/đường kẻ | Chậm hơn |
| **pypdf** | Nhẹ, thuần Python, thao tác cơ bản (text, tách/gộp trang, metadata) | Baseline tốt |
| **pdfminer.six** | Phân tích layout chi tiết | API khó dùng, thường dùng qua pdfplumber |
| **Poppler / pdftotext** | CLI, cực nhanh, tốt làm baseline/fallback | `pdftoppm` convert trang → ảnh |

## 2. LLM/RAG-oriented parsers (xuất Markdown/JSON có cấu trúc)

| Tool | Điểm mạnh | Ghi chú |
|---|---|---|
| **MinerU** | Layout detection bằng DL, công thức → LaTeX, bảng → HTML/MD, OCR tích hợp (PaddleOCR), tự nhận scan PDF, xuất kèm tọa độ block (tiện citation) | Mạnh nhất với tài liệu khoa học; hỗ trợ cả DOCX/PPTX/XLSX/ảnh; CLI/API/WebUI |
| **Docling** (IBM) | Gọn nhẹ, dễ nhúng pipeline, xuất Markdown/JSON có cấu trúc | Đang được ưa chuộng cho RAG |
| **Marker** | PDF → Markdown chất lượng cao, nhanh | Tốt cho tài liệu khoa học |
| **PyMuPDF4LLM** | Wrapper PyMuPDF xuất Markdown thân thiện LLM, rất nhẹ (không cần model DL) | Phù hợp PDF digital, layout đơn giản; AGPL |
| **Unstructured** | Chia tài liệu thành element (Title, NarrativeText, Table...) tiện chunking | |
| **LlamaParse** | API trả phí (có free tier), parse tốt tài liệu phức tạp | Không cần tự host |

## 3. OCR / scanned PDF

| Tool | Vai trò |
|---|---|
| **Tesseract** (pytesseract) | OCR miễn phí phổ biến nhất, hỗ trợ tiếng Việt (`lang="vie"`) |
| **OCRmyPDF** | Thêm text layer vào PDF scan → PDF searchable, sau đó đọc như PDF digital |
| **PaddleOCR / EasyOCR** | Độ chính xác cao hơn Tesseract, chạy được GPU |
| **pdf2image + OCR** | Convert trang → ảnh (wrapper Poppler `pdftoppm`) rồi OCR từng trang |

Flow production thực tế cho PDF scan:

```text
PDF scan → OCRmyPDF → PDF searchable → PyMuPDF/pdfplumber → chunking/RAG
```

## 4. Scientific paper parsing

- **MinerU** — lựa chọn số một cho paper/công thức toán.
- **GROBID** — parse paper thành cấu trúc học thuật (title, authors, abstract, sections, references), output XML/TEI chuẩn; rất mạnh cho citation/bibliography.
- **Marker** — Markdown đẹp cho academic PDF.
- **Nougat** — từng nổi bật cho academic PDF → Markdown/LaTeX, nay gần như legacy (chỉ tham khảo, MinerU/Marker đáng ưu tiên hơn).

## 5. Vision/LLM direct reading

- **Claude PDF input** — gửi thẳng PDF trong API request, model "nhìn" cả text lẫn layout/hình; tốn token theo số trang.
- **GPT / Gemini vision input** — tương tự với hệ sinh thái khác.
- **Convert trang → ảnh + VLM** — kiểm soát chi phí tốt hơn, chọn trang cần đọc.
- **Self-hosted VLM** — Qwen-VL, InternVL, LLaVA... khi cần chạy nội bộ; MinerU cũng có VLM backend.

## 6. Cloud document AI (production, trả phí)

- **Azure Document Intelligence**
- **AWS Textract**
- **Google Document AI**

Mạnh về form, bảng, chữ viết tay; có SLA, phù hợp khối lượng lớn.

## 7. Conversion-based methods

Đôi khi PDF (nhất là PDF tạo từ Word) giữ cấu trúc tốt hơn khi convert trước rồi mới parse:

```text
PDF → DOCX/HTML → extract structure
```

Công cụ: LibreOffice headless, các converter PDF → DOCX, Microsoft Graph (hệ sinh thái M365), Poppler (`pdftohtml`).

## Bộ demo đề xuất cho folder này (~8 phương pháp)

1. `pypdf` — basic baseline
2. `PyMuPDF` — fast text/layout/image extraction
3. `pdfplumber` — table/layout
4. `PyMuPDF4LLM` — Markdown cho RAG
5. `MinerU` — high-quality Markdown/JSON (OCR/layout/formula/table)
6. `Docling` hoặc `Marker` — so sánh với MinerU
7. `OCRmyPDF + PyMuPDF` — flow scan PDF thực tế
8. `Tesseract/PaddleOCR` — OCR trực tiếp từng page image
