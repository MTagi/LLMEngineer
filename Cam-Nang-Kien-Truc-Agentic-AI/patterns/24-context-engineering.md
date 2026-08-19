# 24. Context Engineering

[← Về mục lục chính](../README.md)

5 kỹ thuật quản lý **những gì thực sự nằm trong context window** tại mỗi lượt gọi model: Compaction,
Structured note-taking, Sub-agent context isolation, Just-in-time context loading, Context rot.

## 1. Tên pattern

Compaction · Structured note-taking · Sub-agent context isolation · Just-in-time context loading ·
Context rot (failure mode cần theo dõi).

## 2. Vấn đề cần giải quyết

Ba khái niệm dễ bị nhầm lẫn: **memory** (nội dung được lưu lâu dài, file 22), **prompt engineering**
(cách viết một prompt tĩnh), và **context engineering** (cách chọn, nén, sắp xếp thông tin đưa vào
context window tại một thời điểm cụ thể). Với agent chạy nhiều bước/nhiều lượt, context window
không phải tài nguyên vô hạn — nạp quá nhiều thông tin (kể cả thông tin "đúng" và liên quan) làm
giảm chất lượng phản hồi (Context rot), tốn chi phí, và tăng độ trễ. Context engineering là tập kỹ
thuật chủ động quản lý vấn đề này thay vì để context tự phình đến khi gặp sự cố.

## 3. Bối cảnh sử dụng

Agent chạy nhiều bước (nhiều lượt gọi model liên tiếp trong một task); hội thoại kéo dài nhiều lượt
qua lại; hệ thống multi-agent cần kiểm soát context của từng agent con độc lập với orchestrator.

## 4. Kiến trúc

```
Compaction:                Context gần đầy → Summarize toàn bộ → Khởi tạo context mới = summary +
                           thông tin bắt buộc giữ nguyên (vd. system prompt) → Tiếp tục task

Structured note-taking:     Agent ghi chú RA NGOÀI context (file/scratchpad/task list) →
                           Khi cần, ĐỌC LẠI note thay vì phụ thuộc context đã bị cắt/compact

Sub-agent isolation:        Orchestrator context
                             └─ Subagent: context RIÊNG, không lộ về Orchestrator
                                → Chỉ trả KẾT QUẢ CUỐI (không trả toàn bộ reasoning trung gian)

Just-in-time loading:       Agent chỉ load thông tin KHI THỰC SỰ CẦN (ví dụ đọc file khi chuẩn bị
                           sửa file đó) thay vì nạp trước toàn bộ (ví dụ toàn bộ codebase)
```

## 5. Thành phần

Cơ chế tóm tắt (thường một lệnh gọi LLM riêng cho compaction); scratchpad/file lưu note có thể đọc
lại (structured note-taking); ranh giới context riêng cho mỗi subagent (sub-agent isolation, liên
hệ trực tiếp Orchestrator-Worker, file 15); logic quyết định "khi nào cần load thông tin gì" (just-
in-time loading).

## 6. Luồng xử lý chi tiết

- **Compaction**: khi context gần chạm giới hạn (theo ngưỡng token đã định trước, không đợi đến
  sát giới hạn cứng), hệ thống tóm tắt toàn bộ nội dung hội thoại/task hiện có, rồi khởi tạo lại
  context mới chỉ chứa bản tóm tắt cộng các phần bắt buộc phải giữ nguyên (system prompt, ràng
  buộc quan trọng) — phù hợp nhất với task cần nhiều lượt qua lại liên tục (hội thoại dài, task
  tương tác cao) nơi việc mất chi tiết cũ ít ảnh hưởng miễn giữ được mạch chính.
- **Structured note-taking**: thay vì phụ thuộc hoàn toàn vào context window để "nhớ" mọi thứ,
  agent chủ động ghi thông tin quan trọng ra một nơi lưu trữ ngoài context (file, scratchpad, danh
  sách task có thể persist) — khi cần, agent đọc lại note thay vì trông chờ thông tin đó vẫn còn
  trong context (vốn có thể đã bị compact hoặc cắt bớt). Phù hợp nhất với task nhiều vòng lặp, có
  các mốc rõ ràng.
