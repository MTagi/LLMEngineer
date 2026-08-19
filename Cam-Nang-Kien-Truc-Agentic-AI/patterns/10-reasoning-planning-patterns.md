# 10. Reasoning và planning patterns

[← Về mục lục chính](../README.md)

7 pattern thuộc **reasoning plane** của một agent đơn lẻ: ReAct, Plan-and-Execute, Replanning,
Tree-based exploration, Iterative deepening, Constraint-based planning, Goal decomposition.

## 1. Tên pattern

ReAct · Plan-and-Execute · Replanning · Tree-based exploration · Iterative deepening ·
Constraint-based planning · Goal decomposition.

## 2. Vấn đề cần giải quyết

Một agent cần một chiến lược để đi từ "goal" đến "kết quả" khi số bước và thứ tự bước không thể
biết trước (khác Workflow — file 01). Có nhiều cách tổ chức reasoning khác nhau, mỗi cách phù hợp
với một dạng bài toán: bài toán tuyến tính đơn giản, bài toán cần lập kế hoạch trước, bài toán có
nhiều hướng giải khả thi cần so sánh, bài toán có ràng buộc cứng phải tuân thủ, bài toán lớn cần
chia nhỏ.

## 3. Bối cảnh sử dụng

| Pattern | Phù hợp nhất khi |
|---|---|
| ReAct | Task tương đối ngắn, mỗi bước phụ thuộc trực tiếp kết quả bước trước |
| Plan-and-Execute | Task có thể phác thảo toàn bộ các bước trước khi bắt đầu thực thi |
| Replanning | Plan ban đầu có khả năng cao bị vô hiệu bởi kết quả trung gian bất ngờ |
| Tree-based exploration | Có nhiều hướng giải khả thi, cần so sánh trước khi cam kết một hướng |
| Iterative deepening | Chi phí mỗi lần thử cao, muốn thử giải pháp rẻ trước khi leo thang |
| Constraint-based planning | Có ràng buộc cứng (deadline, budget, quyền) phải tuân thủ khi lập kế hoạch |
| Goal decomposition | Goal lớn, không thể hành động trực tiếp mà cần chia thành các cấp nhỏ hơn |

## 4. Kiến trúc

```
ReAct:                Observe → Decide → Act → Observe → Continue   (lặp, không lập kế hoạch
                      trước toàn bộ)

Plan-and-Execute:      Planner: [Step 1, Step 2, Step 3, ...]  (lập TRƯỚC)
                       Executor: thực hiện Step 1 → Step 2 → Step 3   (lần lượt)

Replanning:            [Plan ban đầu] → Execute Step → Kiểm tra plan còn hợp lệ? →
                       (Có: tiếp tục | Không: [Plan mới])

Tree-based:            Problem
                        ├─ Approach A → A1, A2
                        ├─ Approach B
                        └─ Approach C
                        → Evaluator chọn nhánh tiềm năng

Goal decomposition:     Goal
                         ├─ Milestone
                         │   ├─ Task
                         │   │   ├─ Action
                         │   │   └─ Verification
```

## 5. Thành phần

Model đóng vai trò reasoning engine; với Plan-and-Execute cần tách rõ hai vai trò Planner/Executor
(có thể cùng một model, khác system prompt, hoặc hai model khác nhau); Tree-based cần thêm một
Evaluator (chấm điểm/so sánh các nhánh); Constraint-based planning cần một validator kiểm tra plan
so với ràng buộc trước khi cho phép thực thi.

## 6. Luồng xử lý chi tiết

- **ReAct** (Reasoning + Acting): vòng lặp đơn giản nhất — model quan sát trạng thái hiện tại,
  quyết định hành động tiếp theo, thực thi, quan sát kết quả, lặp lại đến khi đạt goal hoặc chạm
  termination guard (mục 30.9). Không có bước lập kế hoạch tường minh trước — mỗi quyết định chỉ
  dựa vào trạng thái hiện tại.
- **Plan-and-Execute**: tách biệt lập kế hoạch khỏi thực thi — Planner sinh toàn bộ danh sách bước
  trước khi bất kỳ hành động nào được thực hiện, Executor thực hiện lần lượt. Ưu điểm: có thể xem
  xét/audit plan trước khi chạy (kể cả cho human duyệt); nhược điểm: plan có thể sai lệch nếu thực
  tế khác giả định lúc lập kế hoạch.
