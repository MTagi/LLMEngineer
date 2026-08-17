# Bài tập: Building Agentic RAG Systems

## Tổng quan

Luyện tập xây dựng hệ thống Agentic RAG hiện đại nơi AI agent thông minh quyết định khi nào tìm kiếm tài liệu của bạn so với trả lời từ kiến thức chung, kết hợp retrieval với việc tự ra quyết định.

## Yêu cầu trước

- Đã hoàn thành [chương này](./README.md)
- Đã chạy tất cả ví dụ code trong chương
- Hiểu kiến trúc agentic RAG
- Quen thuộc với agent từ [Getting Started with Agents](../05-agents/README.md)

---

## Thử thách: Personal Knowledge Base Q&A 📚

**Mục tiêu**: Xây dựng hệ thống agentic RAG trên tài liệu của riêng bạn nơi agent quyết định khi nào tìm kiếm.

**Nhiệm vụ**:

1. Tạo `knowledge_base_rag.py` trong thư mục `08-agentic-rag-systems/solution/`
2. Thu thập 5-10 tài liệu về một chủ đề bạn hiểu rõ:
   - Ghi chú cá nhân
   - Bài blog bạn đã viết
   - Tài liệu bạn đã tạo
   - Hoặc dùng văn bản mẫu về sở thích/mối quan tâm
3. Xây dựng hệ thống agentic RAG:
   - Load và chia tài liệu thành vector store
   - Tạo retrieval tool cho agent
   - Dùng `create_agent()` để xây dựng agent tự trị
   - Agent quyết định khi nào tìm kiếm vs trả lời trực tiếp
4. Test với 5+ câu hỏi - kết hợp kiến thức chung và câu hỏi cụ thể theo tài liệu

**Tiêu chí thành công**:

- Load tài liệu thành công
- Agent trả lời câu hỏi chung mà không tìm kiếm
- Agent dùng retrieval tool cho câu hỏi cụ thể theo tài liệu
- Cung cấp câu trả lời chính xác với việc ra quyết định thông minh
- Xử lý khéo léo câu hỏi không có trong knowledge base

**Gợi ý**:

```python
# 1. Import required modules
from langchain.agents import create_agent
from langchain_openai import AzureOpenAIEmbeddings, ChatOpenAI
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_core.documents import Document
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import os

load_dotenv()

# 2. Create AzureOpenAIEmbeddings and ChatOpenAI instances

# 3. Create an array of Document objects:
#    - Use your own content as page_content
#    - Add metadata (title, source, etc.)

# 4. Create an InMemoryVectorStore from your documents

# 5. Create a retrieval tool using @tool decorator:
#    - Define function that searches vector store
#    - Provide clear name and description
#    - Format results with source attribution

@tool
def search_my_notes(query: str) -> str:
    """Search my personal knowledge base for information."""
    results = vector_store.similarity_search(query, k=3)
    return "\n\n".join(
        f"[{doc.metadata['title']}]: {doc.page_content}"
        for doc in results
    )

# 6. Create agent with create_agent():
#    - Pass model and tools list
#    - Add system_prompt for context
#    - Agent will decide when to use retrieval tool

agent = create_agent(
    model,
    tools=[search_my_notes],
    system_prompt="You are a helpful assistant with access to my knowledge base..."
)

# 7. Test with questions that demonstrate agent decision-making:
#    - General knowledge (agent answers directly)
#    - Document-specific (agent searches)
#    - Questions not in docs (agent may search but won't find)

response = agent.invoke({
    "messages": [HumanMessage(content="Your question here")],
})
```

> [!TIP]
> **🤖 Nhờ [GitHub Copilot](../docs/copilot.md) giúp đỡ:** Nếu cần hỗ trợ với thử thách này, mở file này trong editor và dùng [Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) để được hỗ trợ và giải thích cá nhân hoá.

---

## Thử thách Bonus: Conversational Agentic RAG 💬

**Mục tiêu**: Xây dựng hệ thống agentic RAG duy trì lịch sử hội thoại.

**Nhiệm vụ**:

1. Tạo `conversational_rag.py` trong thư mục `08-agentic-rag-systems/solution/`
2. Kết hợp agentic RAG với memory hội thoại
3. Cho phép câu hỏi follow-up tham chiếu ngữ cảnh trước đó:

   ```text
   User: "What is Python?"
   Agent: "Python is..."
   User: "What are its main benefits?" ← Agent understands "its" refers to Python
   ```

