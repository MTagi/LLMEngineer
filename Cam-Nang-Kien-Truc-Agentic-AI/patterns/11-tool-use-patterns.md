# 11. Tool-use patterns

[← Về mục lục chính](../README.md)

10 pattern thuộc **action plane** — cách agent gọi và thực thi tool một cách an toàn, đáng tin
cậy: Direct invocation, Tool chaining, Tool router, Tool broker, Tool sandbox, Compensating
transaction, Idempotent tool, Read-before-write, Dry-run và approval, Mandate-based transaction.

## 1. Tên pattern

Direct tool invocation · Tool chaining · Tool router · Tool broker · Tool sandbox · Compensating
transaction · Idempotent tool pattern · Read-before-write · Dry-run và approval · Mandate-based
transaction.

## 2. Vấn đề cần giải quyết

Gọi tool là điểm nơi agent chuyển từ "sinh văn bản" sang "thực hiện hành động thật" — đây là nơi
rủi ro thực tế phát sinh (dữ liệu bị ghi sai, giao dịch bị lặp, hành động không thể hoàn tác). 10
pattern này giải quyết các khía cạnh khác nhau của việc làm cho tool call **an toàn, đáng tin
cậy, và có thể kiểm soát** — không chỉ "gọi được tool" mà "gọi tool đúng cách trong production".

## 3. Bối cảnh sử dụng

| Pattern | Dùng khi |
|---|---|
| Direct invocation | Tool đơn giản, không có bước trung gian |
| Tool chaining | Output tool này là input tool khác |
| Tool router | Agent có nhiều tool, cần giới hạn tập tool khả dụng theo ngữ cảnh |
| Tool broker | Cần quản lý tập trung discovery/permission/timeout/quota/logging |
| Tool sandbox | Tool chạy code/file/browser action có rủi ro |
| Compensating transaction | Chuỗi hành động nhiều bước có thể thất bại giữa chừng |
| Idempotent tool | Tool có retry, cần đảm bảo retry không tạo dữ liệu trùng |
| Read-before-write | Ghi dữ liệu có điều kiện phụ thuộc trạng thái hiện tại |
| Dry-run và approval | Hành động có tác động lớn/khó đảo ngược |
| Mandate-based transaction | Giao dịch có giá trị tài chính thật |

## 4. Kiến trúc

```
Compensating transaction:   Create Reservation → Charge Payment ✗(fail) → Cancel Reservation
                            (mỗi bước thành công có 1 hành động bù tương ứng nếu bước sau fail)

Dry-run và approval:        Proposed Action → Dry Run → Impact Report → Human Approval → Execute

Mandate-based transaction:  User Intent → Intent Mandate → Agent chọn phương án → Cart Mandate
                            (cần user xác nhận) → Payment Mandate (ký số) → Payment Network
                            xác minh → Thực thi
```

## 5. Thành phần

Tool registry (danh sách tool khả dụng + schema); router/broker (lớp trung gian quản lý quyền,
quota, logging); sandbox runtime (môi trường cô lập cho code/file/browser); idempotency key store
(theo dõi các request đã xử lý để tránh lặp); mandate store (lưu các uỷ quyền đã ký cho giao dịch).

## 6. Luồng xử lý chi tiết

- **Direct tool invocation**: agent chọn 1 tool, gọi 1 lần, nhận kết quả — dạng đơn giản nhất, đã
  mô tả chi tiết ở Function Calling & Tools cơ bản.
- **Tool chaining**: output của tool A được dùng trực tiếp làm input cho tool B mà không cần quay
  lại model giữa chừng (khác với ReAct — nơi model quan sát lại sau mỗi tool call) — hữu ích khi
  chuỗi bước cố định, giảm số lệnh gọi model.
- **Tool router**: một lớp đứng trước model, chỉ expose tập con tool phù hợp với ngữ cảnh hiện tại
  thay vì toàn bộ danh sách tool hệ thống có — giảm nhiễu lựa chọn cho model và giảm bề mặt rủi ro
  (agent không thể gọi nhầm tool ngoài phạm vi hiện tại dù có "biết" tool đó tồn tại).
- **Tool broker**: lớp trung gian tập trung quản lý discovery (tool nào tồn tại), permission (ai
  được gọi tool nào), timeout, quota (giới hạn số lần gọi), và logging — tách biệt hạ tầng quản lý
  tool khỏi logic nghiệp vụ của từng tool riêng lẻ.
