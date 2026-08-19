# Cẩm nang kiến trúc Agentic AI — RAG, Agent, Multi-Agent

> Tài liệu này coi nội dung ban đầu (3 nhóm pattern: RAG, AI Agent, Multi-Agent) là **khung ban
> đầu, không phải danh sách đóng**. Bản chuyên sâu mở rộng từ việc "mô tả pattern" thành một
> **cẩm nang kiến trúc và triển khai production**, bao phủ dữ liệu, retrieval, reasoning, memory,
> orchestration, protocol, security, fault tolerance, evaluation và vận hành.
>
> **Phiên bản này** đã (1) tách "Memory, State và Context Engineering" thành một phần riêng đúng
> như đề cương gốc thay vì gộp vào Single-Agent Architecture, và (2) bổ sung các pattern mới xuất
> hiện trong ngành 2025-2026 (context engineering, Agent Skills, giao thức thanh toán agentic,
> late-interaction retrieval, production operations...) sau một vòng research riêng.

## Định hướng mở rộng tài liệu

Nội dung hoàn chỉnh được tổ chức thành 8 phần lớn cộng 1 phụ lục:

1. Nền tảng kiến trúc Generative AI
2. RAG và Knowledge Architecture
3. Single-Agent Architecture
4. Multi-Agent Architecture
5. Memory, State và Context Engineering
6. MCP, A2A và Agent Interoperability
7. Security, Reliability và Governance
8. Evaluation, Observability và Production Operations
9. Phụ lục — Khung phân loại và tiêu chuẩn viết pattern