- **Sub-agent context isolation**: khi giao một subtask cho subagent (liên hệ file 15), subagent
  đó chạy trong context window hoàn toàn riêng biệt với orchestrator. Orchestrator chỉ nhận kết
  quả cuối cùng của subagent, không nhận toàn bộ lịch sử reasoning/tool-call trung gian — nhờ vậy
  context của orchestrator không phình theo độ phức tạp của subtask, bất kể subagent đã "vật lộn"
  qua bao nhiêu bước để đạt kết quả đó.
- **Just-in-time context loading**: thay vì nạp toàn bộ tài liệu/tool schema/dữ liệu liên quan vào
  context ngay từ đầu task, agent chỉ load thông tin **khi thực sự chuẩn bị dùng đến** — ví dụ chỉ
  đọc nội dung một file khi chuẩn bị chỉnh sửa nó, không đọc trước toàn bộ codebase "để phòng khi
  cần". Giảm token lãng phí và giảm nhiễu khiến model khó tập trung vào phần thực sự liên quan tới
  bước hiện tại.
- **Context rot**: đây không phải một kỹ thuật để áp dụng mà là một **failure mode cần chủ động
  theo dõi và phòng tránh** — hiện tượng chất lượng phản hồi của agent suy giảm khi context đã dài
  ra, ngay cả khi tổng số token vẫn còn trong giới hạn cửa sổ context của model. Không phải lỗi do
  vượt giới hạn cứng, mà là suy giảm khả năng "chú ý" đúng phần liên quan khi lượng thông tin không
  liên quan tích luỹ quá nhiều trong context. Cách giảm thiểu: áp dụng Compaction/Structured
  note-taking **chủ động, trước khi** context quá dài (không đợi đến khi chạm giới hạn), và ưu tiên
  Just-in-time loading thay vì nạp trước toàn bộ.

## 7. State và dữ liệu

Note từ Structured note-taking cần được lưu bền vững (liên hệ file 23 — thường là durable state
nếu note cần sống sót qua nhiều lượt/nhiều compaction); bản tóm tắt từ Compaction có thể coi là một
dạng Summary memory (mục 22.3) nếu cần giữ lại lâu dài, hoặc chỉ tạm thời nếu chỉ dùng để tiếp tục
ngay task hiện tại.

## 8. Thuật toán liên quan

Không có thuật toán chuẩn hoá — chủ yếu dựa vào khả năng tóm tắt của chính LLM (cho compaction) và
logic điều khiển (rule-based hoặc do agent tự quyết định) cho việc khi nào ghi note, khi nào load
thông tin.

## 9. Cách triển khai

1. Đặt ngưỡng trigger Compaction **trước** khi chạm giới hạn cứng của context window (ví dụ ở 70-
   80% giới hạn), không đợi đến sát giới hạn mới xử lý — chủ động phòng Context rot thay vì phản
   ứng khi đã xảy ra.
2. Với task nhiều vòng lặp có cấu trúc rõ (ví dụ phát triển phần mềm nhiều giai đoạn), ưu tiên
   Structured note-taking hơn Compaction thuần — note có cấu trúc dễ tra cứu lại chính xác hơn một
   bản tóm tắt tự do.
3. Mặc định áp dụng Sub-agent context isolation cho bất kỳ kiến trúc Orchestrator-Worker nào (file
   15) — đây gần như là điều kiện bắt buộc để pattern đó hoạt động đúng như thiết kế, không phải
   một tuỳ chọn.
4. Thiết kế logic Just-in-time loading dựa trên bước hiện tại của task (liên hệ Goal decomposition,
   mục 10.7) — biết agent đang ở action nào giúp xác định chính xác thông tin nào thực sự cần ngay
   bây giờ.

## 10. Tham số cần tuning

Ngưỡng token trigger compaction; độ dài tối đa cho phép của note (structured note-taking); số cấp
độ chi tiết trong summary (một số hệ thống dùng summary nhiều tầng — chi tiết gần, khái quát xa);
kích thước tối đa context của mỗi subagent trước khi chính subagent đó cũng cần compaction nội bộ.

## 11. Failure modes

- **Compaction quá muộn**: đợi đến khi gần chạm giới hạn cứng mới compact, trong khi Context rot
  đã âm thầm làm giảm chất lượng từ trước đó khá lâu.
- **Note không được đọc lại đúng lúc**: agent ghi note nhưng logic không kích hoạt việc đọc lại khi
  cần, khiến structured note-taking trở thành ghi chép vô ích không phục vụ mục đích.