- **Tool sandbox**: hành động thực thi code, thao tác file, hay browser automation chạy trong môi
  trường cô lập (container, VM tạm thời, quyền hệ thống hạn chế) — giới hạn thiệt hại nếu code
  thực thi có lỗi hoặc bị lợi dụng.
- **Compensating transaction**: khi một chuỗi hành động nhiều bước thất bại ở bước giữa chừng, mỗi
  bước đã thành công trước đó có một **hành động bù** tương ứng để đưa hệ thống về trạng thái nhất
  quán (ví dụ: đã tạo reservation, charge payment thất bại → hủy reservation) — cần thiết vì
  không có transaction thật (ACID) xuyên suốt nhiều hệ thống bên ngoài khác nhau.
- **Idempotent tool pattern**: tool thực hiện ghi dữ liệu phải nhận một idempotency key (do caller
  sinh ra, giữ nguyên qua các lần retry) — nếu cùng key được gửi lại (do retry sau timeout), tool
  trả về kết quả của lần gọi gốc thay vì thực hiện lại hành động, tránh tạo dữ liệu trùng.
- **Read-before-write**: trước khi ghi, agent đọc trạng thái hiện tại và kiểm tra precondition (ví
  dụ: số dư đủ, trạng thái đơn hàng hợp lệ) — tránh ghi đè mù quáng lên trạng thái đã thay đổi kể
  từ lần agent quan sát gần nhất.
- **Dry-run và approval**: trước khi thực thi hành động thật, chạy thử ở chế độ mô phỏng (dry run)
  để tạo báo cáo tác động (impact report), rồi yêu cầu con người phê duyệt dựa trên báo cáo đó
  trước khi thực thi thật — bắt buộc với hành động không thể đảo ngược hoặc tác động lớn.
- **Mandate-based transaction**: xem chi tiết ở mục 11.10 trong tài liệu tổng quan và file 28 —
  tách uỷ quyền có thể kiểm chứng (mandate) khỏi hành động thực thi, cho phép bên thứ ba xác minh
  độc lập.

## 7. State và dữ liệu

Idempotency key store (thường TTL giới hạn — không cần giữ vĩnh viễn, chỉ đủ lâu để bao phủ cửa sổ
retry hợp lý); impact report (dry-run) cần lưu tạm để đối chiếu khi approval được cấp, tránh
trường hợp trạng thái thay đổi giữa lúc dry-run và lúc approval khiến impact report không còn
chính xác.

## 8. Thuật toán liên quan

Không có thuật toán ML — đây là các pattern kỹ thuật phần mềm (distributed systems: idempotency,
compensating transaction/saga — liên hệ mục 30.10) áp dụng vào ngữ cảnh agent gọi tool.

## 9. Cách triển khai

1. Mọi tool có tác dụng phụ (ghi dữ liệu, gọi API bên ngoài) nên có **idempotency key** ngay từ
   đầu thiết kế — chi phí thêm vào thấp, lợi ích phòng ngừa lỗi trùng lặp rất cao.
2. Áp **Tool router** khi số lượng tool trong hệ thống vượt quá vài chục — tránh để model phải
   chọn giữa danh sách tool quá dài, dễ chọn nhầm.
3. Bọc mọi tool chạy code/file/browser trong **sandbox** — không có ngoại lệ, kể cả khi tin tưởng
   nguồn gốc lệnh gọi.
4. Với chuỗi hành động nhiều bước chạm tới hệ thống bên ngoài, thiết kế **compensating action**
   cho từng bước ngay khi thiết kế tool, không thêm sau khi đã gặp sự cố thực tế.
5. Với hành động không thể đảo ngược hoặc giá trị cao, bắt buộc **Dry-run và approval** — không
   để agent tự quyết định thực thi trực tiếp.

## 10. Tham số cần tuning

TTL của idempotency key; timeout mỗi tool call; quota (số lần gọi tool tối đa mỗi phiên/mỗi user);
ngưỡng giá trị giao dịch bắt buộc approval (liên hệ mục 26.8/29.8).

## 11. Failure modes

