# 25. MCP (Model Context Protocol)

[← Về mục lục chính](../README.md)

## 1. Tên pattern

MCP — chuẩn giao thức mở hoá kết nối giữa agent và tool/dữ liệu bên ngoài, cùng 12 pattern triển
khai: One server per application, One server per domain, MCP gateway, MCP proxy, MCP tool registry,
Stateless MCP server, Long-running MCP task, MCP authorization broker, Human approval for MCP
mutation, MCP resource subscription, MCP-fronted legacy API, MCP server wrapping an existing agent.

## 2. Vấn đề cần giải quyết

Trước khi có một chuẩn chung, mỗi agent tích hợp với mỗi tool/nguồn dữ liệu theo một giao diện
riêng — số lượng tích hợp cần viết tăng theo tích (số agent × số tool), mỗi cặp lại có định dạng
tool schema, cách xử lý lỗi, cách phân trang khác nhau. MCP giải quyết bằng cách chuẩn hoá **một**
giao thức giữa agent host và bất kỳ MCP server nào — viết một MCP server một lần, mọi agent host hỗ
trợ MCP đều dùng được, biến bài toán tích hợp từ tích thành tổng (số agent + số tool).

## 3. Bối cảnh sử dụng

Bất kỳ agent nào cần gọi tool, đọc tài nguyên dữ liệu bên ngoài, hoặc dùng prompt template dùng
chung — đặc biệt khi tổ chức có nhiều agent host khác nhau cùng cần truy cập cùng một tập nguồn dữ
liệu/tool nội bộ.

## 4. Kiến trúc

```
Agent Host
   ↓
MCP Client
   ↓ (JSON-RPC)
MCP Server
 ├─ Tools       (hành động agent có thể gọi)
 ├─ Resources   (dữ liệu agent có thể đọc, có thể subscribe)
 └─ Prompts     (template dùng chung)
```

## 5. Thành phần

MCP Host (ứng dụng chứa agent); MCP Client (thành phần trong host quản lý kết nối tới server); MCP
Server (expose tools/resources/prompts qua JSON-RPC); Authorization broker (nếu tách riêng — mục
25.8); Tool registry (nếu dùng — mục 25.5).

## 6. Luồng xử lý chi tiết

**12 pattern triển khai:**

- **One server per application (25.1)**: mỗi ứng dụng nghiệp vụ (CRM, ticketing, kho dữ liệu) có
  một MCP server riêng, expose đúng tập tool/resource của ứng dụng đó — đơn giản, ranh giới rõ
  ràng, nhưng số server tăng tuyến tính theo số ứng dụng.
- **One server per domain (25.2)**: gom nhiều ứng dụng liên quan trong cùng một domain nghiệp vụ
  (ví dụ "domain Tài chính" gồm cả hệ thống hoá đơn lẫn hệ thống thanh toán) vào một MCP server —
  giảm số server phải quản lý, đánh đổi lấy ranh giới quyền hạn rộng hơn trong cùng một server.
- **MCP gateway (25.3)**: một điểm vào duy nhất đứng trước nhiều MCP server, xử lý routing,
  authentication, rate limiting tập trung — agent host chỉ cần biết một endpoint, gateway lo việc
  định tuyến tới server phù hợp.
- **MCP proxy (25.4)**: trung gian giữa client và server thực hiện biến đổi/lọc message (ví dụ ẩn
  bớt tool nhạy cảm khỏi một số client, log mọi request cho audit) mà không cần sửa server gốc.
- **MCP tool registry (25.5)**: danh mục tập trung liệt kê mọi MCP server và tool khả dụng trong tổ
  chức, cho phép agent host khám phá (discovery) tool phù hợp thay vì phải biết trước địa chỉ từng
  server.
- **Stateless MCP server (25.6)**: server không giữ state giữa các request, mỗi request tự chứa đủ
  thông tin cần thiết — cho phép scale ngang dễ dàng (bất kỳ instance nào cũng xử lý được bất kỳ
  request nào), phù hợp phần lớn use case tool call đơn giản.
