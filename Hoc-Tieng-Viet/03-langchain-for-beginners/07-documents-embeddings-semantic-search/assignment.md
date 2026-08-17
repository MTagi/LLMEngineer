# Bài tập: Documents, Embeddings & Semantic Search

## Tổng quan

Luyện tập xây dựng hệ thống semantic search hiểu ý nghĩa. Bạn sẽ tạo tool minh hoạ cách embedding nắm bắt quan hệ ngữ nghĩa.

## Yêu cầu trước

- Đã hoàn thành [chương này](./README.md)
- Đã chạy tất cả ví dụ code trong chương
- Đã cấu hình biến môi trường

---

## Thử thách 1: Similarity Explorer 🔍

Xây dựng tool cho phép user khám phá cách similarity score thay đổi với query và tài liệu khác nhau.

### Yêu cầu

- Tạo similarity explorer với tập tài liệu đa dạng
- Cho phép user test các query tìm kiếm khác nhau
- Tính và hiển thị similarity score giữa query và tài liệu
- Hiển thị top kết quả xếp hạng theo similarity score
- Minh hoạ cách query có ý nghĩa tương tự trả về kết quả tương tự

### Gợi ý

1. Bắt đầu với import và thiết lập Azure OpenAI:

    ```python
    import os
    from dotenv import load_dotenv
    from langchain_core.documents import Document
    from langchain_core.vectorstores import InMemoryVectorStore
    from langchain_openai import AzureOpenAIEmbeddings
    ```

2. Tạo tài liệu bao quát nhiều chủ đề khác nhau:

    ```python
    docs = [
        Document(page_content="Machine learning models can recognize patterns in data"),
        Document(page_content="The recipe calls for flour, eggs, and butter"),
        Document(page_content="Python is a popular programming language for AI"),
        Document(page_content="The sunset painted the sky in shades of orange"),
        # Add more diverse documents...
    ]
    ```

3. Xây dựng vector store từ tài liệu của bạn:

    ```python
    vector_store = InMemoryVectorStore.from_documents(docs, embeddings)
    ```

4. Dùng `similarity_search_with_score` để lấy similarity score:

    ```python
    results = vector_store.similarity_search_with_score(query, k=5)
    for doc, score in results:
        print(f"Score: {score:.4f} - {doc.page_content}")
    ```

5. Test với các query khác nhau để xem similarity thay đổi ra sao:

    ```python
    queries = [
        "How does AI learn?",
        "What ingredients do I need for baking?",
        "Beautiful evening colors",
    ]
    ```

> **💡 MẸO:** Để ý cách similarity score phản ánh ý nghĩa ngữ nghĩa! "How does AI learn?" nên có điểm cao nhất với tài liệu về machine learning, dù các từ khác nhau.

---

## Thử thách 2: Book Search System (Bonus) 📚

Xây dựng hệ thống semantic search trên một bộ sưu tập mô tả sách.

### Yêu cầu

- Tạo ít nhất 5 mô tả sách với metadata (tên sách, tác giả, thể loại)
- Tạo embedding và lưu vào vector store
- Cài đặt semantic search trả về sách liên quan
- Hiển thị metadata sách (tên, tác giả, thể loại) trong kết quả
- Cho thấy cách tìm kiếm tìm sách dựa trên chủ đề và khái niệm

### Gợi ý

1. Tạo tài liệu với metadata phong phú:

    ```python
    from langchain_core.documents import Document

    books = [
        Document(
            page_content="A young wizard discovers his magical powers and battles dark forces at a school of magic",
            metadata={"title": "Harry Potter", "author": "J.K. Rowling", "genre": "Fantasy"}
        ),
        Document(
            page_content="A hobbit embarks on an epic quest through Middle-earth to destroy a powerful ring",
            metadata={"title": "The Lord of the Rings", "author": "J.R.R. Tolkien", "genre": "Fantasy"}
        ),
        # Add more books with diverse genres and themes...
    ]
    ```

2. Xây dựng vector store và tìm kiếm:

    ```python
    book_store = InMemoryVectorStore.from_documents(books, embeddings)
    results = book_store.similarity_search("adventure stories for kids", k=3)
    ```

3. Hiển thị kết quả kèm metadata:

    ```python
    for doc in results:
        print(f"📖 {doc.metadata['title']} by {doc.metadata['author']}")
        print(f"   Genre: {doc.metadata['genre']}")
        print(f"   {doc.page_content[:100]}...")
    ```

4. Thử query ngữ nghĩa không khớp từ chính xác:

    ```python
    queries = [
        "stories about magic and wizards",
        "epic journey adventures",
        "books about the future",
        "mystery and detective stories",
    ]
    ```

> **💡 MẸO:** Tìm kiếm nên tìm được sách liên quan kể cả khi query dùng từ khác với mô tả! Ví dụ, "stories about magic" nên tìm được cả Harry Potter và Lord of the Rings.

---

## Checklist nộp bài

Trước khi nộp, xác nhận:

- [ ] Similarity Explorer hiển thị tài liệu đa dạng trên nhiều chủ đề khác nhau
- [ ] Similarity score được hiển thị và xếp hạng đúng
- [ ] Query khác nhau trả về kết quả phù hợp về ngữ nghĩa
- [ ] (Bonus) Book Search bao gồm ít nhất 5 sách với metadata
- [ ] (Bonus) Metadata sách hiển thị đúng trong kết quả tìm kiếm

---

## Lời giải

Lời giải cho tất cả thử thách có sẵn trong thư mục [`solution/`](./solution/).

- [`similarity_explorer.py`](./solution/similarity_explorer.py) - Lời giải Thử thách 1
- [`book_search.py`](./solution/book_search.py) - Lời giải Thử thách 2 (Bonus)

---

## Cần trợ giúp?

- Xem lại [chapter README](./README.md) để ôn khái niệm
- Kiểm tra ví dụ code trong [`code/`](./code/)
- Xem cài đặt mẫu trong [`samples/`](./samples/)
- Đặt câu hỏi trong [Discord](https://aka.ms/foundry/discord)

---

## Bước tiếp theo

**[Building Agentic RAG Systems](../08-agentic-rag-systems/README.md)**
