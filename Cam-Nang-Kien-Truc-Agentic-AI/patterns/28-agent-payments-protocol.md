# 28. Agent Payments Protocol và giao thức thanh toán agentic

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Agent Payments Protocol (AP2) và các chuẩn thanh toán agentic liên quan (ACP, x402, MPP) — tầng
chuẩn hoá "mandate" nằm giữa tầng agent (MCP/A2A/Agent Skills) và mạng thanh toán.

## 2. Vấn đề cần giải quyết

Khi agent thực hiện giao dịch tài chính thay mặt user (mua hàng, đặt dịch vụ, chuyển tiền), phát
sinh một vấn đề tin cậy đặc thù không có trong tool call thông thường: mạng thanh toán, merchant, và
chính user cần **xác minh độc lập** rằng agent thực sự được uỷ quyền thực hiện đúng giao dịch đó,
với đúng điều kiện đó — không thể chỉ dựa vào lời agent tự khai "user đã đồng ý". Không có một tầng
chuẩn hoá, mỗi tích hợp thanh toán agentic phải tự nghĩ ra cách chứng minh uỷ quyền, dẫn tới rủi ro
không nhất quán và khó kiểm chứng qua nhiều bên tham gia (agent, merchant, ngân hàng/mạng thanh
toán).

## 3. Bối cảnh sử dụng