- **Sub-agent isolation rò rỉ**: vô tình để orchestrator nhận toàn bộ log/reasoning trung gian của
  subagent thay vì chỉ kết quả cuối, làm mất tác dụng cách ly context.
- **Just-in-time load thiếu**: logic quyết định "khi nào cần" sai, khiến agent thiếu thông tin quan
  trọng đúng lúc cần dùng, phải load lại nhiều lần gây chậm và tốn chi phí hơn nạp trước.

## 12. Security considerations

Note lưu ngoài context (structured note-taking) và bản tóm tắt (compaction) cần tuân theo cùng
chính sách ACL/quyền riêng tư như memory dài hạn khác (liên hệ mục 12 ở file 22) — không nên coi
đây là "dữ liệu tạm" ít quan trọng về bảo mật chỉ vì bản chất kỹ thuật của nó là cơ chế quản lý
context.

## 13. Observability

Theo dõi độ dài context theo thời gian trong một task dài, số lần compaction đã kích hoạt, và —
quan trọng nhất — theo dõi **chất lượng output tương quan với độ dài context** như một chỉ số riêng
để phát hiện sớm Context rot trước khi nó ảnh hưởng rõ rệt tới trải nghiệm người dùng.

## 14. Evaluation metrics

Chất lượng output đo theo từng khoảng độ dài context (nếu chất lượng giảm rõ rệt sau một ngưỡng độ
dài nhất định, đó là dấu hiệu cần compact sớm hơn); chi phí token trung bình mỗi task trước/sau khi
áp dụng just-in-time loading (đo lợi ích thực sự về chi phí); tỷ lệ agent cần load lại thông tin đã
bỏ qua do just-in-time loading quá tích cực.

## 15. Ưu điểm

Cho phép agent xử lý task dài mà không bị giới hạn cứng bởi context window, đồng thời chủ động
phòng tránh suy giảm chất lượng do context rot — là điều kiện cần cho các pattern multi-agent phức
tạp như Orchestrator-Worker (file 15) hoạt động hiệu quả.

## 16. Nhược điểm

Thêm độ phức tạp điều khiển (khi nào compact, khi nào ghi note, khi nào load) — thiết kế sai các
ngưỡng/logic này có thể tự nó gây ra vấn đề (mất thông tin do compact quá sớm/mạnh tay, hoặc chi
phí thêm do load thông tin nhiều lần).

## 17. Khi nên dùng

Bất kỳ agent nào có khả năng chạy đủ dài (nhiều bước, nhiều lượt hội thoại, hoặc task phức tạp) để
context window trở thành ràng buộc thực tế — không chỉ các hệ thống multi-agent lớn.

## 18. Khi không nên dùng

Với tương tác ngắn, một lượt gọi model đơn giản không có khả năng context bao giờ đạt tới ngưỡng
đáng lo ngại — không cần áp dụng các kỹ thuật này, thêm vào chỉ tăng độ phức tạp không cần thiết.

## 19. Pattern liên quan

Sub-agent context isolation liên hệ trực tiếp Orchestrator-Worker (file 15); Context rot liên hệ
Context rot mitigation trong Reliability patterns (mục 30.11); Structured note-taking liên hệ
Artifact memory (mục 22.8) khi note được coi là một loại artifact cần lưu trữ lâu dài.

## 20. Ví dụ kiến trúc thực tế

Agent hỗ trợ phát triển phần mềm chạy nhiều giờ cho một tính năng lớn: dùng Structured note-taking
để ghi lại tiến độ theo từng milestone (liên hệ Goal decomposition, mục 10.7) vào một file task
list persistent; áp dụng Just-in-time loading để chỉ đọc file code khi chuẩn bị sửa, không nạp toàn
bộ codebase từ đầu; khi cần thực hiện một nhánh công việc độc lập lớn (ví dụ viết toàn bộ test
suite), giao cho subagent riêng với context cách ly hoàn toàn, chỉ nhận về kết quả tóm tắt (test
nào pass/fail) thay vì toàn bộ log chạy test chi tiết.

---

*Nguồn tham khảo dùng khi biên soạn: tổng hợp và diễn giải lại các hướng dẫn kỹ thuật công khai về
context engineering cho AI agent (Anthropic, 2025) — bao gồm khái niệm compaction, structured
note-taking, sub-agent context isolation và context rot. Nội dung không trích dẫn nguyên văn.*
