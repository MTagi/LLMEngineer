# 01. Phân biệt LLM Application, Workflow, Agent và Multi-Agent

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Không phải một pattern đơn lẻ mà là **khung phân loại nền tảng** (foundational taxonomy) — bốn
hình thái kiến trúc khi đưa LLM vào một hệ thống phần mềm, xếp theo mức độ tự chủ tăng dần: LLM
Application → LLM Workflow → AI Agent → Multi-Agent System.

## 2. Vấn đề cần giải quyết

Đội ngũ kỹ thuật thường gọi mọi thứ có LLM bên trong là "AI Agent", dẫn đến ba hệ quả xấu:

- Chọn sai độ phức tạp kiến trúc — dựng agent loop tốn kém cho việc một workflow tĩnh giải quyết
  tốt hơn, rẻ hơn, dễ audit hơn.
- Đánh giá và giám sát sai cách — một workflow cần test theo nhánh rẽ cố định; một agent cần đánh
  giá theo trajectory (mục 32); nếu nhầm loại, bộ test không bắt được đúng loại lỗi.
- Giao tiếp sai kỳ vọng với stakeholder — "agent tự động hoá toàn bộ quy trình" nghe khác hẳn "một
  workflow có gọi LLM ở bước phân loại".

## 3. Bối cảnh sử dụng

Dùng khung này ở đầu mọi dự án tích hợp LLM, trước khi chọn framework hay viết dòng code đầu
tiên — như một quyết định kiến trúc, không phải sau khi đã lỡ xây agent loop rồi mới nhận ra bài
toán chỉ cần workflow.

## 4. Kiến trúc

```
Mức độ tự chủ tăng dần →

LLM Application        LLM Workflow            AI Agent                 Multi-Agent System
────────────────       ─────────────            ─────────                ──────────────────
Input → LLM → Output    Input → [B1→B2→B3] →     Goal → (Observe→Plan→     Orchestrator
(1 lệnh gọi cố định)    Output (control flow      Act→Observe)* →          ├─ Agent A (scope riêng)
                        do code định nghĩa,       Finish/Escalate          ├─ Agent B (scope riêng)
                        LLM chỉ quyết định        (control flow do         └─ Agent C (scope riêng)
                        NỘI DUNG từng bước)       model quyết định động)   (mỗi agent tự vòng lặp
                                                                            Observe→Plan→Act riêng)
```

## 5. Thành phần

| Hình thái | Thành phần bắt buộc |
|---|---|
| LLM Application | Prompt template, LLM client |
| LLM Workflow | Trên + workflow engine/code điều phối bước, business rule giữa các bước |
| AI Agent | Trên + tool, memory, state, policy, termination condition, retry, checkpoint |
| Multi-Agent | Trên + ranh giới rõ giữa các agent (prompt/tool/knowledge/memory/quyền/lifecycle/owner riêng), cơ chế điều phối (Phần IV) |

## 6. Luồng xử lý chi tiết

- **LLM Application**: `User Input → Prompt Template → LLM → Response`. Không có bước quan sát
  kết quả trung gian để quyết định bước tiếp theo — vì chỉ có một bước.
- **LLM Workflow**: `Input → Classify → Extract → Validate → Generate → Output`. Mỗi mũi tên là một
  cạnh **cố định trong code**; LLM có thể tự quyết định nội dung trả về ở mỗi node (ví dụ nhãn
  phân loại), nhưng không tự quyết được node tiếp theo là gì — node tiếp theo do workflow engine
  định tuyến, kể cả khi có rẽ nhánh if/else dựa trên output LLM.
- **AI Agent**: `Goal → Observe → Plan/Select Action → Execute Tool → Observe Result → Update
  State → Continue/Finish/Escalate`. Vòng lặp này **do chính model quyết định khi nào dừng và
  bước tiếp theo là gì** — đây là ranh giới định nghĩa workflow vs agent.
- **Multi-Agent**: nhiều vòng lặp Agent chạy trong các ranh giới tách biệt, phối hợp qua một trong
  các pattern ở Phần IV (12-21).

## 7. State và dữ liệu

- LLM Application: không có state giữa các lần gọi.
- Workflow: state là biến trung gian giữa các bước, thường do workflow engine quản lý (ví dụ:
  kết quả bước Classify được truyền làm input cho bước Extract).
- Agent: state gồm goal, lịch sử hành động, kết quả tool, working memory (chi tiết ở Phần V).
- Multi-Agent: state được **chia theo ranh giới agent** — mỗi agent giữ state riêng, chỉ trao đổi
  phần cần thiết qua message/artifact (không chia sẻ toàn bộ state).

## 8. Thuật toán liên quan

Không có thuật toán riêng ở tầng phân loại này — quyết định thuộc về **control flow**: cố định
(code) hay động (model). Anthropic dùng chính tiêu chí này để phân biệt workflow với agent.

## 9. Cách triển khai

Quy trình quyết định thực dụng:

1. Liệt kê các bước cần thiết để đi từ input đến output.
2. Hỏi: thứ tự và điều kiện rẽ nhánh giữa các bước có thể viết hết bằng code trước khi chạy
   không? Nếu có → Workflow (hoặc LLM Application nếu chỉ 1 bước).
3. Nếu không — vì số bước, thứ tự bước, hoặc điều kiện dừng phụ thuộc vào nội dung kết quả trung
   gian theo cách không liệt kê hết trước được → Agent.
