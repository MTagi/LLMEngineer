# 27. Agent Skills (SKILL.md)

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Agent Skills — chuẩn mở đóng gói và chia sẻ năng lực (capability) tái sử dụng cho agent qua một file
`SKILL.md`, khác biệt với MCP (agent→tool, chạy như service) và A2A (agent→agent, một agent độc
lập).

## 2. Vấn đề cần giải quyết

MCP giải quyết agent-to-tool và A2A giải quyết agent-to-agent, nhưng còn một loại nhu cầu khác: một
**hướng dẫn/quy trình** mà agent cần "học" để thực hiện một năng lực cụ thể — không cần chạy như một
service riêng biệt (khác MCP server phải triển khai, host, bảo trì như một hệ thống) và không cần là
một agent độc lập có vòng đời riêng (khác A2A). Trước khi có chuẩn này, mỗi tổ chức/cá nhân tự nhúng
hướng dẫn dạng này trực tiếp vào system prompt — không tái sử dụng được qua nhiều agent host khác
nhau, không có cấu trúc thống nhất để chia sẻ.

## 3. Bối cảnh sử dụng

Cần đóng gói một quy trình/kiến thức chuyên biệt (ví dụ cách tuân theo một style guide cụ thể, cách
thực hiện một loại phân tích lặp lại, cách tương tác với một định dạng file đặc thù) để agent áp
dụng lại nhiều lần, chia sẻ được giữa các thành viên hoặc thậm chí giữa các agent host khác nhau.

## 4. Kiến trúc

```
Agent Host
   ↓
Đọc SKILL.md (YAML frontmatter: name, description... + nội dung hướng dẫn dạng Markdown)
   ↓
Agent "biết" cách thực hiện một năng lực cụ thể mà không cần tool call ra ngoài
```

## 5. Thành phần

File `SKILL.md` (YAML frontmatter bắt buộc `name` và `description`, cộng nội dung hướng dẫn dạng
Markdown tự nhiên); cơ chế discovery của agent host (đọc metadata để biết skill nào khả dụng, load
nội dung đầy đủ khi cần); thư mục/registry skill (nếu tổ chức có nhiều skill dùng chung).

## 6. Luồng xử lý chi tiết

Agent host quét các `SKILL.md` khả dụng, đọc phần frontmatter nhẹ (`name`, `description`) để biết
skill nào tồn tại và làm gì — đây là bước discovery chi phí thấp, không cần tải toàn bộ nội dung
skill vào context. Khi một tác vụ khớp với mô tả của một skill, agent host load nội dung đầy đủ
(hướng dẫn dạng Markdown) vào context tại đúng thời điểm cần (liên hệ trực tiếp Just-in-time context
loading, mục 24.4) thay vì nạp sẵn toàn bộ skill ngay từ đầu phiên làm việc. Vì skill chỉ là một file
hướng dẫn tự nhiên (không phải service phải gọi qua mạng như MCP, không phải một agent độc lập có
vòng đời riêng như A2A), một skill viết một lần dùng ngay được trên bất kỳ agent host nào hỗ trợ
chuẩn này, không cần triển khai hạ tầng riêng.

## 7. State và dữ liệu

Skill bản thân không giữ state — mỗi lần agent dùng một skill là một lần đọc lại nội dung hướng dẫn
tĩnh, hành vi runtime (dữ liệu xử lý, kết quả) thuộc về working memory (mục 22.1) của phiên đang
chạy, không thuộc về skill.

## 8. Thuật toán liên quan

Không có thuật toán ML riêng — cơ chế cốt lõi là việc agent host chọn skill phù hợp dựa trên khớp
mô tả tác vụ với trường `description`, tương tự bài toán retrieval/matching đơn giản hơn là một mô
hình học máy chuyên biệt.

## 9. Cách triển khai

1. Viết trường `description` đủ rõ ràng và cụ thể — đây là căn cứ chính để agent host quyết định có
   nên load skill này cho một tác vụ hay không, mô tả mơ hồ dẫn tới skill không bao giờ được chọn
   đúng lúc hoặc bị chọn nhầm.
2. Giữ nội dung hướng dẫn tập trung vào **một** năng lực cụ thể — skill cố gắng bao quát quá nhiều
   việc khác nhau làm giảm độ chính xác của discovery và khó bảo trì.
3. Tách phần nội dung chi tiết ít dùng ra file phụ được tham chiếu từ `SKILL.md` chính, tận dụng cơ
   chế just-in-time loading thay vì nhồi toàn bộ vào một file duy nhất luôn được nạp đầy đủ.