- **Retry tạo dữ liệu trùng**: tool ghi dữ liệu không có idempotency key, agent retry sau timeout
  tạo ra 2 bản ghi cho cùng 1 hành động người dùng dự định.
- **Compensating action không đối xứng hoàn toàn**: hành động bù không thực sự đưa hệ thống về
  đúng trạng thái ban đầu (ví dụ hủy reservation nhưng phí giữ chỗ đã bị trừ không hoàn lại được)
  — cần thiết kế và test compensating action kỹ như hành động chính.
- **Sandbox rò rỉ quyền**: cấu hình sandbox quá lỏng, code/action thực thi bên trong vẫn truy cập
  được tài nguyên ngoài phạm vi dự kiến.
- **Dry-run lạc hậu so với approval thực tế**: trạng thái hệ thống thay đổi giữa lúc dry-run và
  lúc con người phê duyệt, hành động thật thực thi trên giả định đã lỗi thời.

## 12. Security considerations

Tool broker là điểm tập trung để áp permission/quota — nên là **điểm bắt buộc đi qua** (không có
đường tắt gọi tool trực tiếp bỏ qua broker); mandate-based transaction và dry-run/approval là hai
lớp phòng vệ bổ sung cho hành động giá trị cao, nên dùng cùng lúc chứ không thay thế lẫn nhau.

## 13. Observability

Log đầy đủ mỗi tool call: tham số, idempotency key (nếu có), kết quả, thời gian thực thi, và với
compensating transaction — log rõ hành động bù nào đã được kích hoạt và kết quả của nó (hành động
bù thất bại là một loại sự cố nghiêm trọng cần alert riêng).

## 14. Evaluation metrics

Tỷ lệ tool call thành công/thất bại; tỷ lệ retry cần idempotency xử lý; tỷ lệ compensating action
được kích hoạt và tỷ lệ thành công của chính compensating action đó; thời gian trung bình từ
dry-run đến approval (nếu quá lâu, cần cảnh báo về độ lệch trạng thái).

## 15. Ưu điểm

Các pattern này chuyển tool-use từ "gọi hàm demo" sang "thao tác đáng tin cậy trong hệ thống thật"
— giải quyết đúng lớp vấn đề mà phần lớn sự cố agent-in-production gặp phải (không phải model chọn
sai tool, mà là tool được gọi không an toàn).

## 16. Nhược điểm

Thêm đáng kể độ phức tạp thiết kế cho mỗi tool (đặc biệt idempotency, compensating action) — với
tool chỉ đọc dữ liệu (không side effect), phần lớn các pattern này không cần thiết.

## 17. Khi nên dùng

Idempotent tool pattern và Tool sandbox nên là mặc định cho mọi tool có side effect. Các pattern
còn lại áp dụng theo mức độ rủi ro/giá trị của hành động cụ thể.

## 18. Khi không nên dùng

Với tool chỉ đọc (read-only, không side effect, ví dụ tra cứu thông tin), phần lớn pattern trong
nhóm này (compensating transaction, mandate-based, dry-run/approval) không áp dụng — chỉ Direct
invocation/Tool router là đủ.

## 19. Pattern liên quan

Kết hợp trực tiếp với Reasoning/Planning (file 10) — mỗi "Act" trong ReAct là một lệnh gọi theo
các pattern ở đây. Liên hệ Saga (mục 30.10) là dạng mở rộng của Compensating transaction cho task
dài hạn nhiều giai đoạn; liên hệ Approval boundary (mục 29.8) và Agent Payments Protocol (file 28)
cho Mandate-based transaction.

## 20. Ví dụ kiến trúc thực tế

Agent đặt lịch công tác: Tool broker quản lý quyền gọi API đặt vé máy bay/khách sạn; mỗi bước ghi
(đặt vé, đặt phòng, thanh toán) có idempotency key riêng; toàn chuỗi có compensating action (hủy
vé nếu đặt phòng thất bại, hủy phòng nếu thanh toán thất bại); bước thanh toán cuối cùng bắt buộc
qua Mandate-based transaction với Dry-run hiển thị tổng chi phí cho user phê duyệt trước khi agent
thực sự trừ tiền.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về idempotency, saga pattern, và sandboxing
trong hệ thống phân tán, áp dụng vào ngữ cảnh AI agent tool-use. Nội dung là tổng hợp và diễn giải
lại.*