4. Hỏi tiếp: agent có cần nhiều ranh giới trách nhiệm/quyền/context tách biệt (ví dụ: một phần
   việc cần quyền truy cập dữ liệu tài chính, phần khác cần quyền truy cập code) không? Nếu có →
   Multi-Agent; nếu một agent với nhiều tool là đủ → giữ Single-Agent (đơn giản hơn, ít lỗi phối
   hợp hơn).

## 10. Tham số cần tuning

Không áp dụng theo nghĩa tham số runtime — "tham số" ở đây là **quyết định kiến trúc** (ngưỡng độ
phức tạp/độ mở của bài toán để leo từ workflow lên agent, từ agent đơn lên multi-agent).

## 11. Failure modes

- **Over-engineering**: dùng agent loop cho bài toán vốn là workflow tĩnh → chi phí token cao hơn,
  hành vi khó đoán hơn, khó audit hơn, không có lợi ích tương xứng.
- **Under-engineering**: ép workflow cứng xử lý bài toán cần suy luận động → nhánh if/else phình
  to không kiểm soát nổi, không xử lý được case chưa lường trước.
- **Multi-agent giả**: tạo nhiều "agent" chỉ là các prompt nhỏ dùng chung tool, chung quyền, chung
  state — về bản chất vẫn là một workflow bị chia nhỏ thành nhiều node, không có lợi ích thật của
  multi-agent (context isolation, chuyên môn hoá, phát triển độc lập) nhưng lại gánh chi phí điều
  phối của multi-agent.

## 12. Security considerations

Mức độ tự chủ càng cao, bề mặt rủi ro càng lớn: LLM Application chỉ rủi ro ở nội dung output;
Workflow thêm rủi ro logic rẽ nhánh sai; Agent thêm rủi ro hành động thật qua tool (cần guardrail
— Phần VII); Multi-Agent thêm rủi ro về ranh giới quyền giữa các agent (một agent bị chiếm quyền
không được lan quyền sang agent khác — identity propagation, mục 29.1).

## 13. Observability

Cần log rõ **hệ thống đang ở hình thái nào cho mỗi request** — nhầm lẫn phổ biến là gắn trace của
workflow (chỉ cần log từng bước cố định) vào một hệ thống thực ra đang chạy agent loop (cần log
cả reasoning/quyết định dừng), khiến thiếu dữ liệu để debug khi agent hành xử bất ngờ.

## 14. Evaluation metrics

- LLM Application/Workflow: test theo input/output cố định cho từng bước, giống unit test truyền
  thống.
- Agent: cần đánh giá trajectory, không chỉ output cuối (xem mục 32).
- Multi-agent: cần thêm đánh giá phối hợp giữa các agent (mục 33).

## 15. Ưu điểm

Có một khung quyết định rõ ràng giúp tránh tranh cãi mơ hồ "đây có phải agent không", tập trung
vào câu hỏi có ý nghĩa kỹ thuật: control flow cố định hay động, và ranh giới trách nhiệm có thật
hay không.

## 16. Nhược điểm

Ranh giới giữa workflow "có nhiều rẽ nhánh phức tạp" và agent "đơn giản" đôi khi mờ trong thực tế
— cần đánh giá theo tinh thần (ai quyết định bước tiếp theo) hơn là theo hình thức (có bao nhiêu
dòng code if/else).

## 17. Khi nên dùng

Dùng khung này bắt buộc ở giai đoạn thiết kế của **mọi** dự án tích hợp LLM, không riêng dự án
lớn.

## 18. Khi không nên dùng

Không có "khi không nên dùng" — đây là bài tập tư duy trước khi chọn kiến trúc, không phải một
kỹ thuật triển khai cụ thể có chi phí để cân nhắc bỏ qua.

## 19. Pattern liên quan

- Toàn bộ Phần III (Single-Agent) và Phần IV (Multi-Agent) là hệ quả trực tiếp của việc chọn
  "Agent" hoặc "Multi-Agent" ở bước phân loại này.
- Reliability patterns (Phần VII, mục 30) chủ yếu áp dụng cho Agent/Multi-Agent, ít liên quan đến
  LLM Application/Workflow thuần.

## 20. Ví dụ kiến trúc thực tế

- **LLM Application**: chatbot trả lời 1 câu hỏi FAQ dựa trên 1 lệnh gọi model, không có bước
  trung gian.
- **LLM Workflow**: pipeline xử lý email hỗ trợ khách hàng — Classify loại yêu cầu → Extract
  thông tin cần thiết → Validate theo rule nghiệp vụ → Generate câu trả lời — mỗi bước là 1 lệnh
  gọi LLM nhưng thứ tự bước cố định trong code.
- **AI Agent**: trợ lý nghiên cứu tự quyết định cần tìm kiếm thêm hay đã đủ thông tin để trả lời,
  lặp lại đến khi tự tin về câu trả lời.
- **Multi-Agent System**: hệ thống hỗ trợ kỹ thuật có Documentation Agent, Database Agent, Coding
  Agent, Planning Agent và Review Agent — mỗi agent có tool, quyền truy cập và owner riêng, được
  một Orchestrator điều phối theo yêu cầu người dùng (mục 1.4 trong tài liệu tổng quan).

---

*Nguồn tham khảo dùng khi biên soạn: hướng dẫn phân biệt workflow/agent của Anthropic; khuyến
nghị lựa chọn agent của Google. Nội dung là tổng hợp và diễn giải lại, không trích dẫn nguyên
văn.*