4. Với tổ chức có nhiều skill dùng chung, xây dựng một thư mục/registry để các đội tìm và tái sử
   dụng skill đã có thay vì viết trùng lặp.

## 10. Tham số cần tuning

Không có tham số runtime theo nghĩa truyền thống — "tuning" chủ yếu là chất lượng nội dung
`description` và mức độ chi tiết/tập trung của nội dung hướng dẫn.

## 11. Failure modes

- **Description mơ hồ**: agent host không chọn đúng skill khi cần, hoặc chọn nhầm skill không phù
  hợp cho tác vụ.
- **Skill ôm quá nhiều năng lực khác nhau**: giảm độ chính xác discovery, và khi skill được load thì
  tốn context không cần thiết cho phần không liên quan tới tác vụ hiện tại.
- **Nội dung skill lỗi thời không được cập nhật**: agent áp dụng quy trình/kiến thức đã sai lệch so
  với thực tế hiện tại mà không có cơ chế phát hiện.

## 12. Security considerations

Skill là nội dung hướng dẫn agent sẽ tuân theo gần như tuyệt đối — cần coi trọng nguồn gốc skill
tương đương một phần của system prompt: chỉ dùng skill từ nguồn đáng tin cậy, rà soát nội dung trước
khi đưa vào dùng chung trong tổ chức, tránh trường hợp một skill độc hại (chứa hướng dẫn ẩn) bị nạp
và ảnh hưởng hành vi agent tương tự rủi ro prompt injection (liên hệ Input guardrail, mục 29.4).

## 13. Observability

Log skill nào được chọn cho mỗi tác vụ và tại sao (dựa trên khớp description) giúp debug khi agent
áp dụng sai quy trình; theo dõi tần suất mỗi skill được dùng để biết skill nào thực sự hữu ích, skill
nào nên loại bỏ hoặc cập nhật.

## 14. Evaluation metrics

Tỷ lệ agent chọn đúng skill phù hợp khi có skill khớp tác vụ khả dụng; chất lượng thực hiện tác vụ
khi có skill so với không có skill (đo giá trị thực tế skill mang lại); chi phí context bổ sung khi
load skill so với lợi ích đạt được.

## 15. Ưu điểm

Đơn giản hơn đáng kể so với MCP server hay A2A agent (chỉ là một file Markdown, không cần hạ tầng
service riêng); tái sử dụng được qua nhiều agent host khác nhau nhờ là chuẩn mở; dễ viết, dễ chia
sẻ, dễ rà soát nội dung.

## 16. Nhược điểm

Không phù hợp cho năng lực cần gọi ra ngoài hệ thống (đó là việc của MCP) hay cần một agent độc lập
xử lý (đó là việc của A2A) — chỉ giải quyết đúng phạm vi "hướng dẫn agent tự thực hiện trong context
của chính nó".

## 17. Khi nên dùng

Khi cần đóng gói một quy trình/kiến thức chuyên biệt để agent áp dụng lại, không đòi hỏi gọi dịch vụ
bên ngoài hay uỷ quyền cho agent khác — đặc biệt khi muốn skill đó tái sử dụng được qua nhiều agent
host khác nhau.

## 18. Khi không nên dùng

Khi năng lực cần thiết là gọi một hệ thống/API bên ngoài (dùng MCP, file 25) hoặc uỷ quyền một phần
công việc cho agent độc lập khác (dùng A2A, file 26) — Agent Skills không thay thế được hai chuẩn
này.

## 19. Pattern liên quan

Bổ sung cho Just-in-time context loading (mục 24.4) — cơ chế discovery nhẹ rồi load đầy đủ khi cần
là ứng dụng trực tiếp của pattern đó; phân biệt rõ với MCP (file 25, agent→tool) và A2A (file 26,
agent→agent) qua bảng so sánh ba chuẩn trong README chính.

## 20. Ví dụ kiến trúc thực tế

Đội kỹ thuật viết một skill "Tuân theo coding style guide nội bộ" mô tả chi tiết quy ước đặt tên,
cấu trúc file, cách viết commit message của công ty — agent coding tool tự động nạp skill này khi
phát hiện tác vụ liên quan tới viết/sửa code trong repo công ty, không cần nhúng toàn bộ style guide
vào system prompt cố định cho mọi tác vụ (kể cả tác vụ không liên quan tới code), và cùng một skill
này dùng lại được trên nhiều agent coding tool khác nhau trong đội mà không cần viết lại.

---

*Nguồn tham khảo dùng khi biên soạn: công bố công khai về chuẩn Agent Skills và định dạng SKILL.md
(Anthropic, 12/2025). Nội dung là tổng hợp và diễn giải lại, không trích dẫn nguyên văn đặc tả.*