- **Long-running MCP task (25.7)**: với tác vụ mất nhiều thời gian (không thể trả kết quả ngay
  trong một request-response), server trả về task handle, client poll hoặc subscribe cập nhật tiến
  độ — cần cơ chế riêng ngoài request-response đồng bộ thông thường.
- **MCP authorization broker (25.8)**: tách riêng logic xác thực/uỷ quyền khỏi từng MCP server,
  tập trung vào một broker — server chỉ cần tin broker đã xác minh identity/quyền, không tự triển
  khai logic authorization riêng lẻ ở mỗi server (tránh không nhất quán).
- **Human approval for MCP mutation (25.9)**: mọi tool call có tính chất thay đổi dữ liệu (mutation,
  khác với read-only) qua MCP cần điểm dừng cho phép con người duyệt trước khi thực thi, đặc biệt
  với hành động không thể đảo ngược (liên hệ Approval boundary, mục 29.8).
- **MCP resource subscription (25.10)**: thay vì client phải poll liên tục để biết resource có thay
  đổi không, client subscribe và server chủ động đẩy thông báo khi resource cập nhật — giảm tải
  polling không cần thiết.
- **MCP-fronted legacy API (25.11)**: bọc một API cũ (không theo chuẩn hiện đại, khó tích hợp trực
  tiếp) bằng một MCP server làm lớp chuyển đổi — agent host chỉ cần nói chuyện với MCP, không cần
  biết chi tiết API cũ bên dưới.
- **MCP server wrapping an existing agent (25.12)**: expose chính một agent đã có sẵn như một MCP
  server (agent đó trở thành "tool" cho agent khác gọi) — cách đơn giản để tái sử dụng một agent
  chuyên biệt trong một hệ thống agent lớn hơn mà không cần xây lại giao thức riêng.

## 7. State và dữ liệu

Phần lớn MCP server nên thiết kế stateless (mục 25.6) khi có thể; khi cần state (long-running task,
mục 25.7), state đó tuân theo nguyên tắc chung của State (file 23) — durable nếu task cần resume
được qua restart.

## 8. Thuật toán liên quan

Không có thuật toán ML — đây là chuẩn giao thức (protocol specification) dựa trên JSON-RPC.

## 9. Cách triển khai

1. Chọn ranh giới server (per-application hay per-domain) dựa trên ranh giới quyền hạn thực tế
   trong tổ chức, không chỉ dựa trên ranh giới kỹ thuật của hệ thống backend.
2. Thiết kế tool schema rõ ràng, mô tả đầy đủ (description chất lượng cao ảnh hưởng trực tiếp tới
   khả năng agent chọn đúng tool — liên hệ Tool selection, file 11).
3. Tách authorization ra broker riêng (mục 25.8) ngay từ đầu nếu dự kiến có nhiều MCP server — sửa
   lại sau khi đã có nhiều server tự triển khai authorization riêng lẻ tốn công hơn nhiều.
4. Với mọi tool có khả năng mutation, mặc định yêu cầu human approval (mục 25.9) trừ khi đã có đánh
   giá rủi ro rõ ràng cho phép tự động hoá.
5. Dùng gateway (mục 25.3) khi số server vượt quá mức một agent host có thể quản lý kết nối trực
   tiếp hợp lý.

## 10. Tham số cần tuning

Timeout cho tool call; giới hạn số resource subscription đồng thời; ngưỡng số lượng tool trong một
server trước khi nên tách (quá nhiều tool trong một server làm agent khó chọn đúng tool — liên hệ
Tool selection, file 11).

## 11. Failure modes

- **Tool schema mô tả kém**: agent chọn sai tool hoặc truyền sai tham số vì description không đủ rõ
  ràng — đây là nguyên nhân lỗi phổ biến nhất trong thực tế triển khai MCP.
- **Mutation không qua approval**: server cho phép gọi trực tiếp hành động thay đổi dữ liệu quan
  trọng mà không có điểm dừng con người — rủi ro an ninh nghiêm trọng (liên hệ mục 29.8, 29.11).
