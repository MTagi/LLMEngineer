# LangChain for Beginners - Một khoá học

<img src="images/lc-beginners-cover.png" alt="LangChain Cover Image" width="800"/>

Khoá học dạy mọi thứ bạn cần biết để bắt đầu xây dựng AI Agent với LangChain

[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/microsoft/langchain-for-beginners/blob/main/LICENSE)
[![GitHub contributors](https://img.shields.io/github/contributors/microsoft/langchain-for-beginners.svg)](https://github.com/microsoft/langchain-for-beginners/graphs/contributors/)
[![GitHub issues](https://img.shields.io/github/issues/microsoft/langchain-for-beginners.svg)](https://github.com/microsoft/langchain-for-beginners/issues/)
[![GitHub pull-requests](https://img.shields.io/github/issues-pr/microsoft/langchain-for-beginners.svg)](https://github.com/microsoft/langchain-for-beginners/pulls/)

[![GitHub watchers](https://img.shields.io/github/watchers/microsoft/langchain-for-beginners.svg?style=social&label=Watch)](https://github.com/microsoft/langchain-for-beginners/)
[![GitHub forks](https://img.shields.io/github/forks/microsoft/langchain-for-beginners.svg?style=social&label=Fork)](https://github.com/microsoft/langchain-for-beginners/)
[![GitHub stars](https://img.shields.io/github/stars/microsoft/langchain-for-beginners.svg?style=social&label=Star)](https://github.com/microsoft/langchain-for-beginners/)

## 🦜🔗 Chào mừng

Chào mừng bạn đến với **LangChain for Beginners**! Khoá học này bao quát các kiến thức nền tảng để xây dựng ứng dụng với LangChain và Python. Khoá học gồm [**10 chương**](#-mục-lục), mỗi chương tập trung vào một khái niệm cụ thể. Để **bắt đầu** ngay, bạn có thể vào chương [course-setup](/00-course-setup/), nhưng bạn nên đọc phần tổng quan bên dưới trước.

### Bạn sẽ học và xây dựng được gì

Khoá học đưa bạn từ con số 0 đến khả năng xây dựng các ứng dụng AI mạnh mẽ:

- **Conversational AI (AI hội thoại)** - Xây dựng chatbot hiểu ngữ cảnh với phản hồi streaming và hành vi tuỳ biến
- **Semantic Search (Tìm kiếm ngữ nghĩa)** - Tạo hệ thống tìm kiếm hiểu ý nghĩa, không chỉ dựa vào từ khoá
- **Function Calling & Tools (Gọi hàm & công cụ)** - Cho AI khả năng dùng tool và trích xuất dữ liệu có cấu trúc
- **Autonomous Agents (Agent tự trị)** - Xây dựng agent biết suy luận, ra quyết định và tự chọn tool
- **MCP Integration (Tích hợp MCP)** - Kết nối AI với các dịch vụ bên ngoài bằng chuẩn Model Context Protocol
- **Agentic RAG** - Xây dựng hệ thống hỏi-đáp thông minh, nơi agent tự quyết định khi nào cần tra cứu knowledge base
- **Microsoft Foundry Deployment (Triển khai lên Microsoft Foundry)** - Triển khai agent LangChain của bạn thành hosted agent bằng VS Code hoặc Azure Developer CLI

**Sau khi hoàn thành**, bạn sẽ hiểu vững về LangChain và biết cách xây dựng các ứng dụng AI thực tế có thể triển khai được!

> **Cách tiếp cận giảng dạy:** Khoá học đi theo hướng "agent-first" (agent trước). Bạn sẽ học về tool, rồi đến agent, sau đó kết hợp chúng với document retrieval (truy xuất tài liệu) để xây dựng hệ thống agentic RAG. Cách này phản ánh đúng cách các hệ thống AI production hiện đại được xây dựng.

Đừng quên [star (🌟)](https://github.com/microsoft/langchain-for-beginners/stargazers) và [fork repo này](https://github.com/microsoft/langchain-for-beginners/fork) để chạy code.

---

## 📚 Mục lục

Khoá học gồm **10 chương** (setup + 9 chương), mỗi chương xây dựng dựa trên chương trước để dạy bạn LangChain từ gốc rễ. Mỗi chương gồm giải thích khái niệm, ví dụ code chạy được, và bài thử thách thực hành.

| # | Chương | Mô tả | Khái niệm chính |
|---|---------|-------------|--------------|
| 0 | [Course Setup](./00-course-setup/README.md) | Thiết lập môi trường phát triển (local hoặc cloud) | Python, Azure AI Foundry, Codespaces, biến môi trường |
| 1 | [Introduction to LangChain](./01-introduction/README.md) | Hiểu framework và các khái niệm cốt lõi | Kiến thức nền tảng LangChain, lệnh gọi LLM đầu tiên |
| 2 | [Chat Models & Basic Interactions](./02-chat-models/README.md) | Chat model, message, và hội thoại | Các loại message, streaming, xử lý lỗi, temperature |
| 3 | [Prompts, Messages, and Structured Outputs](./03-prompts-messages-outputs/README.md) | Làm việc với prompt, mảng message, và output kiểu an toàn (type-safe) | Message, template, structured output, Pydantic schema |
| 4 | [Function Calling & Tools](./04-function-calling-tools/README.md) | Mở rộng khả năng AI bằng function calling và tool | Pydantic schema, tool binding, type safety |
| 5 | [Getting Started with Agents](./05-agents/README.md) | Xây dựng agent tự trị biết suy luận và chọn tool | ReAct pattern, agent loop, create_agent(), middleware |
| 6 | [Model Context Protocol (MCP)](./06-mcp/README.md) | Kết nối AI với dịch vụ bên ngoài qua chuẩn MCP | MCP server, stdio transport, tích hợp tool, mô hình multi-server |
| 7 | [Documents, Embeddings & Semantic Search](./07-documents-embeddings-semantic-search/README.md) | Load tài liệu, tạo embedding, và xây dựng semantic search | Document loading, chunking, vector embedding, similarity search |
| 8 | [Building Agentic RAG Systems](./08-agentic-rag-systems/README.md) | Xây dựng hệ thống RAG nơi agent thông minh tự quyết định khi nào cần tra cứu tài liệu | Agentic RAG (agent tự quyết định khi nào tìm kiếm), retrieval tool, hỏi-đáp thông minh |
| 9 | [Deploy LangChain Agents to Microsoft Foundry](./09-deploy-to-microsoft-foundry/README.md) | Triển khai agent LangChain thành hosted agent trên Microsoft Foundry | Hosted agent, Responses protocol, triển khai qua VS Code, Azure Developer CLI |

Mỗi chương bao gồm:
- 📖 **Giải thích khái niệm** kèm ví dụ liên hệ thực tế (analogy)
- 💻 **Ví dụ code** bạn có thể chạy ngay
- 🎯 **Thử thách thực hành** để kiểm tra hiểu biết
- 🔑 **Điểm chính cần nhớ** để củng cố kiến thức

Chúng tôi dự định mở rộng khoá học theo thời gian với thêm nhiều chủ đề. Hãy đón chờ cập nhật!

---

## 📋 Yêu cầu trước khi học

Trước khi bắt đầu khoá học, bạn nên nắm vững:

- **Kiến thức nền tảng Python** - Biến, hàm, object, async/await, dùng pip để cài package
- **Khái niệm cơ bản về Generative AI** - Hiểu cơ bản về LLM, prompt, token, được trình bày trong khoá [GenAI for Beginners](https://github.com/microsoft/generative-ai-for-beginners)

### Công cụ cần thiết

- [Python 3.10 trở lên](https://python.org/)
- Tài khoản Github
- Code editor (khuyến nghị [VS Code](https://code.visualstudio.com/)) (nếu chạy khoá học ở local)

## 👫 Gặp gỡ người học khác, giải đáp thắc mắc

Nếu gặp khó khăn hoặc có câu hỏi về xây dựng AI Agent, hãy tham gia kênh Discord LangChain trong [Microsoft Foundry Community Discord](https://aka.ms/langchain-foundry/discord).

## 🙏 Muốn giúp đỡ?

Bạn có góp ý hoặc phát hiện lỗi chính tả/code? [Tạo issue](https://github.com/microsoft/langchain-for-beginners/issues) hoặc [Tạo pull request](https://github.com/microsoft/langchain-for-beginners/pull).


## 📖 Tài nguyên khoá học và mẫu bổ sung

- **[Glossary (Bảng thuật ngữ)](./GLOSSARY.md)** - Định nghĩa đầy đủ mọi thuật ngữ dùng trong khoá học
- **[LangChain Documentation](https://docs.langchain.com/oss/python/langchain/overview)** - Tài liệu chính thức của LangChain để tìm hiểu sâu hơn
- **[LangChain Sales Analysis Agent Sample](https://github.com/Azure-Samples/langchain-agent-python)** - Học cách xây dựng agent phân tích doanh số với LangChain, MCP và PostgreSQL
- **[Email Agent Sample](https://github.com/microsoft/local-email-agent/)** - Học cách xây dựng agent xử lý email với LangChain và MCP, có thể chạy local với phi-4 hoặc deploy lên cloud

---

## Tài nguyên bổ sung

### LangChain
[![LangChain.js for Beginners](https://img.shields.io/badge/LangChain.js%20for%20Beginners-22C55E?style=for-the-badge&labelColor=E5E7EB&color=0553D6)](https://aka.ms/langchainjs-for-beginners?WT.mc_id=m365-94501-dwahlin)
[![LangChain4j for Beginners](https://img.shields.io/badge/LangChain4j%20for%20Beginners-22C55E?style=for-the-badge&&labelColor=E5E7EB&color=0553D6)](https://aka.ms/langchain4j-for-beginners)

---


### Azure / Edge / MCP / Agents

[![AZD for Beginners](https://img.shields.io/badge/AZD%20for%20Beginners-0078D4?style=for-the-badge&labelColor=E5E7EB&color=0078D4)](https://github.com/microsoft/AZD-for-beginners?WT.mc_id=academic-105485-koreyst)
[![Edge AI for Beginners](https://img.shields.io/badge/Edge%20AI%20for%20Beginners-00B8E4?style=for-the-badge&labelColor=E5E7EB&color=00B8E4)](https://github.com/microsoft/edgeai-for-beginners?WT.mc_id=academic-105485-koreyst)
[![MCP for Beginners](https://img.shields.io/badge/MCP%20for%20Beginners-009688?style=for-the-badge&labelColor=E5E7EB&color=009688)](https://github.com/microsoft/mcp-for-beginners?WT.mc_id=academic-105485-koreyst)
[![AI Agents for Beginners](https://img.shields.io/badge/AI%20Agents%20for%20Beginners-00C49A?style=for-the-badge&labelColor=E5E7EB&color=00C49A)](https://github.com/microsoft/ai-agents-for-beginners?WT.mc_id=academic-105485-koreyst)

---

### Chuỗi Generative AI

[![Generative AI for Beginners](https://img.shields.io/badge/Generative%20AI%20for%20Beginners-8B5CF6?style=for-the-badge&labelColor=E5E7EB&color=8B5CF6)](https://github.com/microsoft/generative-ai-for-beginners?WT.mc_id=academic-105485-koreyst)
[![Generative AI (.NET)](https://img.shields.io/badge/Generative%20AI%20(.NET)-9333EA?style=for-the-badge&labelColor=E5E7EB&color=9333EA)](https://github.com/microsoft/Generative-AI-for-beginners-dotnet?WT.mc_id=academic-105485-koreyst)
[![Generative AI (Java)](https://img.shields.io/badge/Generative%20AI%20(Java)-C084FC?style=for-the-badge&labelColor=E5E7EB&color=C084FC)](https://github.com/microsoft/generative-ai-for-beginners-java?WT.mc_id=academic-105485-koreyst)
[![Generative AI (JavaScript)](https://img.shields.io/badge/Generative%20AI%20(JavaScript)-E879F9?style=for-the-badge&labelColor=E5E7EB&color=E879F9)](https://github.com/microsoft/generative-ai-with-javascript?WT.mc_id=academic-105485-koreyst)

---

### Kiến thức nền tảng

[![ML for Beginners](https://img.shields.io/badge/ML%20for%20Beginners-22C55E?style=for-the-badge&labelColor=E5E7EB&color=22C55E)](https://aka.ms/ml-beginners?WT.mc_id=academic-105485-koreyst)
[![Data Science for Beginners](https://img.shields.io/badge/Data%20Science%20for%20Beginners-84CC16?style=for-the-badge&labelColor=E5E7EB&color=84CC16)](https://aka.ms/datascience-beginners?WT.mc_id=academic-105485-koreyst)
[![AI for Beginners](https://img.shields.io/badge/AI%20for%20Beginners-A3E635?style=for-the-badge&labelColor=E5E7EB&color=A3E635)](https://aka.ms/ai-beginners?WT.mc_id=academic-105485-koreyst)
[![Cybersecurity for Beginners](https://img.shields.io/badge/Cybersecurity%20for%20Beginners-F97316?style=for-the-badge&labelColor=E5E7EB&color=F97316)](https://github.com/microsoft/Security-101?WT.mc_id=academic-96948-sayoung)
[![Web Dev for Beginners](https://img.shields.io/badge/Web%20Dev%20for%20Beginners-EC4899?style=for-the-badge&labelColor=E5E7EB&color=EC4899)](https://aka.ms/webdev-beginners?WT.mc_id=academic-105485-koreyst)
[![IoT for Beginners](https://img.shields.io/badge/IoT%20for%20Beginners-14B8A6?style=for-the-badge&labelColor=E5E7EB&color=14B8A6)](https://aka.ms/iot-beginners?WT.mc_id=academic-105485-koreyst)
[![XR Development for Beginners](https://img.shields.io/badge/XR%20Development%20for%20Beginners-38BDF8?style=for-the-badge&labelColor=E5E7EB&color=38BDF8)](https://github.com/microsoft/xr-development-for-beginners?WT.mc_id=academic-105485-koreyst)

---

### Chuỗi Copilot

[![Copilot for AI Paired Programming](https://img.shields.io/badge/Copilot%20for%20AI%20Paired%20Programming-FACC15?style=for-the-badge&labelColor=E5E7EB&color=FACC15)](https://aka.ms/GitHubCopilotAI?WT.mc_id=academic-105485-koreyst)
[![Copilot for C#/.NET](https://img.shields.io/badge/Copilot%20for%20C%23/.NET-FBBF24?style=for-the-badge&labelColor=E5E7EB&color=FBBF24)](https://github.com/microsoft/mastering-github-copilot-for-dotnet-csharp-developers?WT.mc_id=academic-105485-koreyst)
[![Copilot Adventure](https://img.shields.io/badge/Copilot%20Adventure-FDE68A?style=for-the-badge&labelColor=E5E7EB&color=FDE68A)](https://github.com/microsoft/CopilotAdventures?WT.mc_id=academic-105485-koreyst)


---

## Cần trợ giúp?

Nếu gặp khó khăn hoặc có câu hỏi khi xây dựng ứng dụng AI, hãy tham gia:

[![Microsoft Foundry Discord](https://img.shields.io/badge/Discord-Azure_AI_Foundry_Community_Discord-blue?style=for-the-badge&logo=discord&color=5865f2&logoColor=fff)](https://aka.ms/foundry/discord)

Nếu bạn có góp ý sản phẩm hoặc gặp lỗi khi xây dựng, hãy truy cập:

[![Microsoft Foundry Developer Forum](https://img.shields.io/badge/GitHub-Azure_AI_Foundry_Developer_Forum-blue?style=for-the-badge&logo=github&color=000000&logoColor=fff)](https://aka.ms/foundry/forum)

---

## Đóng góp

Dự án này hoan nghênh mọi đóng góp và góp ý. Hầu hết đóng góp yêu cầu bạn đồng ý với Contributor License Agreement (CLA), xác nhận rằng bạn có quyền và thực sự cấp cho chúng tôi quyền sử dụng đóng góp của bạn. Chi tiết tại <https://cla.opensource.microsoft.com>.

Khi bạn gửi pull request, một CLA bot sẽ tự động xác định xem bạn có cần cung cấp CLA hay không và đánh dấu PR phù hợp (vd. status check, comment). Chỉ cần làm theo hướng dẫn của bot. Bạn chỉ cần làm việc này một lần cho tất cả các repo dùng CLA của chúng tôi.

Dự án này tuân theo [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). Để biết thêm thông tin, xem [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) hoặc liên hệ [opencode@microsoft.com](mailto:opencode@microsoft.com) nếu có thắc mắc.

## Nhãn hiệu

Dự án này có thể chứa nhãn hiệu hoặc logo của các dự án, sản phẩm, hoặc dịch vụ. Việc sử dụng hợp lệ nhãn hiệu hoặc logo của Microsoft phải tuân theo [Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/legal/intellectualproperty/trademarks/usage/general). Việc sử dụng nhãn hiệu hoặc logo Microsoft trong các phiên bản chỉnh sửa của dự án này không được gây nhầm lẫn hoặc ngụ ý được Microsoft bảo trợ. Bất kỳ việc sử dụng nhãn hiệu bên thứ ba nào cũng phải tuân theo chính sách của bên thứ ba đó.

---

> 📌 **Ghi chú bản dịch:** Đây là bản dịch tiếng Việt do thành viên tự dịch (không phải bản dịch tự động chính thức của Microsoft), giữ nguyên các thuật ngữ kỹ thuật (LLM, agent, tool, embedding, prompt, token, RAG, chain...). Nội dung gốc tiếng Anh: [microsoft/langchain-for-beginners](https://github.com/microsoft/langchain-for-beginners), giấy phép MIT.