Các pattern không hoàn toàn tách biệt. Một hệ thống thực tế có thể đồng thời sử dụng Hybrid RAG,
Parent-Child Retrieval, Planner-Executor, Supervisor, Shared State, Context Compaction, Agent
Skills, MCP và A2A. Vì vậy, tài liệu chỉ rõ pattern nào thuộc **data plane, knowledge plane,
retrieval plane, reasoning plane, action plane, memory/context plane, coordination plane,
integration plane, reliability plane, security plane, evaluation plane và operations plane**
(xem khung phân loại đầy đủ ở [Phần IX](#phần-ix-phụ-lục--khung-phân-loại-và-tiêu-chuẩn-viết-pattern)),
thay vì liệt kê chúng như các lựa chọn ngang hàng.

---

## Mục lục

- [PHẦN I. NỀN TẢNG KIẾN TRÚC](#phần-i-nền-tảng-kiến-trúc)
- [PHẦN II. RAG VÀ KNOWLEDGE ARCHITECTURE](#phần-ii-rag-và-knowledge-architecture)
- [PHẦN III. SINGLE-AGENT ARCHITECTURE](#phần-iii-single-agent-architecture)
- [PHẦN IV. MULTI-AGENT ARCHITECTURE](#phần-iv-multi-agent-architecture)
- [PHẦN V. MEMORY, STATE VÀ CONTEXT ENGINEERING](#phần-v-memory-state-và-context-engineering)
- [PHẦN VI. MCP, A2A VÀ AGENT INTEROPERABILITY](#phần-vi-mcp-a2a-và-agent-interoperability)
- [PHẦN VII. SECURITY, RELIABILITY VÀ GOVERNANCE](#phần-vii-security-reliability-và-governance)
- [PHẦN VIII. EVALUATION, OBSERVABILITY VÀ PRODUCTION OPERATIONS](#phần-viii-evaluation-observability-và-production-operations)
- [PHẦN IX. PHỤ LỤC — KHUNG PHÂN LOẠI VÀ TIÊU CHUẨN VIẾT PATTERN](#phần-ix-phụ-lục--khung-phân-loại-và-tiêu-chuẩn-viết-pattern)

---

## PHẦN I. NỀN TẢNG KIẾN TRÚC

### 1. Phân biệt LLM Application, Workflow, Agent và Multi-Agent

#### 1.1. LLM Application

LLM application đơn giản chỉ thực hiện một hoặc một số lời gọi model theo logic cố định:

```
User Input
    ↓
Prompt Template
    ↓
LLM
    ↓
Response
```

Ứng dụng không tự lập kế hoạch, không tự chọn tool và không thay đổi control flow dựa trên kết
quả trung gian.

#### 1.2. LLM Workflow

Workflow sử dụng LLM bên trong một quy trình được định nghĩa trước:

```
Input
  ↓
Classify
  ↓
Extract
  ↓
Validate
  ↓
Generate
  ↓
Output
```

LLM có thể ra quyết định trong từng bước, nhưng đường đi tổng thể do code hoặc workflow engine
kiểm soát. Workflow phù hợp với nghiệp vụ cần tính dự đoán, khả năng audit và điều kiện thực hiện
rõ ràng. Anthropic phân biệt workflow với agent dựa trên việc control flow được định nghĩa bằng
code hay được model quyết định động.

#### 1.3. AI Agent

Agent có một vòng điều khiển:

```
Goal
  ↓
Observe Current State
  ↓
Plan or Select Action
  ↓
Execute Tool
  ↓
Observe Result
  ↓
Update State
  ↓
Continue / Finish / Escalate
```

Một agent production không chỉ là LLM cộng prompt. Nó cần model, tool, memory, state, policy,
termination condition, retry, checkpoint, evaluation và observability. Google khuyến nghị chỉ
dùng agent khi bài toán thật sự cần quyền tự chủ, quyết định động hoặc xử lý mục tiêu mở; với quy
trình xác định rõ, workflow thường đơn giản và ổn định hơn.

#### 1.4. Multi-Agent System

Multi-agent system chứa nhiều agent có phạm vi trách nhiệm và context riêng:

```
User
  ↓
Entry Agent / Orchestrator
  ├─ Documentation Agent
  ├─ Database Agent
  ├─ Coding Agent
  ├─ Planning Agent
  └─ Review Agent
```

Điểm quan trọng không phải chỉ là có nhiều LLM call. Mỗi agent phải có ít nhất một ranh giới kiến
trúc rõ ràng, chẳng hạn:

- Prompt hoặc policy riêng
- Tool riêng
- Knowledge riêng
- Memory hoặc state riêng
- Quyền truy cập riêng
- Lifecycle riêng
- Chủ sở hữu hoặc nhóm phát triển riêng

Multi-agent hữu ích khi cần chuyên môn hóa, quản lý context, phát triển phân tán hoặc xử lý song
song. Nếu các "agent" chỉ là các prompt nhỏ dùng cùng tool, cùng quyền và cùng state, hệ thống đó
có thể chỉ là một workflow được chia thành nhiều node.

---

## PHẦN II. RAG VÀ KNOWLEDGE ARCHITECTURE

### 2. Kiến trúc tổng thể của RAG

RAG phải được chia thành hai pipeline độc lập.

#### 2.1. Indexing pipeline

```
Data Sources
    ↓
Ingestion
    ↓
Parsing / OCR / Layout Analysis
    ↓
Cleaning / Normalization
    ↓
Document Classification
    ↓
Chunking
    ↓
Metadata and ACL Enrichment
    ↓
Embedding
    ↓
Lexical + Vector + Graph Index
```

#### 2.2. Query pipeline

```
User Query
    ↓
Authentication / Authorization
    ↓
Intent and Query Analysis
    ↓
Query Transformation
    ↓
Retriever Selection
    ↓
Candidate Retrieval
    ↓
Fusion and Reranking
    ↓
Context Assembly
    ↓
Answer Generation
    ↓
Groundedness and Citation Validation
```

Microsoft mô tả RAG như một quá trình kỹ thuật gồm chuẩn bị dữ liệu, chunking, enrichment,
embedding, index configuration, retrieval và evaluation. Việc chỉ tạo vector rồi tìm top-K mới
bao phủ một phần nhỏ của kiến trúc RAG production.

#### 2.3. Sự hội tụ về Agentic RAG (cập nhật 2025-2026)

Ngành đang dịch chuyển rõ rệt từ RAG dạng pipeline cố định ("luôn retrieve rồi generate") sang
**Agentic RAG**: hệ thống không chỉ retrieve một lần mà tự suy luận có cần retrieve không, retrieve
cái gì, và khi nào nên dừng — LLM đóng vai trò một reasoning engine tự lập kế hoạch, thực thi và
lặp lại hành động retrieval. Đây là kiến trúc đang chi phối phần lớn hệ thống RAG production hiện
nay, và khớp với pattern **Adaptive retrieval** (mục 6.9) cùng **Agentic RAG** đã có trong roadmap
học cơ bản của bộ tài liệu này.

### 3. Data ingestion patterns

#### 3.1. Batch ingestion

Dữ liệu được quét và xử lý theo lịch:

```
Scheduler
   ↓
Scan Data Source
   ↓
Detect Changed Documents
   ↓
Reprocess
   ↓
Update Index
```

Phù hợp với:

- User manual
- Policy document
- Wiki nội bộ
- Hồ sơ ít cập nhật

#### 3.2. Event-driven ingestion

```
Document Changed
    ↓
Event Bus
    ↓
Parser
    ↓
Chunk and Embed
    ↓
Index Update
```

Phù hợp với dữ liệu cập nhật liên tục hoặc yêu cầu freshness cao.

#### 3.3. Change Data Capture

CDC theo dõi insert, update và delete trong database:

```
Database Transaction Log
          ↓
        CDC
          ↓
Knowledge Projection
          ↓
Search Index
```

Không nên embed trực tiếp mọi row của database. Cần xây knowledge projection có ý nghĩa nghiệp
vụ, ví dụ kết hợp customer, contract và product thành một record có thể truy xuất.

#### 3.4. Incremental indexing

Mỗi tài liệu và chunk cần checksum:

```
document_hash = hash(normalized_document)
chunk_hash    = hash(normalized_chunk)
```

Incremental indexing chỉ tái xử lý nội dung thay đổi, đồng thời phải xóa vector, lexical term và
graph edge của phiên bản cũ.

#### 3.5. Temporal indexing

Với dữ liệu có phiên bản:

```json
{
  "document_id": "policy-001",
  "version": "4.2",
  "valid_from": "2026-07-01",
  "valid_to": null,
  "is_current": true
}
```

Query "quy định hiện tại" và "quy định tại tháng 1 năm 2025" phải trả hai tập dữ liệu khác nhau.
Version và effective date vì vậy phải là metadata có thể filter, không chỉ xuất hiện trong text.

### 4. Parsing và document understanding

#### 4.1. Layout-aware parsing

PDF, Word và PowerPoint cần được phân tích thành:

- Heading
- Paragraph
- List
- Table
- Figure
- Caption
- Footnote
- Header và footer
- Page number
- Reading order

Nếu chỉ extract plain text, hệ thống có thể ghép nhầm hai cột, đưa header vào giữa câu hoặc tách
caption khỏi hình.

#### 4.2. Table-aware parsing

Bảng nên được lưu dưới nhiều representation:

```json
{
  "table_markdown": "...",
  "table_summary": "...",
  "column_names": ["Product", "Q1", "Q2"],
  "row_entities": ["Product A", "Product B"],
  "source_page": 15
}
```

Có thể search trên summary, column và entity, nhưng khi trả lời phải dùng dữ liệu bảng gốc làm
evidence.

#### 4.3. Multimodal parsing

Với hình, sơ đồ và biểu đồ:

```
Image
 ├─ OCR Text
 ├─ Caption
 ├─ Visual Description
 ├─ Detected Entities
 └─ Link to Parent Section
```

Pipeline có thể sinh mô tả ảnh lúc indexing hoặc retrieve ảnh gốc và gửi cho multimodal model tại
inference. Hai cách có trade-off khác nhau về chi phí indexing, độ chính xác và latency.

#### 4.4. Code-aware parsing

Source code nên chia theo cấu trúc ngôn ngữ:

```
Repository
 └─ Module
     └─ Class
         ├─ Method
         ├─ Docstring
         ├─ Dependencies
         └─ Call Relationships
```

Metadata nên chứa:

- Repository
- Branch
- Commit
- File path
- Symbol
- Language
- Imported modules
- Caller và callee
- Test liên quan

### 5. Chunking patterns mở rộng

Chunking không chỉ có fixed-size và semantic. Một taxonomy đầy đủ nên gồm:

#### 5.1. Fixed-token chunking

- Đơn giản
- Dễ tạo baseline
- Dễ kiểm soát token
- Có nguy cơ cắt giữa ý

#### 5.2. Sentence-based chunking

Ghép một số câu liên tiếp đến khi đạt giới hạn token.

#### 5.3. Recursive chunking

Ưu tiên chia từ cấu trúc lớn xuống nhỏ:

```
Chapter → Section → Paragraph → Sentence → Token
```

#### 5.4. Structure-aware chunking

Dựa trên heading, article, clause, class, method, API operation hoặc table boundary.

#### 5.5. Semantic chunking

Tạo boundary khi similarity giữa các câu liền kề giảm xuống dưới ngưỡng.

#### 5.6. Proposition chunking

Tách nội dung thành các phát biểu nguyên tử:

```
Original:
LangGraph supports checkpoints, which allow workflows to resume
after interruption and support human-in-the-loop execution.

Propositions:
1. LangGraph supports checkpoints.
2. Checkpoints allow workflows to resume after interruption.
3. Checkpoints support human-in-the-loop execution.
```

Pattern này tăng khả năng match câu hỏi cụ thể, nhưng làm tăng số chunk và có thể mất ngữ cảnh.

#### 5.7. Parent-child chunking

Search trên chunk nhỏ, trả context lớn hơn.

#### 5.8. Sliding-window chunking

Mỗi chunk chứa overlap với chunk trước để giảm khả năng cắt mất thông tin tại boundary.

#### 5.9. Small-to-big retrieval

Retrieve sentence hoặc child chunk, sau đó mở rộng thành paragraph, section hoặc parent document.

#### 5.10. Contextual chunking

Gắn thêm document title, section path và summary trước khi embedding:

```
Document: Multi-Agent User Manual
Section: Fault Tolerance > Checkpoint Recovery
Version: 3.2

<original chunk>
```

#### 5.11. Late chunking

Tạo embedding với nhận thức về context dài hơn trước khi tách representation thành các chunk
nhỏ. Mục tiêu là giúp chunk giữ được thông tin ngữ cảnh từ toàn tài liệu.

#### 5.12. Agentic chunking

LLM hoặc agent quyết định boundary dựa trên loại tài liệu, chủ đề, semantic completeness và mục
tiêu retrieval.

Chunk quá nhỏ làm mất context; chunk quá lớn làm giảm độ sắc nét của embedding và đưa noise vào
prompt. Baseline 512 hoặc 1.024 token có overlap chỉ là điểm bắt đầu, cần được đánh giá bằng bộ
query đại diện cho dữ liệu thật.

### 6. Retrieval patterns mở rộng

#### 6.1. Dense retrieval

Dùng embedding và vector similarity để tìm nội dung gần nghĩa.

#### 6.2. Sparse retrieval

Dùng BM25 hoặc sparse learned representation để tìm term match.

#### 6.3. Hybrid retrieval

Kết hợp sparse và dense:

```
BM25 Results ─────┐
                  ├─ Fusion → Rerank
Vector Results ───┘
```

#### 6.4. Multi-index retrieval

```
Query
 ├─ Document Chunk Index
 ├─ Document Summary Index
 ├─ Title Index
 ├─ Entity Index
 └─ Question Index
```

Một tài liệu có thể được biểu diễn bằng nhiều index để giải quyết các loại query khác nhau.

#### 6.5. Multi-vector retrieval

Mỗi document hoặc chunk có nhiều vector:

```
Document
 ├─ Content Embedding
 ├─ Summary Embedding
 ├─ Title Embedding
 ├─ Question Embedding
 └─ Entity Embedding
```

#### 6.6. Hierarchical retrieval

Search lần lượt:

```
Corpus
  ↓
Relevant Collection
  ↓
Relevant Document
  ↓
Relevant Section
  ↓
Relevant Chunk
```

Pattern này giảm search space và phù hợp với corpus được tổ chức theo sản phẩm, dự án hoặc
domain.

#### 6.7. Federated retrieval

Query nhiều search service độc lập rồi tổng hợp kết quả:

```
Enterprise Query
 ├─ Engineering Search
 ├─ HR Search
 ├─ Product Search
 └─ External Search
```

Mỗi domain có thể dùng embedding model, index và policy riêng.

#### 6.8. Multi-hop retrieval

Kết quả bước trước tạo query cho bước sau:

```
Question
  ↓
Retrieve Entity A
  ↓
Extract Relationship
  ↓
Retrieve Entity B
  ↓
Compose Evidence
```

Phù hợp với câu hỏi cần kết hợp nhiều tài liệu.

#### 6.9. Adaptive retrieval

Hệ thống quyết định:

- Có cần retrieval không
- Dùng BM25, vector hay graph
- Top-K bao nhiêu
- Có cần rerank không
- Có cần tìm kiếm lần nữa không

#### 6.10. Negative retrieval

Hệ thống không chỉ tìm bằng chứng ủng hộ mà còn chủ động tìm:

- Tài liệu mâu thuẫn
- Phiên bản cũ
- Ngoại lệ
- Điều khoản loại trừ
- Bằng chứng phản bác

Các RAG hiện đại đang dịch chuyển từ pipeline retrieval cố định sang kiến trúc adaptive và
agentic, nơi query reformulation, source selection, context filtering và multi-hop evidence được
điều khiển động.

#### 6.11. Late-interaction retrieval (ColBERT-style) — *bổ sung*

Thay vì nén cả document thành **một** vector duy nhất (single-vector embedding), late-interaction
giữ lại embedding ở **cấp token** cho cả query và document, rồi tính độ khớp bằng cách so khớp
token-với-token (thường qua phép "MaxSim") tại thời điểm truy vấn thay vì tại thời điểm index.

```
Query tokens   [q1] [q2] [q3]
                 ╲    │    ╱
Document tokens [d1] [d2] [d3] [d4] ...
                 → MaxSim(qi, dj) cho từng qi, cộng lại thành điểm cuối
```

- **Ưu điểm**: giữ được nhiều thông tin ngữ cảnh hơn dense retrieval single-vector, khớp tinh hơn
  với câu hỏi có nhiều thực thể/điều kiện.
- **Nhược điểm**: chi phí lưu trữ và tính toán cao hơn (mỗi document lưu nhiều vector thay vì một),
  cần thư viện/index chuyên dụng để scale (ColBERT, ColPali/ColQwen cho multimodal).
- **Khi dùng**: corpus vừa/nhỏ cần độ chính xác cao, hoặc dùng như tầng rerank thứ hai sau khi
  dense/hybrid retrieval đã thu hẹp candidate set — tương tự vai trò của cross-encoder reranking
  (mục 9.3) nhưng giữ được nhiều tín hiệu token-level hơn.

### 7. BM25 và lexical search

BM25 đánh giá tài liệu dựa trên:

- Term frequency
- Inverse document frequency
- Term-frequency saturation
- Normalization theo độ dài tài liệu

Hai tham số quan trọng:

- `k1`: mức độ bão hòa của term frequency
- `b`: mức độ normalization theo chiều dài field

OpenSearch dùng Okapi BM25 cho keyword search mặc định; Elasticsearch và Solr sử dụng Lucene,
trong đó BM25 là cơ chế scoring lexical phổ biến.

Chất lượng BM25 phụ thuộc mạnh vào analyzer:

```
Character Filter
    ↓
Tokenizer
    ↓
Lowercase
    ↓
Stop-word Filter
    ↓
Stemming / Lemmatization
    ↓
Synonym Filter
```

Với tài liệu kỹ thuật, cần giữ nguyên:

- Error code
- Class name
- API path
- Version
- Acronym
- Product identifier

Không nên stem hoặc tách sai các token như `CheckpointManager`, `/v1/tasks/cancel` hoặc
`ERR_AGENT_042`.

### 8. Vector index và vector database

#### 8.1. Flat search

So sánh query với mọi vector. Chính xác nhưng chi phí gần tuyến tính theo số vector.

#### 8.2. HNSW

HNSW xây graph nhiều tầng. Tầng cao chứa đường kết nối xa để định vị nhanh vùng phù hợp; tầng
thấp chứa neighborhood dày hơn để tìm top-K chính xác.

Tham số chính:

- `M`
- `efConstruction`
- `efSearch`

HNSW thường cho recall và latency tốt nhưng tiêu tốn RAM để lưu graph.

#### 8.3. IVF

IVF dùng clustering để chia vector space thành các partition.

Tham số chính:

- `nlist`: số cluster
- `nprobe`: số cluster được search

`nprobe` càng cao thì recall càng tốt nhưng query càng chậm.

#### 8.4. Product Quantization

PQ chia vector thành subvector và thay mỗi subvector bằng code của centroid gần nhất. Nó giảm
đáng kể bộ nhớ nhưng chỉ cho khoảng cách xấp xỉ.

#### 8.5. Scalar và binary quantization

- Scalar quantization giảm float32 xuống số bit thấp hơn.
- Binary quantization chuyển representation thành bit.
- Phù hợp khi cần giảm RAM hoặc tăng throughput.
- Thường cần rescoring bằng vector gốc để phục hồi precision.

#### 8.6. Disk-oriented ANN

Dùng SSD để lưu phần lớn vector hoặc graph thay vì giữ toàn bộ trong RAM. Phù hợp với corpus rất
lớn nhưng cần kiểm soát I/O và cache.

#### 8.7. Lựa chọn database

- **FAISS**: thư viện ANN cho local, research và custom service.
- **pgvector**: phù hợp khi metadata và transaction đã nằm trong PostgreSQL.
- **Qdrant**: vector-first, metadata filtering và self-hosting.
- **Milvus**: quy mô lớn, nhiều index và mô hình distributed.
- **Elasticsearch/OpenSearch**: mạnh khi cần BM25, vector, filter và aggregation.
- **Azure AI Search**: phù hợp với Azure, hybrid retrieval, semantic ranking và enterprise
  search.
- **Weaviate**: object schema, vector và hybrid search.

Không nên lựa chọn chỉ dựa vào benchmark vector. Cần đánh giá filtering, update/delete, backup,
replication, index rebuild, multi-tenancy, hybrid search và kỹ năng vận hành của đội ngũ.

### 9. Fusion, reranking và context construction

#### 9.1. Reciprocal Rank Fusion

RRF kết hợp các danh sách theo thứ hạng thay vì cộng trực tiếp BM25 score với cosine score:

```
RRF(d) = Σ 1 / (k + rank_i(d))
```

#### 9.2. Weighted score fusion

Chuẩn hóa score từ từng retriever rồi áp dụng trọng số:

```
final_score =
    w1 × normalized_bm25 +
    w2 × normalized_vector +
    w3 × metadata_score
```

#### 9.3. Cross-encoder reranking

Cross-encoder đọc đồng thời query và candidate chunk để tính relevance sâu hơn.

#### 9.4. LLM reranking

LLM có thể đánh giá:

- Chunk có trả lời trực tiếp không
- Đúng product và version không
- Có phải nguồn chính thức không
- Có chứa ngoại lệ quan trọng không

#### 9.5. Diversity reranking

Maximal Marginal Relevance cân bằng relevance với diversity để tránh top-K chứa nhiều chunk gần
như giống nhau.

#### 9.6. Context compression

Chỉ trích phần liên quan từ chunk lớn trước khi đưa vào prompt.

#### 9.7. Evidence packing

Context builder phải:

- Deduplicate
- Group theo document
- Mở rộng neighboring chunks
- Giữ source ID
- Ưu tiên phiên bản hiện hành
- Tuân thủ token budget
- Đảm bảo ACL
- Phát hiện contradiction

RAG cần được đánh giá tách biệt theo retrieval, reranking, context assembly và generation. Nếu
chỉ đánh giá câu trả lời cuối, nhóm phát triển khó xác định lỗi đến từ chunking, retriever hay
model.

---

## PHẦN III. SINGLE-AGENT ARCHITECTURE

> Memory và Reliability đã được tách sang [Phần V](#phần-v-memory-state-và-context-engineering)
> và [Phần VII](#phần-vii-security-reliability-và-governance) để đúng với đề cương 8 phần gốc.
> Phần này chỉ giữ lại các pattern thuộc **reasoning plane** và **action plane** của một agent
> đơn lẻ.

### 10. Reasoning và planning patterns

#### 10.1. ReAct

```
Observe → Decide → Act → Observe → Continue
```

#### 10.2. Plan-and-Execute

Planner tạo plan đầy đủ, executor thực hiện lần lượt.

#### 10.3. Replanning

Sau mỗi bước, agent kiểm tra plan còn phù hợp không.

#### 10.4. Tree-based exploration

Agent tạo nhiều hướng giải quyết:

```
Problem
 ├─ Approach A
 │   ├─ A1
 │   └─ A2
 ├─ Approach B
 └─ Approach C
```

Một evaluator chọn nhánh tiềm năng.

#### 10.5. Iterative deepening

Agent bắt đầu bằng giải pháp đơn giản, chỉ tăng độ sâu reasoning khi kết quả chưa đạt.

#### 10.6. Constraint-based planning

Plan phải thỏa:

- Deadline
- Cost budget
- Tool permission
- Execution dependency
- Approval requirement
- Data residency

#### 10.7. Goal decomposition

Tách goal thành milestone, task và action:

```
Goal
 ├─ Milestone
 │   ├─ Task
 │   │   ├─ Action
 │   │   └─ Verification
```

Planning, reflection, tool use và multi-agent collaboration là các thành phần nền tảng của
Agentic RAG và agent architecture hiện đại.

### 11. Tool-use patterns

#### 11.1. Direct tool invocation

Agent chọn tool và gọi một lần.

#### 11.2. Tool chaining

Output của tool trước là input cho tool sau.

#### 11.3. Tool router

Một router giới hạn tập tool được expose cho model.

#### 11.4. Tool broker

Broker quản lý discovery, permission, timeout, quota và logging.

#### 11.5. Tool sandbox

Code, file hoặc browser action được thực hiện trong môi trường cô lập.

#### 11.6. Compensating transaction

Khi action thứ ba thất bại, agent chạy action bù:

```
Create Reservation
    ↓
Charge Payment
    ↓ failure
Cancel Reservation
```

#### 11.7. Idempotent tool pattern

Tool write operation phải có idempotency key để retry không tạo dữ liệu trùng.

#### 11.8. Read-before-write

Agent đọc trạng thái hiện tại, kiểm tra precondition rồi mới cập nhật.

#### 11.9. Dry-run và approval

```
Proposed Action
    ↓
Dry Run
    ↓
Impact Report
    ↓
Human Approval
    ↓
Execute
```

Trong production, tool phải được đối xử như API: schema rõ ràng, validate input/output, timeout,
permission, idempotency, audit và giới hạn tác động.

#### 11.10. Mandate-based transaction (thanh toán agentic) — *bổ sung*

Khi tool thực hiện giao dịch có giá trị thật (mua hàng, chuyển tiền), agent không tự "quyết định
và trả tiền" — nó tạo ra một **mandate** (uỷ quyền) có thể kiểm chứng, để bên thứ ba (merchant,
payment network) xác minh độc lập user đã cho phép gì.

```
User Intent
    ↓
Intent Mandate (agent được phép tìm/đề xuất gì)
    ↓
Agent chọn phương án
    ↓
Cart Mandate (nội dung giỏ hàng cụ thể, cần user xác nhận)
    ↓
Payment Mandate (uỷ quyền thanh toán, ký số)
    ↓
Payment Network xác minh mandate → Thực thi
```

Đây là mô hình mà giao thức **Agent Payments Protocol (AP2)** của Google chuẩn hoá (9/2025): ba
mandate — Intent, Cart, Payment — được ký dưới dạng W3C Verifiable Credential, tạo bằng chứng
không thể chối bỏ về việc "user cho phép gì, agent chọn gì, và cái gì đã được charge". Pattern
này áp dụng được ngay cả khi không dùng AP2, miễn là tool giao dịch tuân theo nguyên tắc: **tách
uỷ quyền (mandate) khỏi thực thi (execution)**, và giao dịch giá trị cao luôn cần approval boundary
(xem mục 26.8) trước khi mandate được ký.

---

## PHẦN IV. MULTI-AGENT ARCHITECTURE

### 12. Federated multi-agent

Các agent thuộc nhiều domain hoặc hạ tầng khác nhau:

```
Enterprise Supervisor
 ├─ Local Engineering Agent
 ├─ Remote Finance Agent
 ├─ Partner Agent
 └─ Cloud Research Agent
```

Mỗi agent giữ model, memory, tool và policy riêng. Microsoft mô tả local và remote agent
execution như một mô hình liên bang, cần secure channel, traceability và supervisor coordination.

### 13. Market-based task allocation

Worker đưa ra bid dựa trên:

- Capability
- Cost
- Current load
- Estimated completion time
- Confidence

Coordinator chọn worker có utility tốt nhất.

### 14. Contract-net pattern

```
Manager announces task
        ↓
Agents submit proposals
        ↓
Manager awards contract
        ↓
Selected agent executes
        ↓
Manager validates result
```

### 15. Orchestrator-Worker / Map-reduce agents

```
Large Input
   ↓
Lead Agent (Orchestrator) — phân tích, lập chiến lược
   ↓
Worker Agents (song song) — mỗi worker khám phá 1 hướng độc lập, context riêng
   ↓
Partial Results
   ↓
Synthesis / Reduce Agent — tổng hợp, kèm 1 pass citation riêng
   ↓
Final Result
```

**Ví dụ thực tế đã được ghi nhận (2025):** hệ thống "Research" của Anthropic dùng đúng pattern
này — lead agent lập chiến lược rồi spawn 3-5 subagent chuyên biệt chạy song song, mỗi subagent
có context window, tool và hướng khám phá riêng; kết quả được tổng hợp qua một bước synthesis
tách biệt cộng một pass kiểm tra citation. Cách này vượt trội hơn agent đơn ~90% trên các câu hỏi
nghiên cứu dạng "breadth-first" (cần khám phá nhiều nhánh độc lập, tổng thông tin vượt quá một
context window), nhưng đổi lại chi phí token cao hơn đáng kể (~15 lần so với 1 lượt chat thông
thường) — cần cân nhắc rõ trade-off chi phí/chất lượng trước khi áp dụng cho use case có volume
lớn.

### 16. Committee-of-experts

Router chọn một nhóm expert thay vì chỉ một agent. Judge hoặc synthesizer tổng hợp output.

### 17. Red-team and blue-team

- Blue agent tạo giải pháp.
- Red agent tìm lỗi, rủi ro hoặc attack path.
- Judge xác định vấn đề nào hợp lệ.
- Blue agent sửa kết quả.

### 18. Shared artifact workspace

Agent cộng tác qua file và artifact thay vì chỉ message:

```
Shared Workspace
 ├─ requirements.md
 ├─ architecture.md
 ├─ implementation/
 ├─ test-results.json
 └─ review-comments.json
```

### 19. Event-driven multi-agent

Agent subscribe event:

```
DocumentCreated
    ↓
Extraction Agent
    ↓ DocumentExtracted
Validation Agent
    ↓ DocumentValidated
Indexing Agent
```

Pattern này giảm coupling nhưng cần correlation ID, idempotency và event schema governance.

### 20. Choreography

Không có orchestrator trung tâm. Mỗi agent phản ứng với event và phát event tiếp theo.

### 21. Hybrid orchestration

Kết hợp:

- Serial cho bước phụ thuộc
- Parallel cho kiểm tra độc lập
- Supervisor cho lựa chọn domain
- Handoff cho hội thoại
- Event-driven cho tác vụ nền

Microsoft lưu ý rằng use case phức tạp thường cần kết hợp serial, concurrent và orchestrated
agent types, thay vì cố dùng một pattern duy nhất cho toàn bộ hệ thống.

---

## PHẦN V. MEMORY, STATE VÀ CONTEXT ENGINEERING

> Phần này trước đây bị gộp lẫn vào Single-Agent Architecture. Tách riêng vì memory, state và
> context là ba khái niệm liên quan nhưng không đồng nhất: **memory** là *cái gì được nhớ lại*,
> **state** là *hệ thống lưu nó ở đâu và bằng công nghệ gì*, còn **context engineering** là *cách
> chọn, nén và sắp xếp thông tin đưa vào context window tại một thời điểm cụ thể*.

### 22. Memory patterns

#### 22.1. Working memory

State của task hiện tại:

- Plan
- Tool results
- Current step
- Intermediate artifact
- Errors

#### 22.2. Conversation memory

Message giữa user và agent trong session.

#### 22.3. Summary memory

Tóm tắt lịch sử để giảm token.

#### 22.4. Semantic memory

Lưu fact và knowledge tổng quát.

#### 22.5. Episodic memory

Lưu event hoặc task đã xảy ra:

- Task
- Action
- Outcome
- Feedback
- Lesson

#### 22.6. Procedural memory

Lưu cách làm, rule, workflow hoặc reusable skill.

#### 22.7. Entity memory

Lưu thông tin theo user, customer, product, project hoặc organization.

#### 22.8. Artifact memory

Lưu file, report, code, query result và document được tạo trong quá trình làm việc.

#### 22.9. Memory consolidation

Nhiều memory ngắn được tổng hợp thành memory ổn định hơn.

#### 22.10. Memory forgetting

Xóa hoặc giảm trọng số memory theo:

- Thời gian
- Độ liên quan
- Privacy policy
- User request
- Data retention
- Contradiction với thông tin mới

Memory không đồng nghĩa với vector database. Một kiến trúc tốt thường kết hợp state store,
relational database, object storage, vector index và event log.

### 23. State — *bổ sung*

State là hạ tầng lưu trữ đằng sau memory và tiến trình agent, cần được thiết kế tách biệt khỏi
"trí nhớ" theo nghĩa nội dung.

#### 23.1. Session state vs durable state

- **Session state**: sống trong một phiên tương tác, có thể mất khi process restart (working
  memory, biến tạm trong 1 lượt chạy).
- **Durable state**: phải sống sót qua restart, deploy, hoặc failover — cần persist vào database
  hoặc state store bên ngoài process của agent.

#### 23.2. State store patterns

- **Key-value / document store**: lưu state theo `task_id`/`session_id`, đơn giản, dễ scale
  ngang.
- **Event-sourced state**: lưu state dưới dạng chuỗi sự kiện thay vì snapshot; state hiện tại =
  replay toàn bộ event. Cho phép audit trail đầy đủ và time-travel debugging, đổi lại phức tạp
  hơn khi cần đọc nhanh.
- **Graph-based state**: dùng cho agent có nhiều bước phụ thuộc phi tuyến (branching, merge) —
  phù hợp với framework theo mô hình state machine dạng đồ thị (ví dụ LangGraph), nơi mỗi node
  đọc/ghi vào một state object dùng chung.

#### 23.3. State versioning và migration

State schema của agent (cấu trúc plan, tool result, metadata) sẽ thay đổi qua thời gian khi agent
được nâng cấp. Cần:

- Gắn `schema_version` vào mỗi state record
- Có migration path rõ ràng khi đọc state cũ bằng code mới
- Không để một agent version mới đọc nhầm state được ghi bởi version cũ mà không kiểm tra tương
  thích

#### 23.4. State và checkpoint (liên hệ Phần VII)

State store là nơi checkpoint (mục 30.1) thực sự được ghi vào. Checkpoint/resume, retry, và
saga pattern (Phần VII) đều phụ thuộc trực tiếp vào việc state được thiết kế đúng — nếu state
không đủ chi tiết để tái tạo lại điểm dừng, agent không thể resume chính xác.

### 24. Context Engineering — *bổ sung*

Context engineering là tập kỹ thuật quản lý **những gì thực sự nằm trong context window** tại
mỗi lượt gọi model — phân biệt với memory (nội dung được lưu lâu dài) và với prompt engineering
(cách viết một prompt tĩnh). Ba kỹ thuật cốt lõi, theo hướng dẫn kỹ thuật của Anthropic:

#### 24.1. Compaction

Khi hội thoại/task gần chạm giới hạn context, hệ thống tóm tắt toàn bộ nội dung đã có, rồi khởi
tạo lại context mới chỉ chứa bản tóm tắt đó thay vì toàn bộ lịch sử:

```
Context gần đầy
    ↓
Summarize toàn bộ conversation/task hiện tại
    ↓
Khởi tạo context mới = summary + (các thông tin bắt buộc phải giữ nguyên, vd. system prompt)
    ↓
Tiếp tục task với context đã "nén"
```

Phù hợp nhất với task cần nhiều lượt qua lại liên tục (hội thoại dài, task tương tác cao).

#### 24.2. Structured note-taking (bộ nhớ ngoài context)

Agent chủ động ghi chú ra ngoài context window (file, scratchpad, task list persistent) thay vì
giữ toàn bộ thông tin trong context. Khi cần, agent đọc lại note thay vì phụ thuộc vào context đã
bị compact hoặc cắt bớt. Phù hợp với task lặp lại nhiều vòng, có các mốc rõ ràng (ví dụ: các bước
trong một dự án phát triển phần mềm nhiều giai đoạn).

#### 24.3. Sub-agent context isolation

Khi một subtask được giao cho subagent, subagent đó chạy trong **context window riêng**, tách
biệt hoàn toàn khỏi context của orchestrator. Orchestrator chỉ nhận **kết quả cuối cùng**, không
nhận toàn bộ reasoning/tool-call trung gian của subagent:

```
Orchestrator context
 └─ Subagent A: context riêng (không lộ về Orchestrator)
     → Chỉ trả kết quả cuối
 └─ Subagent B: context riêng
     → Chỉ trả kết quả cuối
```

Nhờ vậy context của orchestrator **không phình theo độ phức tạp của task** — đây là lý do cốt lõi
khiến pattern multi-agent (mục 15, 22.1) mở rộng tốt hơn agent đơn khi task đủ phức tạp.

#### 24.4. Just-in-time context loading

Thay vì nạp toàn bộ tài liệu/tool schema vào context ngay từ đầu, agent chỉ load thông tin **khi
thực sự cần** (ví dụ: chỉ đọc nội dung file khi chuẩn bị chỉnh sửa file đó, thay vì đọc trước toàn
bộ codebase). Giảm token lãng phí và giảm nhiễu khiến model mất tập trung vào phần liên quan.

#### 24.5. Context rot (failure mode cần theo dõi)

**Context rot** là hiện tượng chất lượng phản hồi của agent suy giảm khi context đã dài ra —
ngay cả khi tổng số token *vẫn còn trong giới hạn cửa sổ context* của model. Đây không phải lỗi
do vượt giới hạn cứng, mà là suy giảm khả năng "chú ý" đúng phần liên quan khi lượng thông tin
không liên quan tích luỹ quá nhiều. Cách giảm thiểu: áp dụng compaction/note-taking chủ động
*trước khi* context quá dài (không đợi đến khi chạm limit), và ưu tiên just-in-time loading thay
vì nạp trước toàn bộ. Nên bổ sung theo dõi độ dài context và chất lượng output như một chỉ số
observability riêng (liên hệ mục 34).

---

## PHẦN VI. MCP, A2A VÀ AGENT INTEROPERABILITY

### 25. MCP

MCP chuẩn hóa kết nối:

```
Agent Host
   ↓
MCP Client
   ↓
MCP Server
 ├─ Tools
 ├─ Resources
 └─ Prompts
```

MCP giải quyết agent-to-tool và agent-to-context integration. Specification hiện tại định nghĩa
host, client, server, JSON-RPC message và các capability như tools, resources, prompts cũng như
extensions cho tác vụ dài hạn.

**MCP patterns cần viết chi tiết**

- One server per application
- One server per domain
- MCP gateway
- MCP proxy
- MCP tool registry
- Stateless MCP server
- Long-running MCP task
- MCP authorization broker
- Human approval for MCP mutation
- MCP resource subscription
- MCP-fronted legacy API
- MCP server wrapping an existing agent

### 26. A2A

A2A chuẩn hóa agent-to-agent collaboration:

```
Client Agent
    ↓
Agent Card Discovery
    ↓
Send Task
    ↓
Remote Agent Processing
    ↓
Progress / Artifact / Result
```

A2A hỗ trợ capability discovery, message, task, artifact, streaming, cancellation và asynchronous
execution. Remote agent không phải công khai internal memory, prompt hoặc tool implementation.

**A2A patterns cần viết chi tiết**

- Direct agent invocation
- Agent registry discovery
- Brokered A2A
- A2A task delegation
- Long-running task
- Streaming progress
- Artifact exchange
- Human-in-the-loop task
- Agent federation
- Cross-organization agent collaboration
- A2A gateway
- A2A plus MCP composition

**Quan hệ chuẩn**

```
A2A: Agent → Agent
MCP: Agent → Tool / Data
```

Microsoft khuyến nghị dùng native orchestration cho internal subagent flow, MCP cho tool và data
access, và A2A cho cross-platform agent interaction có published capability và task contract.

### 27. Agent Skills (SKILL.md) — *bổ sung, chuẩn mới 2025*

Bên cạnh MCP (agent→tool) và A2A (agent→agent), ngành đã hình thành thêm một chuẩn thứ ba giải
quyết một vấn đề khác: **đóng gói và chia sẻ năng lực** (capability) mà một agent có thể "học"
để dùng lại, không cần chạy như một service riêng (khác MCP server) và không cần là một agent độc
lập (khác A2A).

```
Agent Host
   ↓
Đọc SKILL.md (YAML frontmatter: name, description... + nội dung hướng dẫn dạng Markdown)
   ↓
Agent "biết" cách thực hiện một năng lực cụ thể mà không cần tool call ra ngoài
```

- Do Anthropic công bố như một open standard (12/2025), quản lý bởi Agentic AI Foundation.
- Một file `SKILL.md` chỉ bắt buộc 2 trường: `name` và `description`; phần còn lại là hướng dẫn
  dạng tự nhiên cho agent.
- Được nhiều nền tảng khác nhau hỗ trợ (Claude, các agent coding tool phổ biến khác...), nghĩa là
  một skill viết một lần có thể tái sử dụng qua nhiều agent host khác nhau — tương tự vai trò
  "thư viện kỹ năng dùng chung" hơn là "kết nối dịch vụ" (MCP) hay "gọi agent khác" (A2A).

**So sánh 3 chuẩn interoperability**

| Chuẩn | Giải quyết | Đơn vị trao đổi |
|---|---|---|
| MCP | Agent → Tool/Data | Tool call, resource, prompt |
| A2A | Agent → Agent | Task, artifact, message |
| Agent Skills | Agent → Capability tái sử dụng | 1 file hướng dẫn (SKILL.md) |

### 28. Agent Payments Protocol và giao thức thanh toán agentic — *bổ sung*

Khi agent cần thực hiện giao dịch tài chính thay mặt user (mua hàng, đặt dịch vụ, chuyển tiền),
cần một tầng chuẩn hoá nằm **giữa** tầng agent (MCP/A2A) và mạng thanh toán, để đảm bảo mọi bên có
thể xác minh độc lập "ai cho phép cái gì".

```
Tầng Agent (MCP / A2A / Agent Skills)
        ↓
Tầng Mandate — AP2 (Intent / Cart / Payment, ký dạng Verifiable Credential)
        ↓
Mạng thanh toán (thẻ, chuyển khoản, real-time payment, stablecoin...)
```

- **AP2 (Agent Payments Protocol)**: Google công bố 9/2025, không ràng buộc vào một phương thức
  thanh toán cụ thể (card, ACH, real-time payment, stablecoin).
- Các chuẩn liên quan trong cùng hệ sinh thái: **ACP** (checkout thương mại điện tử), **x402** và
  **MPP** (thanh toán máy-với-máy). Trong thực tế các chuẩn này thường được dùng bổ sung cho nhau
  chứ không loại trừ lẫn nhau.
- Về mặt kiến trúc, pattern quan trọng nhất cần áp dụng — bất kể có dùng đúng AP2 hay không — là
  **Mandate-based transaction** đã mô tả ở mục 11.10: tách rõ *uỷ quyền có thể kiểm chứng* khỏi
  *hành động thực thi*, và luôn đặt approval boundary (mục 26.8) trước giao dịch giá trị cao.

---

## PHẦN VII. SECURITY, RELIABILITY VÀ GOVERNANCE

### 29. Security patterns

#### 29.1. Identity propagation

User identity phải được truyền qua:

```
User
  ↓
Gateway
  ↓
Supervisor
  ↓
Remote Agent
  ↓
MCP Tool
  ↓
Data Source
```

Không được để agent dùng service account có quyền quá rộng thay cho user.

#### 29.2. Least-privilege agent

Mỗi agent chỉ có access scope cần thiết.

#### 29.3. Capability-based authorization

Quyền thực hiện gắn với capability cụ thể, không chỉ gắn với endpoint.

#### 29.4. Input guardrail

Kiểm tra:

- Prompt injection
- PII
- Malware
- Tool instruction injection
- Invalid schema
- Unsupported file type

#### 29.5. Retrieval guardrail

Kiểm tra:

- ACL
- Tenant
- Data residency
- Poisoned document
- Hidden instruction
- Source trust
- Version validity

#### 29.6. Tool guardrail

Kiểm tra:

- Tool allowlist
- Parameter constraint
- Write permission
- Rate limit
- Transaction amount
- Target environment

#### 29.7. Output guardrail

Kiểm tra:

- Data leakage
- Unsupported claims
- Citation
- Policy violation
- Secret
- PII

#### 29.8. Approval boundary

Các action không thể đảo ngược cần approval.

#### 29.9. Agent admission control

Agent mới phải qua:

- Capability validation
- Schema validation
- Security scanning
- Evaluation suite
- Registration approval

Dynamic Agent Registry không chỉ lưu endpoint. Registry cần validation, security requirement và
evaluation trước khi agent được phép tham gia hệ thống.

#### 29.10. Memory và context poisoning — *bổ sung*

Khác với retrieval guardrail (mục 29.5, vốn xử lý tài liệu bị "đầu độc" nạp vào khi truy xuất),
memory/context poisoning là rủi ro nội dung độc hại bị **ghi vào chính memory dài hạn của agent**
(episodic, semantic, entity memory — mục 22.4-22.7) qua một tương tác trước đó, rồi ảnh hưởng đến
quyết định của agent ở các phiên sau — kể cả khi nguồn gây hại ban đầu đã không còn trong context
hiện tại. Cần:

- Validate nội dung trước khi ghi vào memory dài hạn (không tự động lưu nguyên văn output chưa
  qua kiểm tra)
- Gắn nguồn gốc (provenance) cho mỗi memory record để biết nó đến từ đâu, có đáng tin không
- Có cơ chế memory forgetting (mục 22.10) áp dụng được cho cả nội dung bị nghi ngờ là độc hại,
  không chỉ theo thời gian/độ liên quan

#### 29.11. Excessive agency — ba nguyên nhân gốc — *bổ sung*

Khung an ninh cho ứng dụng agentic (OWASP, 2026) tách "quyền tự chủ quá mức" thành ba nguyên nhân
gốc riêng biệt, hữu ích để audit hệ thống một cách có hệ thống thay vì chỉ nói chung chung "agent
có quá nhiều quyền":

- **Excessive functionality**: agent có thể gọi tới các tool/hành động vượt ngoài phạm vi task
  được giao (liên hệ mục 29.3 Capability-based authorization).
- **Excessive permissions**: tool mà agent gọi có quyền rộng hơn mức cần thiết cho chính tool đó
  (liên hệ mục 29.2 Least-privilege agent).
- **Excessive autonomy**: hành động có tác động lớn được thực thi mà không qua điểm dừng cần con
  người phê duyệt (liên hệ mục 29.8 Approval boundary, và mục 11.10 Mandate-based transaction).

### 30. Reliability patterns cho agent

> Chuyển từ Single-Agent Architecture sang đây để khớp đúng tiêu đề "Security, **Reliability** và
> Governance" trong đề cương gốc.

#### 30.1. Checkpoint and resume

Lưu state sau bước quan trọng (liên hệ mục 23.4).

#### 30.2. Retry with backoff

Retry với giới hạn và exponential backoff.

#### 30.3. Circuit breaker

Tạm ngừng tool hoặc agent đang lỗi liên tục.

#### 30.4. Timeout budget

Mỗi task, step và tool có deadline riêng.

#### 30.5. Dead-letter task

Task thất bại được đưa vào hàng chờ để inspect hoặc xử lý lại.

#### 30.6. Fallback model

Nếu model chính timeout hoặc không đạt quality threshold, chuyển sang model khác.

#### 30.7. Graceful degradation

Nếu một nguồn dữ liệu hỏng, hệ thống trả kết quả từ nguồn còn lại và nêu giới hạn.

#### 30.8. Deterministic state machine

Dùng code kiểm soát transition quan trọng, chỉ dùng LLM cho quyết định ngữ nghĩa.

#### 30.9. Termination guard

Giới hạn:

- Số vòng lặp
- Token
- Chi phí
- Thời gian
- Số tool call
- Số lần handoff

#### 30.10. Saga for long-running agent tasks

Một task lớn được chia thành transaction nhỏ, mỗi transaction có compensating action.

#### 30.11. Context rot mitigation — *bổ sung, liên hệ mục 24.5*

Đặt context rot (mục 24.5) như một failure mode chính thức cần termination guard/observability
theo dõi: agent chạy càng lâu, càng cần chủ động compact/note-taking (mục 24.1-24.2) thay vì để
context tự phình đến giới hạn cứng rồi mới xử lý.

---

## PHẦN VIII. EVALUATION, OBSERVABILITY VÀ PRODUCTION OPERATIONS

### 31. RAG evaluation

**Retrieval**

- Recall@K
- Precision@K
- MRR
- NDCG
- Hit Rate
- Filter accuracy

**Generation**

- Faithfulness
- Answer relevance
- Completeness
- Citation correctness
- Citation completeness
- Unsupported claim rate

**End-to-end**

- Task success
- User satisfaction
- Latency
- Cost
- Abstention quality
- Security leakage

### 32. Agent evaluation

Không chỉ đánh giá final answer. Phải đánh giá cả trajectory:

- Agent có chọn đúng tool không
- Tool parameter có đúng không
- Có gọi thừa tool không
- Có dừng đúng lúc không
- Plan có hợp lệ không
- Có phục hồi sau lỗi không
- Có tuân thủ policy không

**Agent-as-a-Judge — *bổ sung*.** Khác với LLM-as-judge truyền thống (chỉ chấm output cuối), cách
tiếp cận Agent-as-a-Judge dùng một hệ thống agentic để đánh giá — có khả năng multi-step reasoning
và quan sát cả quá trình, không chỉ kết quả — nhờ đó đánh giá được cả *cách* agent đi tới câu trả
lời, không chỉ *câu trả lời cuối*. Một số bộ benchmark public gần đây theo hướng đánh giá
trajectory này gồm AgentRewardBench (chấm khả năng đánh giá plan/thực thi), TRACE (đánh giá dựa
trên evidence bank), và TRAJECT-Bench (đánh giá chi tiết việc dùng tool). Khi tự xây eval nội bộ,
nên tách rõ 2 lớp: (1) rubric chấm từng bước (tool đúng/sai, tham số đúng/sai), và (2) judge chấm
toàn trajectory để bắt các lỗi chỉ lộ ra khi nhìn tổng thể (ví dụ: mỗi bước đều hợp lý nhưng chiến
lược tổng thể sai).

### 33. Multi-agent evaluation

Đánh giá thêm:

- Routing accuracy
- Handoff accuracy
- Delegation success
- Coordination overhead
- Duplicate work
- Conflict resolution
- Agent contribution
- Termination rate
- End-to-end trace completeness

**Đánh giá 3 tầng — *bổ sung*.** Một cách tiếp cận hữu ích cho multi-agent là tách eval thành 3
tầng độc lập thay vì chỉ chấm 1 điểm số cuối: (1) đánh giá độc lập từng agent con (agent đó có
làm đúng phần việc của mình không, tách biệt khỏi phần còn lại của hệ thống), (2) đánh giá chất
lượng phối hợp (handoff, delegation, tổng hợp kết quả giữa các agent có đúng không), và (3) đánh
giá kết quả toàn hệ thống (task cuối cùng có thành công không). Cách tách này giúp định vị lỗi
nằm ở agent con, ở khâu phối hợp, hay ở outcome cuối — thay vì chỉ biết "hệ thống sai" mà không
biết sai ở đâu.

### 34. Observability

Mỗi request cần trace:

```
Trace
 ├─ Authentication Span
 ├─ Router Span
 ├─ Retrieval Span
 │   ├─ BM25 Span
 │   ├─ Vector Span
 │   └─ Reranker Span
 ├─ Agent Span
 │   ├─ Planning Span
 │   ├─ Tool Span
 │   ├─ Context Span (độ dài context, số lần compaction — liên hệ mục 24)
 │   └─ Evaluation Span
 └─ Generation Span
```

Evaluation trả lời "hệ thống có tốt không", còn observability trả lời "hệ thống đang làm gì, lỗi
ở đâu và tại sao". Production RAG cần tách span retrieval khỏi generation, lưu prompt/model/index
version và liên kết quality score ngược về từng trace.

### 35. Production Operations — *bổ sung*

Phần vận hành production trước đây chưa có nội dung riêng dù đã được đặt tên trong đề cương gốc.

#### 35.1. Caching nhiều tầng

Production stack hiện đại thường xếp chồng 3 loại cache, mỗi loại giải quyết một vấn đề khác
nhau:

```
Request
  ↓
1. Exact-match cache — cùng input y hệt → trả thẳng response đã lưu
  ↓ (miss)
2. Semantic cache — input khác chữ nhưng cùng ý nghĩa (so bằng embedding) → trả response gần nhất
  ↓ (miss)
3. Prompt/prefix cache — phần đầu prompt (system prompt, context tĩnh) không đổi → tái dùng phần
   tính toán đã cache ở phía provider, chỉ tính phần mới
  ↓ (miss hoàn toàn)
Gọi model đầy đủ
```

- **Prompt/prefix caching**: tái sử dụng phần tính toán (KV-cache) cho đoạn đầu prompt không đổi
  giữa các lần gọi — hiệu quả nhất khi system prompt/tool schema/context nền tảng dài và lặp lại.
- **Semantic caching**: dùng embedding để phát hiện câu hỏi *khác chữ nhưng cùng ý*, bỏ qua hẳn
  lệnh gọi model nếu đã có câu trả lời tương tự đủ tin cậy — cần threshold similarity đủ chặt để
  tránh trả lời sai ngữ cảnh.
- Hai loại cache trên **giải quyết vấn đề khác nhau** (một giảm chi phí tính toán, một giảm hẳn
  số lệnh gọi) nên thường được dùng **cùng lúc**, không thay thế nhau.

#### 35.2. Cost governance

- Model routing (câu đơn giản → model rẻ, câu phức tạp → model mạnh — liên hệ roadmap LLMOps cơ
  bản)
- Theo dõi cost theo từng trace (mục 34), không chỉ theo tổng hoá đơn cuối tháng
- Đặt budget/alert ở cấp tenant, agent, hoặc task type để phát hiện sớm chi phí bất thường

#### 35.3. Deployment và versioning

- Version hoá đồng thời: prompt, model, index (embedding model đổi → phải re-index), và schema
  tool — bốn thứ này thường lệch pha nhau nếu không quản lý tập trung
- Rollout dần (canary/shadow traffic) khi đổi model hoặc đổi prompt production, đo lại toàn bộ
  evaluation suite (Phần VIII) trước khi rollout 100%
- Rollback plan rõ ràng khi một bản deploy mới làm giảm chất lượng theo eval hoặc observability

---

## PHẦN IX. PHỤ LỤC — KHUNG PHÂN LOẠI VÀ TIÊU CHUẨN VIẾT PATTERN

Một pattern catalog hoàn chỉnh nên phân loại theo layer như sau:

| Layer | Nhóm pattern |
|---|---|
| Data plane | Ingestion, parsing, chunking, enrichment, versioning |
| Knowledge plane | Lexical, vector, graph, structured knowledge |
| Retrieval plane | Hybrid, multi-query, multi-hop, reranking, late-interaction, compression |
| Reasoning plane | ReAct, planning, reflection, evaluator |
| Action plane | Tool use, sandbox, transaction, mandate, compensation |
| Memory/Context plane | Working, semantic, episodic, procedural, artifact, state, context engineering |
| Coordination plane | Supervisor, handoff, swarm, pipeline, blackboard, orchestrator-worker |
| Integration plane | MCP, A2A, Agent Skills, payment mandate, registry, gateway |
| Reliability plane | Checkpoint, retry, timeout, circuit breaker, saga, context-rot mitigation |
| Security plane | Identity, ACL, guardrails, approval, audit, memory poisoning defense |
| Evaluation plane | Retrieval, answer, trajectory, multi-agent evaluation |
| Operations plane | Tracing, metrics, cost, caching, deployment, versioning |

### Tiêu chí viết chi tiết cho từng pattern

Mỗi pattern trong tài liệu chính thức nên có cùng một cấu trúc:

1. Tên pattern
2. Vấn đề cần giải quyết
3. Bối cảnh sử dụng
4. Kiến trúc
5. Thành phần
6. Luồng xử lý chi tiết
7. State và dữ liệu
8. Thuật toán liên quan
9. Cách triển khai
10. Tham số cần tuning
11. Failure modes
12. Security considerations
13. Observability
14. Evaluation metrics
15. Ưu điểm
16. Nhược điểm
17. Khi nên dùng
18. Khi không nên dùng
19. Pattern liên quan
20. Ví dụ kiến trúc thực tế

Ví dụ, phần Hybrid RAG không dừng ở định nghĩa BM25 cộng vector search. Nó phải giải thích
analyzer, BM25 scoring, embedding, HNSW/IVF/PQ, metadata pre-filter và post-filter, parallel
retrieval, RRF, weighted fusion, cross-encoder reranking, contextual compression, token budget,
citation, evaluation và tuning. Tương tự, phần Supervisor phải bao phủ routing policy, agent
registry, shared state, context isolation, timeout, retry, delegation contract, loop prevention,
MCP/A2A integration, tracing và failure recovery.

### Cải tiến chính (đã áp dụng ở phiên bản này)

1. Tách **Memory, State và Context Engineering** thành Phần V riêng thay vì gộp vào Single-Agent
   Architecture, đúng như đề cương 8-phần gốc.
2. Chuyển **Reliability patterns** sang Phần VII để khớp tiêu đề "Security, Reliability và
   Governance".
3. Bổ sung **Production Operations** (Phần VIII, mục 35) — nội dung được đặt tên trong đề cương
   gốc nhưng trước đó chưa có nội dung thực tế.
4. Bổ sung các pattern/khái niệm mới ghi nhận trong ngành 2025-2026: Context Engineering (compaction,
   structured note-taking, sub-agent isolation, context rot), Orchestrator-Worker case study,
   Agent Skills (SKILL.md), Agent Payments Protocol / mandate-based transaction, late-interaction
   retrieval, Agent-as-a-Judge và đánh giá 3 tầng cho multi-agent, memory/context poisoning, và
   phân loại 3 nguyên nhân gốc của excessive agency.
5. Đồng bộ lại khung phân loại plane ở Phụ lục cho khớp với thuật ngữ dùng xuyên suốt tài liệu
   (không còn mâu thuẫn "5 plane" ở phần giới thiệu và "12 plane" ở phụ lục).

Bản mở rộng này chuyển nội dung từ một danh sách pattern thành một reference architecture có hệ
thống. Các phần bổ sung phủ đầy đủ data pipeline, retrieval algorithms, memory/state/context,
tool execution, interoperability, fault tolerance, security, evaluation và production operations,
đồng thời giữ nguyên ba trục chính RAG, AI Agent và Multi-Agent đã xây dựng ban đầu.