- **Long-running task không có cơ chế theo dõi tiến độ**: client chờ vô thời hạn hoặc timeout sớm vì
  server không hỗ trợ đúng pattern 25.7.
- **Authorization triển khai không nhất quán giữa các server**: mỗi server tự làm theo cách riêng,
  dẫn tới lỗ hổng ở server nào đó triển khai thiếu chặt chẽ.

## 12. Security considerations

MCP server thường là điểm truy cập trực tiếp vào dữ liệu/hành động nhạy cảm — cần identity
propagation xuyên suốt (mục 29.1), least-privilege cho mỗi server (mục 29.2), capability-based
authorization (mục 29.3), và guardrail riêng cho input/tool call (mục 29.4, 29.6). MCP-fronted
legacy API (mục 25.11) đặc biệt cần rà soát kỹ vì API cũ bên dưới có thể chưa được thiết kế với giả
định "sẽ được agent gọi tự động".

## 13. Observability

Trace riêng biệt cho từng MCP tool call (liên hệ Tool Span, mục 34), log đầy đủ tham số gọi và kết
quả trả về (trừ dữ liệu nhạy cảm), theo dõi tỷ lệ lỗi theo từng server/tool để phát hiện sớm server
nào đang gặp vấn đề.

## 14. Evaluation metrics

Tỷ lệ agent chọn đúng tool khi có tool phù hợp khả dụng (đo chất lượng tool schema); độ trễ trung
bình mỗi tool call; tỷ lệ mutation call được approve đúng quy trình (không có bypass).

## 15. Ưu điểm

Chuẩn hoá tích hợp agent-to-tool, giảm số lượng tích hợp cần viết từ tích thành tổng, cho phép tái
sử dụng MCP server qua nhiều agent host khác nhau không cần sửa đổi.

## 16. Nhược điểm

Thêm một lớp giao thức cần học và vận hành; thiết kế sai ranh giới server (quá nhiều tool trong một
server, hoặc quá manh mún) gây khó khăn cho cả agent lẫn đội vận hành.

## 17. Khi nên dùng

Mọi tích hợp agent-to-tool/data trong tổ chức có từ hai agent host trở lên, hoặc dự kiến sẽ có nhiều
agent host trong tương lai — chuẩn hoá sớm bằng MCP tránh phải viết lại tích hợp riêng lẻ sau này.

## 18. Khi không nên dùng

Một agent đơn lẻ, một tool đơn lẻ, không dự kiến mở rộng — tích hợp trực tiếp đơn giản hơn, không
cần thêm lớp giao thức.

## 19. Pattern liên quan

Bổ sung trực tiếp cho Tool use patterns (file 11); liên hệ A2A (file 26) — MCP là agent→tool/data,
A2A là agent→agent; liên hệ Approval boundary (mục 29.8) và Mandate-based transaction (mục 11.10)
cho mutation nhạy cảm.

## 20. Ví dụ kiến trúc thực tế

Tổ chức có 5 agent host khác nhau (trợ lý nội bộ, agent hỗ trợ khách hàng, agent phân tích dữ liệu,
hai agent coding tool) cùng cần truy cập hệ thống CRM và kho tài liệu nội bộ: thay vì viết 10 tích
hợp riêng lẻ (5 host × 2 hệ thống), triển khai 2 MCP server (per-application, mục 25.1) đứng sau một
gateway chung (mục 25.3) với authorization broker tập trung (mục 25.8) — mọi agent host kết nối qua
cùng một giao thức, mutation trên CRM (tạo/sửa bản ghi khách hàng) luôn qua human approval (mục
25.9).

---

*Nguồn tham khảo dùng khi biên soạn: đặc tả công khai của Model Context Protocol (Anthropic) và các
khuyến nghị triển khai phổ biến trong ngành. Nội dung là tổng hợp và diễn giải lại, không trích dẫn
nguyên văn đặc tả.*
