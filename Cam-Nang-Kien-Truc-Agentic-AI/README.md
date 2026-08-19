# Cẩm nang kiến trúc Agentic AI — RAG, Agent, Multi-Agent

> Tài liệu này coi nội dung ban đầu (3 nhóm pattern: RAG, AI Agent, Multi-Agent) là **khung ban
> đầu, không phải danh sách đóng**. Bản chuyên sâu mở rộng từ việc "mô tả pattern" thành một
> **cẩm nang kiến trúc và triển khai production**, bao phủ dữ liệu, retrieval, reasoning, memory,
> orchestration, protocol, security, fault tolerance, evaluation và vận hành.

## Định hướng mở rộng tài liệu

Thay vì chỉ có ba nhóm ban đầu, nội dung hoàn chỉnh được tổ chức thành 8 phần lớn:

1. Nền tảng kiến trúc Generative AI
2. RAG và Knowledge Architecture
3. Single-Agent Architecture
4. Multi-Agent Architecture
5. Memory, State và Context Engineering
6. MCP, A2A và Agent Interoperability
7. Security, Reliability và Governance
8. Evaluation, Observability và Production Operations

Các pattern không hoàn toàn tách biệt. Một hệ thống thực tế có thể đồng thời sử dụng Hybrid RAG,
Parent-Child Retrieval, Planner-Executor, Supervisor, Shared State, MCP và A2A. Vì vậy, tài liệu
phải chỉ rõ pattern nào thuộc **data plane, control plane, reasoning plane, integration plane và
governance plane**, thay vì liệt kê chúng như các lựa chọn ngang hàng.

---

## Mục lục