Agent có khả năng thực hiện giao dịch tài chính thay mặt user, đặc biệt khi giao dịch có thể xảy ra
không đồng bộ với thời điểm user đưa ra chỉ thị ban đầu (ví dụ agent được giao "mua vé khi giá xuống
dưới X", thực hiện giao dịch sau đó mà không có user trực tiếp giám sát tại thời điểm thực thi).

## 4. Kiến trúc

```
Tầng Agent (MCP / A2A / Agent Skills)
        ↓
Tầng Mandate — AP2 (Intent / Cart / Payment, ký dạng Verifiable Credential)
        ↓
Mạng thanh toán (thẻ, chuyển khoản, real-time payment, stablecoin...)
```

## 5. Thành phần

Intent Mandate (uỷ quyền ở mức ý định — "user đồng ý cho agent mua vé máy bay trong khoảng giá X-Y");
Cart Mandate (uỷ quyền ở mức giỏ hàng cụ thể — "user đồng ý với đúng danh sách sản phẩm/giá này");
Payment Mandate (uỷ quyền thực hiện thanh toán cụ thể); cơ chế ký dạng Verifiable Credential (bằng
chứng mật mã học có thể xác minh độc lập bởi bên thứ ba); approval boundary trước giao dịch giá trị
cao (liên hệ mục 29.8).

## 6. Luồng xử lý chi tiết

Ba loại mandate trong AP2 tương ứng với ba mức độ cụ thể hoá uỷ quyền, thực hiện theo thứ tự tăng
dần mức chi tiết: **Intent Mandate** ghi nhận ý định chung của user (điều kiện, giới hạn giá, phạm vi
cho phép) tại thời điểm user giao việc cho agent — đây là bằng chứng "user đã đồng ý về nguyên tắc"
trước khi biết chi tiết giao dịch cụ thể sẽ là gì. Khi agent tìm được lựa chọn cụ thể khớp với Intent
Mandate, hệ thống tạo **Cart Mandate** ghi nhận chính xác nội dung giao dịch cụ thể (sản phẩm, giá,
điều khoản) — mandate này là bằng chứng "đây chính xác là những gì sẽ được mua", tách biệt khỏi ý
định chung ban đầu để merchant/mạng thanh toán có thể xác minh đúng nội dung giao dịch thực tế. Cuối
cùng **Payment Mandate** uỷ quyền hành động thanh toán thực sự, thường là bước cuối trước khi tiền
thực sự di chuyển. Mỗi mandate được ký dạng Verifiable Credential — cho phép bất kỳ bên nào trong
chuỗi (merchant, ngân hàng, hệ thống audit) xác minh độc lập tính hợp lệ của uỷ quyền mà không cần
tin tưởng mù quáng vào lời khai của agent.

Các chuẩn liên quan trong cùng hệ sinh thái phục vụ mục đích bổ sung: **ACP** tập trung vào checkout
thương mại điện tử; **x402** và **MPP** giải quyết thanh toán máy-với-máy (machine-to-machine),
thường ở quy mô giao dịch nhỏ, tần suất cao hơn giao dịch tiêu dùng thông thường. Trong triển khai
thực tế các chuẩn này thường dùng bổ sung cho nhau theo từng loại giao dịch, không loại trừ lẫn
nhau.

## 7. State và dữ liệu

Mỗi mandate là một bản ghi bất biến (immutable) một khi đã ký — không sửa đổi ngược, chỉ có thể tạo
mandate mới (ví dụ Cart Mandate mới nếu nội dung giỏ hàng đổi) — tương tự nguyên tắc event-sourced
state (liên hệ mục 23.6): lịch sử đầy đủ các mandate là audit trail của toàn bộ quá trình uỷ quyền.

## 8. Thuật toán liên quan

Cơ chế ký số/Verifiable Credential (mật mã học chữ ký số, không phải thuật toán ML) là nền tảng kỹ
thuật chính; không có thuật toán học máy liên quan trực tiếp tới bản thân giao thức.

## 9. Cách triển khai

1. Áp dụng nguyên tắc kiến trúc quan trọng nhất bất kể có dùng đúng AP2 hay không: **tách rõ uỷ
   quyền có thể kiểm chứng khỏi hành động thực thi** — đây chính là Mandate-based transaction đã mô
   tả ở mục 11.10, AP2 là một cách chuẩn hoá cụ thể của nguyên tắc đó.
2. Luôn đặt approval boundary (mục 29.8) trước giao dịch giá trị cao, dù đã có đầy đủ Intent/Cart/
   Payment Mandate hợp lệ — mandate chứng minh uỷ quyền hợp lệ, không thay thế hoàn toàn nhu cầu
   điểm dừng con người cho giao dịch rủi ro lớn.
3. Thiết kế Intent Mandate với giới hạn rõ ràng, cụ thể (khoảng giá, danh mục sản phẩm cho phép) —
   Intent Mandate mơ hồ làm giảm giá trị của toàn bộ chuỗi mandate phía sau.
4. Với giao dịch máy-với-máy tần suất cao (x402/MPP), cân nhắc riêng về giới hạn tốc độ/tổng giá trị
   trong một khoảng thời gian, không chỉ giới hạn từng giao dịch đơn lẻ.

## 10. Tham số cần tuning

Ngưỡng giá trị giao dịch yêu cầu approval boundary bổ sung; thời hạn hiệu lực (TTL) của Intent
Mandate trước khi cần user xác nhận lại; giới hạn tổng giá trị/tần suất giao dịch trong một khoảng
thời gian cho use case machine-to-machine.

## 11. Failure modes

- **Thiếu tách bạch giữa các mức mandate**: gộp uỷ quyền ý định chung và uỷ quyền giao dịch cụ thể
  làm một, mất khả năng xác minh độc lập từng bước, tăng rủi ro giao dịch vượt quá những gì user
  thực sự đồng ý.
- **Bỏ qua approval boundary vì "đã có mandate hợp lệ"**: coi mandate hợp lệ là đủ để tự động hoá
  hoàn toàn giao dịch giá trị cao, bỏ qua điểm dừng con người cần thiết.
- **Intent Mandate quá rộng**: giới hạn ban đầu (khoảng giá, phạm vi) quá lỏng lẻo khiến agent có
  không gian quyết định rộng hơn mức user thực sự mong muốn.

## 12. Security considerations

Đây thuộc nhóm rủi ro Excessive autonomy (mục 29.11) nếu không kiểm soát chặt — mandate cung cấp
bằng chứng uỷ quyền có thể kiểm chứng, nhưng hệ thống vẫn cần approval boundary (mục 29.8) cho hành
động không thể đảo ngược (tiền đã chuyển khó/không thể hoàn tác dễ dàng). Cần bảo vệ khoá ký mandate
khỏi bị lộ (nếu khoá bị chiếm đoạt, kẻ tấn công có thể tạo mandate giả mạo hợp lệ về mặt mật mã học).

## 13. Observability

Log đầy đủ chuỗi Intent → Cart → Payment Mandate cho mỗi giao dịch, có thể truy vết ngược từ giao
dịch thực tế về đúng uỷ quyền ban đầu của user; theo dõi tỷ lệ giao dịch bị chặn ở approval boundary
và lý do.

## 14. Evaluation metrics

Tỷ lệ giao dịch có đầy đủ chuỗi mandate hợp lệ trước khi thực thi (nên gần 100%); thời gian từ Intent
Mandate tới Payment Mandate hoàn tất; tỷ lệ giao dịch cần con người can thiệp bổ sung tại approval
boundary.

## 15. Ưu điểm

Cho phép nhiều bên (merchant, mạng thanh toán, hệ thống audit) xác minh độc lập tính hợp lệ của uỷ
quyền mà không cần tin tưởng mù quáng agent, mở khoá khả năng agent thực hiện giao dịch tài chính an
toàn hơn ở quy mô lớn.

## 16. Nhược điểm

Thêm độ phức tạp đáng kể (ba lớp mandate, cơ chế ký số) so với tích hợp thanh toán truyền thống; hệ
sinh thái chuẩn (AP2, ACP, x402, MPP) còn đang phát triển, có thể thay đổi trước khi ổn định hoàn
toàn.

## 17. Khi nên dùng

Bất kỳ agent nào thực hiện giao dịch tài chính thay mặt user, đặc biệt khi giao dịch diễn ra không
đồng bộ (không có user giám sát trực tiếp tại thời điểm thực thi) — nguyên tắc mandate-based nên áp
dụng dù có dùng đúng chuẩn AP2 hay tự xây cơ chế tương đương.

## 18. Khi không nên dùng

Giao dịch luôn có user xác nhận trực tiếp ngay tại thời điểm thực hiện (không có khoảng cách thời
gian giữa uỷ quyền và thực thi) — cơ chế xác nhận đơn giản tại chỗ có thể đủ, không cần đầu tư đầy đủ
hạ tầng ba lớp mandate.

## 19. Pattern liên quan

Là hiện thực hoá chuẩn hoá của Mandate-based transaction (mục 11.10); phụ thuộc trực tiếp vào
Approval boundary (mục 29.8) cho giao dịch giá trị cao; là một dạng đặc biệt của Excessive autonomy
(mục 29.11) cần kiểm soát; nằm ở tầng trên các chuẩn interoperability MCP/A2A/Agent Skills (file
25-27).

## 20. Ví dụ kiến trúc thực tế

Agent trợ lý mua sắm cá nhân: user cấp Intent Mandate cho phép agent mua vé máy bay tuyến cụ thể nếu
giá dưới ngưỡng X trong vòng 7 ngày tới; agent theo dõi giá liên tục, khi tìm được vé phù hợp tạo
Cart Mandate ghi nhận chính xác chuyến bay/giá/điều khoản; vì giá trị giao dịch vượt ngưỡng cấu hình
approval boundary, hệ thống gửi thông báo xác nhận cuối tới user trước khi tạo Payment Mandate và
hoàn tất giao dịch — toàn bộ chuỗi ba mandate được lưu làm audit trail, có thể tra cứu lại nếu user
thắc mắc về giao dịch sau này.

---

*Nguồn tham khảo dùng khi biên soạn: công bố công khai của Google về Agent Payments Protocol (AP2,
9/2025) và các chuẩn liên quan trong hệ sinh thái thanh toán agentic (ACP, x402, MPP). Nội dung là
tổng hợp và diễn giải lại, không trích dẫn nguyên văn đặc tả.*
