# Deploy LangChain Agents to Microsoft Foundry (Triển khai Agent LangChain lên Microsoft Foundry)

Trong chương tuỳ chọn này, bạn sẽ lấy agent LangChain đã xây trong [Building Agentic RAG Systems](../08-agentic-rag-systems/README.md) và triển khai nó lên **Microsoft Foundry** như một hosted agent. Bạn sẽ giữ nguyên pattern `create_agent()`, cấu hình quyền truy cập model, test giao thức hosting ở local, và triển khai code lên một Foundry project có sẵn.

Chương này theo sát [Microsoft Foundry hosted agent quickstart](https://learn.microsoft.com/azure/foundry/agents/quickstarts/quickstart-hosted-agent). Nó giả định bạn đã có Foundry project và model deployment. Nếu chưa, dùng quickstart để tạo chúng, rồi quay lại Bước 2.

## Yêu cầu trước

- Đã hoàn thành [Building Agentic RAG Systems](../08-agentic-rag-systems/README.md)
- Đã hoàn thành thiết lập local trong [Course Setup](../00-course-setup/README.md)
- Một Foundry project với chat model đã deploy, như `gpt-4.1` hoặc `gpt-5-mini`, và embedding model, như `text-embedding-3-small`
- Quyền **Foundry Project Manager** trên project đó để triển khai hosted-agent
- Python 3.13 trở lên để triển khai hosted agent từ source code
- [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) với extension `azure.ai.agents`

> **Xác thực**: `AI_API_KEY` cấu hình lệnh gọi model trong ứng dụng, trong khi `azd auth login` xác thực deployment CLI ở Bước 5.

> **Dùng virtual environment có sẵn**: Trước khi chạy lệnh trong chương này, đảm bảo Python virtual environment bạn tạo trong [Course Setup](../00-course-setup/README.md#bước-2-clone-repository) đang active. Đừng tạo môi trường riêng cho Chương 09. Trong GitHub Codespaces, `.venv` tự động được kích hoạt. Với phát triển local, kích hoạt lại môi trường từ thư mục gốc repository nếu cần:
>
> **Windows:** `venv\Scripts\activate`
>
> **macOS hoặc Linux:** `source venv/bin/activate`

## 🎯 Mục tiêu học tập

Kết thúc chương này, bạn sẽ có thể:

- ✅ Giải thích hosted agent của Microsoft Foundry cung cấp gì
- ✅ Bọc graph `create_agent()` của LangChain để triển khai hosted
- ✅ Dùng semantic search dựa trên embedding trong hosted agentic RAG tool
- ✅ Cấu hình xác thực model bằng Foundry API key
- ✅ Cấu hình và test hosted agent ở local
- ✅ Triển khai trực tiếp lên Foundry project có sẵn bằng `azd`
- ✅ Xác minh và xử lý sự cố triển khai

---

## 📌 Bạn sẽ xây dựng gì

Ví dụ này nhúng vài ghi chú khoá học vào `InMemoryVectorStore` của LangChain và tìm kiếm chúng bằng semantic similarity. Microsoft Foundry cung cấp managed runtime, hosted endpoint, lịch sử deployment, và log cho agent đã triển khai.

Các file của chương này:

- **Project**: [`code/`](./code/)
- **Code**: [`code/main.py`](./code/main.py)
- **Cấu hình hosted agent**: [`code/azure.yaml`](./code/azure.yaml)
- **Dependency deployment**: [`code/requirements.txt`](./code/requirements.txt)
- **Mẫu biến môi trường**: [`.env.example`](../.env.example) tại thư mục gốc repository

Code hosting và xác thực quan trọng:

```python
import os
from pathlib import Path

from dotenv import load_dotenv
from langchain.agents import create_agent
from langchain_core.tools import tool
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_azure_ai.agents.hosting import ResponsesHostServer


load_dotenv()


def build_chat_model():
    return ChatOpenAI(
        model=os.getenv("AI_MODEL"),
        base_url=os.getenv("AI_ENDPOINT"),
        api_key=os.getenv("AI_API_KEY"),
    )


def build_embeddings_model():
    return OpenAIEmbeddings(
        base_url=os.getenv("AI_ENDPOINT"),
        model=os.getenv("AI_EMBEDDING_MODEL", "text-embedding-3-small"),
        api_key=os.getenv("AI_API_KEY"),
    )


def build_agent():
    vector_store = InMemoryVectorStore.from_documents(
        COURSE_NOTES,
        build_embeddings_model(),
    )

    @tool
    def search_course_notes(query: str) -> str:
        results = vector_store.similarity_search(query, k=3)
        return "\n\n".join(
            f"[{document.metadata['source']}]: {document.page_content}"
            for document in results
        )

    return create_agent(
        build_chat_model(),
        tools=[search_course_notes],
        system_prompt="You are a helpful agentic RAG assistant...",
    )


port = int(os.getenv("PORT", "8088"))
ResponsesHostServer(build_agent()).run(port=port)
```

`ChatOpenAI` và `OpenAIEmbeddings` đều dùng `AI_ENDPOINT` và `AI_API_KEY` cho request model. Ghi chú khoá học được embed khi agent khởi động, và `similarity_search()` truy xuất ba ghi chú gần nhất với query của tool. `ResponsesHostServer` expose `/responses` bằng giao thức Foundry dùng để gửi input cho agent và nhận output của nó.

---

## Bước 1: Chuẩn bị Agent LangChain của bạn để Hosting

Mở [`main.py`](./code/main.py) và tìm `search_course_notes`. Nó theo pattern của Chương 8: `InMemoryVectorStore.from_documents()` embed `COURSE_NOTES`, và tool chạy `similarity_search()` khi agent quyết định retrieval là cần thiết.

Chạy ví dụ mẫu không thay đổi trước. Sau khi hoạt động, bạn có thể thêm tài liệu của riêng bạn hoặc thay `InMemoryVectorStore` bằng vector store bền vững. Tool, metadata tài liệu, logic retrieval, và system prompt của bạn có thể thay đổi mà không cần đổi `build_chat_model()` hay wrapper `ResponsesHostServer`.

---

## Bước 2: Cấu hình Foundry Project của bạn

Chương 09 dùng cùng file `.env` tại thư mục gốc repository cho cả test local và deployment.

Nếu bạn chưa tạo `.env` gốc, tạo nó từ [`.env.example`](../.env.example):

**Windows PowerShell:**

```powershell
Copy-Item .env.example .env
```

**macOS hoặc Linux:**

```bash
cp .env.example .env
```

Mở `.env` gốc và thêm bốn giá trị deployment của Chương 09:

| Nơi tìm giá trị                                           | Biến                                                        |
| ----------------------------------------------------------------- | ---------------------------------------------------------------- |
| Foundry portal, project overview                                  | `FOUNDRY_PROJECT_ENDPOINT`                                       |
| Azure portal, tổng quan resource Foundry và project JSON view | `AZURE_SUBSCRIPTION_ID`, `AZURE_AI_PROJECT_ID`, `AZURE_LOCATION` |

`FOUNDRY_PROJECT_ENDPOINT` là project endpoint kết thúc bằng `/api/projects/<project>`. Repository bỏ qua `.env` gốc (gitignore); `main.py` load nó để test local, và Bước 5 import nó vào môi trường azd.

---

## Bước 3: Xác nhận Dependency

Không cần cài đặt thêm nếu bạn đã hoàn thành [Course Setup](../00-course-setup/README.md). File [`code/requirements.txt`](./code/requirements.txt) của Chương 09 khớp với [`requirements.txt`](../requirements.txt) tại thư mục gốc repository và được bao gồm để Foundry có thể cài cùng dependency trong lúc triển khai.

---

## Bước 4: Test Hosted Wrapper ở Local

Sau khi hoàn thành file `.env` gốc ở Bước 2, chạy host từ thư mục source:

```bash
cd 09-deploy-to-microsoft-foundry/code
python main.py
```

Host lắng nghe tại `http://localhost:8088/responses`. Ở terminal khác, gửi một request test.

**PowerShell:**

```powershell
$body = @{
    input = "When was LangChain created?"
    stream = $false
} | ConvertTo-Json

$response = Invoke-RestMethod `
    -Uri http://localhost:8088/responses `
    -Method Post `
    -Body $body `
    -ContentType "application/json"

$response.output |
    Where-Object { $_.type -eq "message" } |
    ForEach-Object { $_.content } |
    Where-Object { $_.type -eq "output_text" } |
    ForEach-Object { $_.text }
```

**Bash:**

```bash
curl -sS -H "Content-Type: application/json" \
    -X POST http://localhost:8088/responses \
    -d '{"input":"When was LangChain created?","stream":false}'
```

Nếu response nói LangChain được tạo năm 2022, embedding search, xác thực model, và hosted Responses wrapper đang hoạt động. Bấm **Ctrl+C** trong terminal server khi hoàn thành test local.

---

## Bước 5: Triển khai lên Foundry Project có sẵn bằng `azd`

Thư mục [`code/`](./code/) đã là một azd hosted-agent project. `AI_ENDPOINT` cấu hình agent đang chạy, nhưng không báo cho `azd` biết triển khai ở đâu. Để triển khai code trực tiếp, azd yêu cầu riêng subscription ID, project endpoint, resource ID đầy đủ, và Azure region đã thu thập ở Bước 2.

> **Project có sẵn**: Đừng chạy `azd provision` khi Foundry project và model deployment đã tồn tại. Với `infra.provider: microsoft.foundry`, provisioning có thể tạo resource group, Foundry resource, và project mới. Nếu `azd deploy` gợi ý chạy `azd provision`, trước tiên xác nhận cả bốn giá trị deployment bên dưới đã được thiết lập; gợi ý chung chung cũng có thể nghĩa là `AZURE_SUBSCRIPTION_ID` bị thiếu.

Cài extension agent, đăng nhập, và tạo môi trường azd local:

```bash
cd 09-deploy-to-microsoft-foundry/code
azd ext install azure.ai.agents
azd auth login
azd env new langchain-course # Or other name you prefer
```

Nếu môi trường đã tồn tại, chọn nó thay thế:

```bash
azd env select langchain-course # Or name of your existing environment
```

Load giá trị deployment và runtime trực tiếp từ file `.env` gốc vào môi trường azd đang active. Vì bạn đang ở thư mục `code` của Chương 09, file gốc nằm hai thư mục phía trên. Giữ `/openai/v1` ngoài `FOUNDRY_PROJECT_ENDPOINT`, và bao gồm resource ID project đầy đủ trong `AZURE_AI_PROJECT_ID`:

```bash
azd env set --file ../../.env
```

Xác minh azd resolve đúng project trước khi triển khai:

```bash
azd ai project show --output json
azd env get-values
```

Kiểm tra `azd ai project show` in ra đúng project endpoint không có `/openai/v1` và resolve nó từ môi trường azd đang active thay vì global config. Trong `azd env get-values`, xác nhận `AZURE_SUBSCRIPTION_ID`, `FOUNDRY_PROJECT_ENDPOINT`, `AZURE_AI_PROJECT_ID`, và `AZURE_LOCATION` đều được thiết lập. Sau đó triển khai source trực tiếp. Vì `codeConfiguration` có mặt trong `azure.yaml`, Foundry thực hiện remote build dùng `code/requirements.txt`:

```bash
azd deploy --no-prompt
azd ai agent show --output json
azd ai agent invoke "Explain agentic RAG in one paragraph."
```

Triển khai thành công tạo ra một agent version bất biến. `azd ai agent show` nên báo trạng thái `active` hoặc `deployed` và liệt kê Responses endpoint.

> **⚠️ Dọn dẹp**: Hosted agent và lệnh gọi model có thể phát sinh chi phí Azure. Xoá agent và resource bạn chỉ tạo cho bài tập này khi hoàn thành.

---

## 🔧 Xử lý sự cố

| Vấn đề                                       | Kiểm tra gì                                                                                                                                                                                                                                                           |
| --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `azd deploy` yêu cầu `azd provision`         | Với project có sẵn, trước tiên kiểm tra `AZURE_SUBSCRIPTION_ID`. Cũng xác nhận `FOUNDRY_PROJECT_ENDPOINT`, `AZURE_AI_PROJECT_ID`, và `AZURE_LOCATION`; đừng chạy `azd provision` trừ khi bạn muốn tạo hoặc cập nhật hạ tầng.                                     |
| `agent.yaml` hoặc `agent.yml` không tồn tại    | Các azd extension đã cài có thể không hỗ trợ cấu hình agent inline trong `azure.yaml`. Chạy `azd extension upgrade --all`, rồi thử lại `azd deploy`.                                                                                                              |
| `AZURE_SUBSCRIPTION_ID` chưa được thiết lập            | Thiết lập nó tường minh bằng `azd env set AZURE_SUBSCRIPTION_ID "<subscription-id>"`. azd không tự suy ra từ phần subscription trong `AZURE_AI_PROJECT_ID`.                                                                                                      |
| `no Foundry project endpoint resolved`        | Chọn đúng môi trường azd và thiết lập `FOUNDRY_PROJECT_ENDPOINT` không có `/openai/v1`.                                                                                                                                                                            |
| `Microsoft Foundry project ID is required`    | Thiết lập `AZURE_AI_PROJECT_ID` thành Azure Resource Manager ID đầy đủ của project, bắt đầu bằng `/subscriptions/`.                                                                                                                                                            |
| `AZURE_LOCATION is not set`                   | Thiết lập `AZURE_LOCATION` thành Azure region của Foundry resource có sẵn, như `swedencentral`.                                                                                                                                                                     |
| Request model trả về `401` hoặc `403`          | Xác nhận `AI_API_KEY` và `AI_ENDPOINT` đến từ cùng một Foundry resource.                                                                                                                                                                                        |
| `DeploymentNotFound` hoặc `ResourceNotFound`    | Kiểm tra `AI_MODEL` và `AI_EMBEDDING_MODEL` là tên deployment và `AI_ENDPOINT` thuộc cùng Foundry resource.                                                                                                                                                        |
| Request embedding trả về `404`               | Xác nhận `AI_ENDPOINT` kết thúc bằng `/openai/v1` và `AI_EMBEDDING_MODEL` là tên embedding deployment trên resource đó.                                                                                                                                           |
| `Conversation '<id>' not found` trả về `404` | `azd ai agent invoke` có thể tái sử dụng session và conversation ID đã lưu local sau khi remote state hết hạn hoặc thay đổi. Chạy `azd ai agent invoke --new-session --new-conversation "<message>"` một lần để reset cả hai; các lần invoke sau có thể dùng lệnh thông thường. |
| Request local báo connection refused      | Giữ `python main.py` đang chạy và xác nhận port `8088` khả dụng.                                                                                                                                                                                                                |
| Deployment mới tạm thời chưa sẵn sàng     | Đợi `azd ai agent show` báo `active` hoặc `deployed`, rồi invoke lại.                                                                                                                                                                                                    |

---

## 🎓 Điểm chính cần nhớ

- Bạn có thể triển khai graph `create_agent()` của LangChain lên Microsoft Foundry như một hosted agent
- `OpenAIEmbeddings` và `InMemoryVectorStore` cung cấp semantic retrieval mà không cần đổi hosted-agent wrapper
- `ResponsesHostServer` expose graph qua Foundry Responses protocol
- `AI_ENDPOINT` và `AI_API_KEY` cấu hình quyền truy cập model cho cả hai LangChain client
- `AZURE_SUBSCRIPTION_ID`, `FOUNDRY_PROJECT_ENDPOINT`, `AZURE_AI_PROJECT_ID`, và `AZURE_LOCATION` xác định deployment target, trong khi `AI_ENDPOINT` dùng cho model inference
- Chương 09 dùng `.env` gốc của repository cho cả test local và cấu hình `azd`
- Project có sẵn có thể được nhắm tới bằng `azd deploy` mà không cần chạy `azd provision`

---

## 🏆 Bài tập

Sẵn sàng luyện tập chưa? Hoàn thành thử thách deployment trong [assignment.md](./assignment.md)!

---

## 🗺️ Điều hướng

[← Trước: Agentic RAG Systems](../08-agentic-rag-systems/README.md) | [Về trang chính](../README.md)

---

## 🎉 Chúc mừng!

Bạn đã hoàn thành khoá học LangChain for Beginners và học được cách đưa agent LangChain local lên tận Microsoft Foundry hosted agent.

---

## 💬 Có thắc mắc?

[![Microsoft Foundry Discord](https://img.shields.io/badge/Discord-Azure_AI_Foundry_Community_Discord-blue?style=for-the-badge&logo=discord&color=5865f2&logoColor=fff)](https://aka.ms/foundry/discord)