- **Replanning**: bổ sung cho Plan-and-Execute — sau mỗi bước thực thi, kiểm tra xem plan còn phù
  hợp với thực tế mới quan sát được không; nếu không, gọi lại Planner để tạo plan mới thay vì cố
  chấp thực hiện plan cũ đã lỗi thời.
- **Tree-based exploration**: khi có nhiều hướng giải khả thi mà không rõ hướng nào tốt nhất, agent
  sinh nhiều nhánh tiếp cận song song (hoặc tuần tự), một evaluator (có thể là chính model, một
  model khác, hoặc rule-based) đánh giá và chọn nhánh tiềm năng nhất để đi tiếp — có thể quay lại
  thử nhánh khác nếu nhánh đã chọn thất bại (backtracking).
- **Iterative deepening**: bắt đầu bằng giải pháp/độ sâu reasoning đơn giản nhất, rẻ nhất; chỉ
  tăng độ sâu (thử nhiều bước hơn, dùng model mạnh hơn, khám phá nhiều nhánh hơn) khi giải pháp
  đơn giản chưa đạt yêu cầu — tránh lãng phí chi phí cho các task thực ra đơn giản.
- **Constraint-based planning**: plan được sinh ra phải thoả một tập ràng buộc tường minh (deadline,
  ngân sách, quyền dùng tool, thứ tự phụ thuộc bắt buộc giữa các bước, yêu cầu approval, ràng buộc
  vị trí lưu trữ dữ liệu) — validator kiểm tra plan trước khi cho phép Executor chạy, từ chối hoặc
  yêu cầu lập lại plan nếu vi phạm.
- **Goal decomposition**: chia goal lớn thành cấu trúc phân cấp milestone → task → action →
  verification — mỗi action có bước xác minh riêng trước khi coi là hoàn thành, giúp theo dõi tiến
  độ và phát hiện lỗi sớm ở cấp thấp thay vì chỉ biết "task lớn thất bại" mà không rõ ở đâu.

## 7. State và dữ liệu

Plan (danh sách bước, trạng thái mỗi bước) là state chính cần lưu — liên hệ Working memory (mục
22.1) và State store (file 23); với Tree-based exploration cần lưu cả các nhánh đã thử và kết quả
đánh giá để hỗ trợ backtracking mà không lặp lại công việc đã làm.

## 8. Thuật toán liên quan

Không có thuật toán chuẩn hoá — reasoning được điều khiển bởi khả năng suy luận của model qua
prompt engineering (few-shot ReAct examples, cấu trúc output cho plan). Tree-based exploration có
thể mượn ý tưởng từ tìm kiếm cây cổ điển (best-first search) nhưng "hàm đánh giá" ở đây thường là
một LLM call thay vì hàm heuristic tường minh.

## 9. Cách triển khai

1. Mặc định dùng **ReAct** cho hầu hết agent đơn giản — chi phí thấp nhất, đủ tốt cho task không
   quá phức tạp hoặc không cần audit trước khi hành động.
2. Chuyển sang **Plan-and-Execute** khi cần review/approval plan trước khi thực thi (đặc biệt với
   hành động có rủi ro — liên hệ Dry-run và approval, mục 11.9), hoặc khi task đủ dài mà lập kế
   hoạch trước giúp tránh đi vào ngõ cụt.
3. Luôn cân nhắc thêm **Replanning** khi dùng Plan-and-Execute cho môi trường không hoàn toàn dự
   đoán được — plan tĩnh không tự sửa dễ dẫn tới thực thi mù quáng khi thực tế lệch giả định.
4. Chỉ dùng **Tree-based exploration** khi có bằng chứng bài toán thực sự có nhiều hướng giải khả
   thi khác biệt đáng kể — đây là pattern tốn chi phí (nhiều lệnh gọi model song song).
5. Áp **Constraint-based planning** bắt buộc cho bất kỳ agent nào có quyền thực hiện hành động
   giá trị cao hoặc không thể đảo ngược.

## 10. Tham số cần tuning

Số bước tối đa trong plan trước khi bắt buộc replanning; số nhánh song song trong tree-based
exploration; ngưỡng "đủ tốt" để dừng iterative deepening thay vì leo thang tiếp; độ sâu phân cấp
tối đa trong goal decomposition.