4. Cài đặt quản lý lịch sử hội thoại
5. Thêm CLI tương tác cho hội thoại nhiều lượt
6. Thêm tuỳ chọn bắt đầu hội thoại mới

**Tiêu chí thành công**:

- Duy trì ngữ cảnh hội thoại qua nhiều lượt
- Agent xử lý đúng câu hỏi follow-up
- Agent quyết định khi nào tìm kiếm dựa trên lịch sử hội thoại
- Thể hiện rõ ràng trạng thái hội thoại
- Có tuỳ chọn reset hội thoại

**Gợi ý**:

```python
# 1. Create retrieval tool as in Challenge 1

# 2. Create agent with create_agent()

# 3. Initialize empty message history list
conversation_history: list[HumanMessage | AIMessage] = []

# 4. For each user question:
#    - Add new HumanMessage with user input to history
#    - Invoke agent with full message history
#    - Display agent's response
#    - Add agent's response to history
#    - Continue conversation loop

conversation_history.append(HumanMessage(content=user_input))

response = agent.invoke({
    "messages": list(conversation_history),
})

agent_message = response["messages"][-1]
conversation_history.append(AIMessage(content=agent_message.content))

# 5. Handle special commands:
#    - "exit" or "quit" to end conversation
#    - "reset" to clear history and start fresh

# 6. The agent will autonomously:
#    - Understand context from conversation history
#    - Decide when to search documents
#    - Answer follow-up questions intelligently
```

---

## Checklist nộp bài

Trước khi hoàn thành chương này:

- [ ] Thử thách: Personal knowledge base agentic RAG hoạt động
- [ ] Agent thể hiện khả năng ra quyết định thông minh (tìm kiếm khi cần, trả lời trực tiếp khi có thể)
- [ ] Bonus: Conversational agentic RAG duy trì ngữ cảnh (tuỳ chọn)

---

## Lời giải

Lời giải có sẵn trong thư mục [`solution/`](./solution/). Hãy thử tự làm trước!

- [`knowledge_base_rag.py`](./solution/knowledge_base_rag.py) - Lời giải thử thách chính
- [`conversational_rag.py`](./solution/conversational_rag.py) - Lời giải thử thách Bonus

**Lưu ý**: Lời giải cung cấp dùng cách tiếp cận agentic RAG hiện đại với `create_agent()`. Để so sánh với pattern traditional RAG, xem ví dụ code trong thư mục [`code/`](./code/).

**Ví dụ bổ sung**: Xem thư mục [`samples/`](./samples/) để có thêm ví dụ bao gồm agentic RAG có trích dẫn, agentic RAG đa nguồn, và kỹ thuật hybrid search!

---

## Cần trợ giúp?

- **Kiến thức cơ bản về Agentic RAG**: Xem lại `02_agentic_rag.py`
- **Kiến thức nền tảng về agent**: Xem lại [Getting Started with Agents](../05-agents/README.md)
- **Retrieval tool**: Xem `citation_rag.py` trong samples
- **Câu hỏi bất kỳ**: Dùng [Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) trong GitHub Copilot
- **Vẫn còn vướng**: Tham gia [cộng đồng Discord](https://aka.ms/foundry/discord)

---

## Bước tiếp theo

Chúc mừng! Bạn đã hoàn thành chương Agentic RAG. 🎉

Giờ bạn đã biết cách:

- ✅ Làm việc với LLM bằng prompt và mảng message
- ✅ Quản lý lịch sử hội thoại và state
- ✅ Tạo và dùng tool với function calling
- ✅ Xây dựng agent tự trị biết ra quyết định
- ✅ Xử lý tài liệu và thực hiện semantic search
- ✅ Xây dựng hệ thống agentic RAG thông minh truy xuất thông tin

### Học tiếp

- Triển khai agent LangChain của bạn trong [Deploy LangChain Agents to Microsoft Foundry](../09-deploy-to-microsoft-foundry/README.md)
- Khám phá [LangChain Python Documentation](https://python.langchain.com/)
- Xây dựng ứng dụng agentic của riêng bạn
- Thử nghiệm với các vector store khác nhau (Pinecone, Chroma, Weaviate)
- Thử pattern agent mở rộng và hệ thống multi-agent
- Tích hợp MCP server để mở rộng khả năng
- Chuẩn bị agent của bạn để hosting production

---

Làm rất tốt việc xây dựng hệ thống agentic RAG! 🎉

Tiếp tục xây dựng và khám phá!