- [PHẦN I. NỀN TẢNG KIẾN TRÚC](#phần-i-nền-tảng-kiến-trúc)
- [PHẦN II. RAG VÀ KNOWLEDGE ARCHITECTURE](#phần-ii-rag-và-knowledge-architecture)
- [PHẦN III. AI AGENT PATTERNS MỞ RỘNG](#phần-iii-ai-agent-patterns-mở-rộng)
- [PHẦN IV. MULTI-AGENT PATTERNS MỞ RỘNG](#phần-iv-multi-agent-patterns-mở-rộng)
- [PHẦN V. MCP, A2A VÀ INTEROPERABILITY](#phần-v-mcp-a2a-và-interoperability)
- [PHẦN VI. SECURITY VÀ GOVERNANCE](#phần-vi-security-và-governance)
- [PHẦN VII. EVALUATION VÀ OBSERVABILITY](#phần-vii-evaluation-và-observability)
- [PHẦN VIII. KHUNG PHÂN LOẠI CUỐI CÙNG](#phần-viii-khung-phân-loại-cuối-cùng)

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

## PHẦN III. AI AGENT PATTERNS MỞ RỘNG

Ngoài 10 pattern ban đầu, phần AI Agent nên bổ sung các nhóm sau.

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

### 12. Memory patterns

#### 12.1. Working memory

State của task hiện tại:

- Plan
- Tool results
- Current step
- Intermediate artifact
- Errors

#### 12.2. Conversation memory

Message giữa user và agent trong session.

#### 12.3. Summary memory

Tóm tắt lịch sử để giảm token.

#### 12.4. Semantic memory

Lưu fact và knowledge tổng quát.

#### 12.5. Episodic memory

Lưu event hoặc task đã xảy ra:

- Task
- Action
- Outcome
- Feedback
- Lesson

#### 12.6. Procedural memory

Lưu cách làm, rule, workflow hoặc reusable skill.

#### 12.7. Entity memory

Lưu thông tin theo user, customer, product, project hoặc organization.

#### 12.8. Artifact memory

Lưu file, report, code, query result và document được tạo trong quá trình làm việc.

#### 12.9. Memory consolidation

Nhiều memory ngắn được tổng hợp thành memory ổn định hơn.

#### 12.10. Memory forgetting

Xóa hoặc giảm trọng số memory theo:

- Thời gian
- Độ liên quan
- Privacy policy
- User request
- Data retention
- Contradiction với thông tin mới

Memory không đồng nghĩa với vector database. Một kiến trúc tốt thường kết hợp state store,
relational database, object storage, vector index và event log.

### 13. Reliability patterns cho agent

#### 13.1. Checkpoint and resume

Lưu state sau bước quan trọng.

#### 13.2. Retry with backoff

Retry với giới hạn và exponential backoff.

#### 13.3. Circuit breaker

Tạm ngừng tool hoặc agent đang lỗi liên tục.

#### 13.4. Timeout budget

Mỗi task, step và tool có deadline riêng.

#### 13.5. Dead-letter task

Task thất bại được đưa vào hàng chờ để inspect hoặc xử lý lại.

#### 13.6. Fallback model

Nếu model chính timeout hoặc không đạt quality threshold, chuyển sang model khác.

#### 13.7. Graceful degradation

Nếu một nguồn dữ liệu hỏng, hệ thống trả kết quả từ nguồn còn lại và nêu giới hạn.

#### 13.8. Deterministic state machine

Dùng code kiểm soát transition quan trọng, chỉ dùng LLM cho quyết định ngữ nghĩa.

#### 13.9. Termination guard

Giới hạn:

- Số vòng lặp
- Token
- Chi phí
- Thời gian
- Số tool call
- Số lần handoff

#### 13.10. Saga for long-running agent tasks

Một task lớn được chia thành transaction nhỏ, mỗi transaction có compensating action.

---

## PHẦN IV. MULTI-AGENT PATTERNS MỞ RỘNG

Ngoài 12 pattern ban đầu, phần này nên có thêm các pattern sau.

### 14. Federated multi-agent

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

### 15. Market-based task allocation

Worker đưa ra bid dựa trên:

- Capability
- Cost
- Current load
- Estimated completion time
- Confidence

Coordinator chọn worker có utility tốt nhất.

### 16. Contract-net pattern

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

### 17. Map-reduce agents

```
Large Input
   ↓
Map Agents
   ↓
Partial Results
   ↓
Reduce Agent
   ↓
Final Result
```

### 18. Committee-of-experts

Router chọn một nhóm expert thay vì chỉ một agent. Judge hoặc synthesizer tổng hợp output.

### 19. Red-team and blue-team

- Blue agent tạo giải pháp.
- Red agent tìm lỗi, rủi ro hoặc attack path.
- Judge xác định vấn đề nào hợp lệ.
- Blue agent sửa kết quả.

### 20. Shared artifact workspace

Agent cộng tác qua file và artifact thay vì chỉ message:

```
Shared Workspace
 ├─ requirements.md
 ├─ architecture.md
 ├─ implementation/
 ├─ test-results.json
 └─ review-comments.json
```

### 21. Event-driven multi-agent

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

### 22. Choreography

Không có orchestrator trung tâm. Mỗi agent phản ứng với event và phát event tiếp theo.

### 23. Hybrid orchestration

Kết hợp:

- Serial cho bước phụ thuộc
- Parallel cho kiểm tra độc lập
- Supervisor cho lựa chọn domain
- Handoff cho hội thoại
- Event-driven cho tác vụ nền

Microsoft lưu ý rằng use case phức tạp thường cần kết hợp serial, concurrent và orchestrated
agent types, thay vì cố dùng một pattern duy nhất cho toàn bộ hệ thống.

---

## PHẦN V. MCP, A2A VÀ INTEROPERABILITY

### 24. MCP

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

### 25. A2A

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

---

## PHẦN VI. SECURITY VÀ GOVERNANCE

### 26. Security patterns cần bổ sung

#### 26.1. Identity propagation

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

#### 26.2. Least-privilege agent

Mỗi agent chỉ có access scope cần thiết.

#### 26.3. Capability-based authorization

Quyền thực hiện gắn với capability cụ thể, không chỉ gắn với endpoint.

#### 26.4. Input guardrail

Kiểm tra:

- Prompt injection
- PII
- Malware
- Tool instruction injection
- Invalid schema
- Unsupported file type

#### 26.5. Retrieval guardrail

Kiểm tra:

- ACL
- Tenant
- Data residency
- Poisoned document
- Hidden instruction
- Source trust
- Version validity

#### 26.6. Tool guardrail

Kiểm tra:

- Tool allowlist
- Parameter constraint
- Write permission
- Rate limit
- Transaction amount
- Target environment

#### 26.7. Output guardrail

Kiểm tra:

- Data leakage
- Unsupported claims
- Citation
- Policy violation
- Secret
- PII

#### 26.8. Approval boundary

Các action không thể đảo ngược cần approval.

#### 26.9. Agent admission control

Agent mới phải qua:

- Capability validation
- Schema validation
- Security scanning
- Evaluation suite
- Registration approval

Dynamic Agent Registry không chỉ lưu endpoint. Registry cần validation, security requirement và
evaluation trước khi agent được phép tham gia hệ thống.

---

## PHẦN VII. EVALUATION VÀ OBSERVABILITY

### 27. RAG evaluation

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

### 28. Agent evaluation

Không chỉ đánh giá final answer. Phải đánh giá cả trajectory:

- Agent có chọn đúng tool không
- Tool parameter có đúng không
- Có gọi thừa tool không
- Có dừng đúng lúc không
- Plan có hợp lệ không
- Có phục hồi sau lỗi không
- Có tuân thủ policy không

### 29. Multi-agent evaluation

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

### 30. Observability

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
 │   └─ Evaluation Span
 └─ Generation Span
```

Evaluation trả lời "hệ thống có tốt không", còn observability trả lời "hệ thống đang làm gì, lỗi
ở đâu và tại sao". Production RAG cần tách span retrieval khỏi generation, lưu prompt/model/index
version và liên kết quality score ngược về từng trace.

---

## PHẦN VIII. KHUNG PHÂN LOẠI CUỐI CÙNG

Một pattern catalog hoàn chỉnh nên phân loại theo layer như sau:

| Layer | Nhóm pattern |
|---|---|
| Data plane | Ingestion, parsing, chunking, enrichment, versioning |
| Knowledge plane | Lexical, vector, graph, structured knowledge |
| Retrieval plane | Hybrid, multi-query, multi-hop, reranking, compression |
| Reasoning plane | ReAct, planning, reflection, evaluator |
| Action plane | Tool use, sandbox, transaction, compensation |
| Memory plane | Working, semantic, episodic, procedural, artifact |
| Coordination plane | Supervisor, handoff, swarm, pipeline, blackboard |
| Integration plane | MCP, A2A, registry, gateway |
| Reliability plane | Checkpoint, retry, timeout, circuit breaker, saga |
| Security plane | Identity, ACL, guardrails, approval, audit |
| Evaluation plane | Retrieval, answer, trajectory, multi-agent evaluation |
| Operations plane | Tracing, metrics, cost, deployment, versioning |

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

### Cải tiến chính

Bản mở rộng này chuyển nội dung từ một danh sách pattern thành một reference architecture có hệ
thống. Các phần mới bổ sung đầy đủ data pipeline, retrieval algorithms, memory, tool execution,
interoperability, fault tolerance, security, evaluation và production operations, đồng thời giữ
nguyên ba trục chính RAG, AI Agent và Multi-Agent đã xây dựng ban đầu.