## 11. Failure modes

- **ReAct lặp không hội tụ**: agent cứ quan sát-quyết định mà không tiến triển thực chất, cần
  Termination guard (mục 30.9).
- **Plan cứng nhắc**: Plan-and-Execute không có Replanning, cố chấp thực hiện plan đã lỗi thời so
  với thực tế mới quan sát.
- **Tree-based bùng nổ tổ hợp**: số nhánh tăng theo cấp số nhân nếu không giới hạn độ rộng/độ sâu
  khám phá, chi phí vượt tầm kiểm soát.
- **Goal decomposition quá chi tiết**: chia goal thành quá nhiều cấp/action nhỏ khiến overhead
  điều phối vượt quá lợi ích, làm chậm task đơn giản không cần cấu trúc phức tạp vậy.

## 12. Security considerations

Constraint-based planning là điểm chặn quan trọng để đảm bảo plan không vi phạm quyền hạn/ngân
sách trước khi thực thi — cần validate **trước khi Executor chạy bất kỳ bước nào**, không chỉ
validate từng hành động riêng lẻ lúc thực thi (lúc đó có thể đã quá muộn nếu bước đầu đã gây hậu
quả).

## 13. Observability

Log plan đầy đủ (không chỉ hành động cuối cùng) cho Plan-and-Execute/Goal decomposition — khi debug
một kết quả sai, cần biết plan ban đầu là gì, có replanning không, và tại bước nào lệch khỏi kỳ
vọng.

## 14. Evaluation metrics

Liên hệ Agent evaluation (file 32): tỷ lệ plan hợp lệ (thoả constraint) ngay từ lần đầu; số lần
replanning trung bình mỗi task; với tree-based, chất lượng lựa chọn nhánh của evaluator so với
nhánh tối ưu (nếu biết được qua đánh giá thủ công).

## 15. Ưu điểm

Cho phép agent xử lý bài toán có độ mở khác nhau bằng chiến lược reasoning phù hợp, thay vì ép mọi
bài toán qua một khuôn ReAct đơn giản hoặc một Plan-and-Execute cứng nhắc.

## 16. Nhược điểm

Các pattern phức tạp hơn (Tree-based, Constraint-based, Goal decomposition) đều tăng số lệnh gọi
model và độ phức tạp code điều phối — cần cân nhắc kỹ trước khi vượt khỏi ReAct/Plan-and-Execute
cơ bản.

## 17. Khi nên dùng

ReAct và Plan-and-Execute nên là hai lựa chọn mặc định đầu tiên. Các pattern còn lại áp dụng khi
có đặc điểm bài toán cụ thể đòi hỏi (nhiều hướng giải, ràng buộc cứng, cấu trúc phân cấp rõ).

## 18. Khi không nên dùng

Nếu control flow thực ra có thể liệt kê hết trước — đây không còn là bài toán Agent nữa mà là
Workflow (file 01), không nên ép vào bất kỳ pattern reasoning nào ở đây.

## 19. Pattern liên quan

Kết hợp trực tiếp với Tool-use patterns (file 11) — mỗi "Act" trong ReAct hoặc mỗi "Action" trong
Goal decomposition thường là một lệnh gọi tool. Liên hệ Termination guard (mục 30.9) và Context
Engineering (file 24) để kiểm soát chi phí/độ dài context khi reasoning kéo dài nhiều bước.

## 20. Ví dụ kiến trúc thực tế

Agent xử lý yêu cầu hỗ trợ kỹ thuật phức tạp: dùng Goal decomposition để chia "giải quyết sự cố"
thành milestone (chẩn đoán → đề xuất giải pháp → thực hiện → xác minh); trong milestone chẩn đoán
dùng ReAct để lặp tìm nguyên nhân; trước khi thực hiện thay đổi hệ thống thật, chuyển sang
Plan-and-Execute với Constraint-based planning (yêu cầu approval, giới hạn phạm vi thay đổi) và
Replanning nếu bước xác minh phát hiện chẩn đoán ban đầu sai.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về pattern ReAct (Reasoning + Acting) và các
chiến lược lập kế hoạch phổ biến trong kiến trúc AI agent. Nội dung là tổng hợp và diễn giải lại.*
