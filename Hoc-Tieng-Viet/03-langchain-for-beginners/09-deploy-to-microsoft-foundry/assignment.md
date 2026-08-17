# Bài tập: Triển khai Agent LangChain lên Microsoft Foundry

## Tổng quan

Chỉnh sửa hệ thống agentic RAG bạn đã xây ở Chương 8 và triển khai nó lên một Microsoft Foundry project có sẵn như một hosted agent.

## Yêu cầu trước

- Hoàn thành các bước và yêu cầu trong [chương này](./README.md)
- Hoàn thành [Building Agentic RAG Systems](../08-agentic-rag-systems/README.md)

---

## Thử thách: Triển khai Agentic RAG Agent của bạn

**Mục tiêu**: Triển khai agent agentic RAG bạn đã xây ở Chương 8 như một Microsoft Foundry hosted agent.

**Nhiệm vụ**:

1. Trong [`code/main.py`](./code/main.py), thay thế ghi chú khoá học mẫu và retrieval tool bằng tài liệu và logic retrieval Chương 8 của riêng bạn.
2. Giữ nguyên pattern `create_agent()` và `ResponsesHostServer`, sau đó cập nhật system prompt cho knowledge base của bạn.
3. Thêm bốn giá trị deployment của Chương 09 vào `.env` gốc, dùng [`.env.example` gốc](../.env.example) làm tham khảo. Dùng file này cho cả test local và deployment.
4. Làm theo Bước 4 trong [chapter README](./README.md#bước-4-test-hosted-wrapper-ở-local) để test một request non-streaming tại `http://localhost:8088/responses`.
5. Làm theo Bước 5 trong [chapter README](./README.md#bước-5-triển-khai-lên-foundry-project-có-sẵn-bằng-azd) để import giá trị `.env`, triển khai bằng `azd deploy --no-prompt`, và xác minh deployment bằng `azd ai agent show --output json`.
6. Dùng `azd ai agent invoke` để hỏi ít nhất ba câu hỏi:
   - Một câu hỏi chung agent có thể trả lời trực tiếp
   - Một câu hỏi cụ thể theo tài liệu nên dùng retrieval
   - Một câu hỏi không có trong tài liệu của bạn
7. Xoá bất kỳ agent hoặc resource nào chỉ tạo cho bài tập này khi hoàn thành.

**Tiêu chí thành công**:

- Agent khởi động local không lỗi
- `/responses` chấp nhận request test non-streaming
- `azd ai agent show` báo hosted agent là `active` hoặc `deployed`
- Agent dùng retrieval và trích dẫn nguồn cho câu hỏi knowledge-base
- Agent trả lời khéo léo khi knowledge base không chứa câu trả lời
- Agent được triển khai lên đúng Foundry project có sẵn mà không cần chạy `azd provision`

---

## Checklist nộp bài

Trước khi hoàn thành chương này:

- [ ] Tài liệu và logic retrieval Chương 8 đã được chỉnh sửa trong `code/main.py`
- [ ] Cả bốn giá trị deployment Chương 09 đã được điền trong `.env` gốc
- [ ] Hosted wrapper local chạy thành công
- [ ] `azd ai agent show` báo agent `active` hoặc `deployed`
- [ ] Agent đã triển khai trả lời phù hợp cả ba loại câu hỏi test
- [ ] Agent và resource chỉ dùng cho bài tập đã được dọn dẹp

---

## Cần trợ giúp?

- **Hướng dẫn chương và xử lý sự cố**: Xem lại [Chapter 09 README](./README.md)
- **Hosted agent quickstart**: Xem lại [Microsoft Foundry hosted agent quickstart](https://learn.microsoft.com/azure/foundry/agents/quickstarts/quickstart-hosted-agent)
- **Kiến thức cơ bản về Agentic RAG**: Xem lại [Building Agentic RAG Systems](../08-agentic-rag-systems/README.md)
- **Thiết lập khoá học**: Xem lại [Course Setup](../00-course-setup/README.md)
- **Vẫn còn vướng**: Tham gia [cộng đồng Discord](https://aka.ms/foundry/discord)
