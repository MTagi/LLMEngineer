# Lộ trình nâng cao: RAG & Agent nâng cao + LLMOps

> Tài liệu này dành cho người đã nắm nền tảng LLM engineering (prompt, embedding, RAG cơ bản,
> function calling, agent, MCP — xem [01-07](../README.md)) và muốn đi sâu hơn để xây hệ thống
> production-grade.
>
> Đánh dấu `[x]` vào từng mục khi đã học/thực hành xong để theo dõi tiến độ.

## Mục lục

1. [Retrieval nâng cao](#1-retrieval-nâng-cao)
2. [Đánh giá RAG có hệ thống](#2-đánh-giá-rag-có-hệ-thống)
3. [Kiến trúc Agent nâng cao](#3-kiến-trúc-agent-nâng-cao)
4. [LLMOps / Production Engineering](#4-llmops--production-engineering)
5. [Dự án thực hành tổng hợp](#5-dự-án-thực-hành-tổng-hợp)

---

## 1. Retrieval nâng cao

Nền tảng để RAG thực sự chính xác và đáng tin cậy trong production, thay vì chỉ demo được.

- [ ] **Chunking nâng cao**
  - Semantic chunking — chia tài liệu theo ranh giới ý nghĩa (dùng embedding để phát hiện chỗ
    "đổi chủ đề") thay vì cắt theo số ký tự cố định
  - Hierarchical chunking — chia nhiều cấp (document → section → paragraph), retrieval ở cấp nhỏ
    nhưng có thể "leo" lên cấp lớn để lấy thêm ngữ cảnh
  - Sliding window với overlap động (điều chỉnh overlap theo mật độ thông tin)
- [ ] **Hybrid search** — kết hợp BM25 (khớp từ khoá chính xác) với vector search (khớp ngữ
  nghĩa). BM25 mạnh với tên riêng, mã số, thuật ngữ chính xác; vector search mạnh với ý nghĩa/diễn
  đạt khác nhau. Kết hợp cả hai thường vượt trội hơn dùng riêng lẻ.
- [ ] **Reranking** — sau khi lấy top-k kết quả thô bằng vector search, dùng cross-encoder (mô
  hình so sánh trực tiếp query-document, chính xác hơn nhưng chậm hơn) để sắp xếp lại. Công cụ:
  `bge-reranker`, Cohere Rerank.
- [ ] **Query transformation**
  - Query rewriting — viết lại câu hỏi user cho rõ ràng/đầy đủ hơn trước khi search
  - HyDE (Hypothetical Document Embeddings) — cho LLM sinh một câu trả lời giả định, rồi embed
    câu trả lời đó để tìm kiếm (thường khớp tốt hơn embed thẳng câu hỏi)
  - Multi-query — sinh nhiều biến thể của cùng một câu hỏi, search song song, gộp kết quả
- [ ] **GraphRAG** — kết hợp knowledge graph (thực thể + quan hệ) với vector search, hữu ích cho
  câu hỏi cần suy luận nhiều bước qua nhiều tài liệu (multi-hop reasoning)
- [ ] **Metadata filtering** — kết hợp lọc theo metadata (ngày, danh mục, quyền truy cập user)
  với similarity search để thu hẹp phạm vi tìm kiếm

## 2. Đánh giá RAG có hệ thống

Phần hầu hết người mới bỏ qua — không đo được thì không biết thay đổi chunking/model/prompt có
thực sự cải thiện hay không.

- [ ] **RAGAS** — framework đo các chỉ số cốt lõi:
  - *Faithfulness*: câu trả lời có bám sát tài liệu truy xuất được không (hay đang "bịa")
  - *Answer relevance*: câu trả lời có thực sự trả lời đúng câu hỏi không
  - *Context precision/recall*: tài liệu truy xuất được có chứa đúng thông tin cần, và có bỏ sót
    thông tin quan trọng không
- [ ] **Công cụ thay thế/bổ sung**: TruLens, DeepEval, Arize Phoenix — có dashboard trực quan để
  theo dõi theo thời gian
- [ ] **Golden test set** — tự xây bộ câu hỏi-đáp chuẩn (hoặc sinh tự động bằng LLM từ tài liệu
  gốc) để so sánh khách quan mỗi khi đổi chunking, embedding model, hay prompt
- [ ] **A/B testing retrieval strategy** — so sánh hybrid search vs vector-only, có reranking vs
  không, trên cùng golden test set

## 3. Kiến trúc Agent nâng cao

- [ ] **Planning pattern** — agent lập kế hoạch nhiều bước trước khi hành động (plan-and-execute),
  khác với ReAct thuần (suy luận từng bước một, không có kế hoạch tổng thể trước)
- [ ] **Reflection / self-critique** — agent tự đánh giá lại output của chính nó (hoặc dùng một
  "critic" LLM riêng) trước khi trả lời final, giúp bắt lỗi sớm
- [ ] **Memory systems**
  - Short-term: lịch sử trong một hội thoại (đã học ở phần cơ bản)
  - Long-term: lưu thông tin quan trọng vào vector store, truy xuất lại ở phiên hội thoại sau
  - Episodic memory: agent nhớ lại các "sự kiện"/tương tác trước đó có liên quan đến task hiện tại
- [ ] **Multi-agent orchestration**
  - Pattern supervisor — một agent điều phối, giao việc cho nhiều agent con chuyên biệt
  - Framework: **LangGraph** (kiểm soát luồng agent dạng graph, tích hợp tốt với LangChain đã
    học), CrewAI, AutoGen, OpenAI Swarm
- [ ] **Human-in-the-loop** — điểm dừng bắt buộc cần con người duyệt trước khi agent thực hiện
  hành động có rủi ro (gửi email, xoá dữ liệu, giao dịch tiền...)
- [ ] **Tool design ở quy mô lớn** — khi có hàng chục/hàng trăm tool, cần chiến lược chọn/nhóm tool
  (tool routing) để không làm agent bối rối vì quá nhiều lựa chọn

## 4. LLMOps / Production Engineering

- [ ] **Serving & tối ưu inference** (nếu tự host model thay vì gọi API)
  - vLLM, TGI (Text Generation Inference) — serving framework tối ưu throughput
  - Quantization (GPTQ, AWQ, bitsandbytes) — giảm kích thước model, chạy được trên GPU nhỏ hơn,
    đổi lại một phần chất lượng
  - KV cache, batching, speculative decoding — kỹ thuật tăng tốc inference
- [ ] **Observability** — trace từng bước agent, đo latency/token/cost mỗi request. Công cụ:
  LangSmith, Langfuse
- [ ] **Guardrails** — chặn prompt injection, validate output trước khi trả về user. Framework:
  NeMo Guardrails, Guardrails AI
- [ ] **Resilience** — retry với exponential backoff, fallback sang model/provider khác khi lỗi,
  rate limiting, circuit breaker
- [ ] **Eval-driven development** — coi prompt như code: version hoá, test regression tự động mỗi
  khi đổi prompt hoặc đổi model
- [ ] **Cost optimization**
  - Model routing — câu hỏi đơn giản dùng model rẻ/nhanh, câu phức tạp mới dùng model mạnh
  - Semantic caching — cache theo *ý nghĩa* câu hỏi (dùng embedding để phát hiện câu hỏi tương tự
    đã hỏi trước đó), không chỉ cache theo chuỗi ký tự giống hệt
  - Prompt caching (cache phần system prompt/context tĩnh để giảm token phải xử lý lại)

## 5. Dự án thực hành tổng hợp

Áp dụng toàn bộ lộ trình trên vào một hệ thống end-to-end:

1. Xây agentic RAG với **hybrid search + reranking**
2. Đo chất lượng bằng **RAGAS** trên golden test set tự tạo
3. Thêm **Langfuse** để trace toàn bộ agent loop (mỗi lần tìm kiếm, mỗi lần gọi tool)
4. Thêm **guardrail** chặn prompt injection cơ bản
5. Thêm **retry + fallback model** để hệ thống không sập khi 1 provider lỗi
6. (Tuỳ chọn) Thêm **semantic caching** để giảm chi phí cho câu hỏi lặp lại

Đây gần như là "capstone project" tổng hợp — hoàn thành được dự án này nghĩa là đã thực sự nắm
vững cả 2 hướng RAG/Agent nâng cao và LLMOps.

---

## Tài liệu tham khảo được nhắc tới

| Chủ đề | Công cụ / Framework |
|---|---|
| Reranking | bge-reranker, Cohere Rerank |
| Đánh giá RAG | RAGAS, TruLens, DeepEval, Arize Phoenix |
| Multi-agent | LangGraph, CrewAI, AutoGen, OpenAI Swarm |
| Serving | vLLM, TGI (Text Generation Inference) |
| Quantization | GPTQ, AWQ, bitsandbytes |
| Observability | LangSmith, Langfuse |
| Guardrails | NeMo Guardrails, Guardrails AI |

---

[← Về lộ trình học chính](../README.md)
