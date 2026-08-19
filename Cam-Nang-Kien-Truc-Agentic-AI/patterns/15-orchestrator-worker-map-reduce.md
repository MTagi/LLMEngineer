# 15. Orchestrator-Worker / Map-reduce agents

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Orchestrator-Worker (còn gọi Map-reduce agents) — một agent điều phối lập chiến lược và chia việc
cho nhiều worker agent chạy song song, rồi tổng hợp kết quả.

## 2. Vấn đề cần giải quyết

Nhiều bài toán nghiên cứu/khám phá có tính "breadth-first": câu trả lời đòi hỏi khám phá nhiều
hướng độc lập cùng lúc, và tổng lượng thông tin cần xử lý vượt quá những gì một agent đơn lẻ có
thể giữ trong một context window (liên hệ Context rot, mục 24.5). Chạy tuần tự từng hướng một agent
đơn sẽ chậm và dễ context rot; Orchestrator-Worker giải quyết bằng cách **song song hoá** việc khám
phá, mỗi worker giữ context riêng, chỉ trả kết quả cô đọng về cho orchestrator.

## 3. Bối cảnh sử dụng

Task nghiên cứu/tổng hợp thông tin từ nhiều nguồn độc lập; task có thể chia thành các phần con
không phụ thuộc lẫn nhau (có thể chạy song song thực sự, không chỉ tuần tự giả song song); task mà
chất lượng quan trọng hơn chi phí (vì pattern này tốn kém).

## 4. Kiến trúc

```
Large Input
   ↓
Lead Agent (Orchestrator) — phân tích yêu cầu, lập chiến lược, xác định các hướng khám phá
   ↓
   ├─ Worker Agent 1 (context riêng, tool riêng, hướng khám phá A)
   ├─ Worker Agent 2 (context riêng, tool riêng, hướng khám phá B)
   └─ Worker Agent 3 (context riêng, tool riêng, hướng khám phá C)
   ↓ (mỗi worker chỉ trả KẾT QUẢ CUỐI, không trả toàn bộ reasoning/tool-call trung gian)
Synthesis / Reduce Agent — tổng hợp kết quả từ các worker
   ↓ (thường kèm 1 pass riêng để kiểm tra/gắn citation)
Final Result
```

## 5. Thành phần

Lead agent/Orchestrator (lập chiến lược, phân rã task, không tự khám phá chi tiết); nhiều Worker
agent chạy song song, mỗi worker có context window/tool/hướng khám phá riêng (liên hệ Sub-agent
context isolation, mục 24.3); bước Synthesis tổng hợp; (tuỳ chọn) bước citation pass riêng biệt để
kiểm tra nguồn trước khi trả kết quả cuối.

## 6. Luồng xử lý chi tiết

Lead agent nhận yêu cầu, phân tích và quyết định chia thành các hướng khám phá độc lập (map);
spawn các worker agent tương ứng chạy **song song thực sự** (không tuần tự) — mỗi worker có
context riêng biệt hoàn toàn với lead agent và với các worker khác, tự thực hiện reasoning/tool-use
của riêng mình trong phạm vi được giao. Khi hoàn thành, mỗi worker chỉ trả về **kết quả cô đọng
cuối cùng**, không trả toàn bộ lịch sử reasoning/tool call trung gian về cho lead agent — đây là cơ
chế cốt lõi giữ context của lead agent không phình theo độ phức tạp của task (liên hệ trực tiếp
Sub-agent context isolation, mục 24.3). Cuối cùng, một bước synthesis (có thể do chính lead agent
hoặc một agent tổng hợp riêng) gộp kết quả các worker thành câu trả lời cuối, thường kèm một bước
riêng kiểm tra/gắn citation trước khi trả về.

## 7. State và dữ liệu

Lead agent chỉ giữ state ở mức điều phối (hướng nào đã giao, worker nào đã xong, kết quả cô đọng
của từng worker) — không giữ bản sao context chi tiết của từng worker. Mỗi worker giữ state/context
độc lập, có thể bị huỷ hoàn toàn sau khi trả kết quả (không cần giữ lại).

## 8. Thuật toán liên quan

Không có thuật toán chuẩn — về bản chất là mô hình lập trình Map-Reduce cổ điển (chia việc, xử lý
song song, gộp kết quả) áp dụng vào ngữ cảnh agent thay vì xử lý dữ liệu lớn truyền thống.

## 9. Cách triển khai

1. Lead agent nên có prompt/hướng dẫn rõ ràng về việc **chia task thế nào** — hướng dẫn mơ hồ dẫn
   tới worker chồng chéo phạm vi hoặc bỏ sót hướng khám phá quan trọng.
2. Mỗi worker cần một mục tiêu cụ thể, ranh giới phạm vi rõ ràng, định dạng output kỳ vọng, và
   hướng dẫn nên dùng tool nào — thiếu rõ ràng ở đây là nguyên nhân phổ biến khiến kết quả worker
   không hữu ích cho bước synthesis.
3. Giới hạn số worker song song theo ngân sách chi phí đã chấp nhận trước (liên hệ mục 10 bên
   dưới) — không để lead agent tự do spawn số lượng worker không giới hạn.
4. Tách bước citation/kiểm tra nguồn thành một pass riêng sau synthesis, không gộp chung vào bước
   tổng hợp nội dung — giúp kiểm tra độ chính xác trích dẫn độc lập với chất lượng tổng hợp nội
   dung.

## 10. Tham số cần tuning

Số worker tối đa chạy song song (3-5 là khoảng phổ biến trong thực tế đã ghi nhận); ngân sách
token/chi phí tối đa cho toàn bộ task (bao gồm cả lead agent lẫn tất cả worker); timeout cho từng
worker (một worker chậm không nên chặn toàn bộ synthesis — liên hệ Graceful degradation, mục 30.7).

## 11. Failure modes

- **Chi phí vượt kiểm soát**: đây là pattern **tốn kém nhất** trong nhóm multi-agent (theo case
  study thực tế đã ghi nhận, có thể tốn tới hàng chục lần chi phí một lượt chat thông thường) — cần
  ngân sách rõ ràng, không áp dụng mặc định cho mọi loại câu hỏi.
- **Worker chồng chéo phạm vi**: hai worker vô tình khám phá cùng một hướng do lead agent chia
  việc không rõ ràng, lãng phí chi phí song song mà không tăng độ phủ thông tin.
- **Synthesis mất thông tin quan trọng**: bước tổng hợp tóm tắt quá mạnh tay, làm mất chi tiết quan
  trọng mà worker đã tìm được — cần cân bằng giữa cô đọng và giữ đủ thông tin.
- **Áp dụng cho task đơn giản**: dùng Orchestrator-Worker cho câu hỏi mà một agent đơn giải quyết
  tốt — chi phí cao hơn nhiều so với lợi ích thực tế mang lại.

## 12. Security considerations

Mỗi worker cần tuân theo Least-privilege agent (mục 29.2) riêng theo phạm vi được giao — không nên
cấp cho mọi worker toàn bộ quyền truy cập của lead agent, vì worker chạy độc lập, khó giám sát trực
tiếp trong lúc đang chạy song song.

## 13. Observability

Trace phải ghi rõ: chiến lược chia việc của lead agent, từng worker được giao gì, kết quả cô đọng
mỗi worker trả về, và bước synthesis đã dùng những kết quả nào — vì bản chất "context isolation"
khiến việc debug khó hơn nhóm agent đơn (không thể chỉ nhìn một luồng context liên tục).

## 14. Evaluation metrics

Liên hệ Multi-agent evaluation (file 33): tỷ lệ hoàn thành task đúng so với chi phí bỏ ra (quan
trọng đặc biệt với pattern tốn kém này); độ phủ thông tin so với chạy tuần tự bằng agent đơn (đo
được lợi ích thực sự của việc song song hoá); tỷ lệ worker trả kết quả hữu ích cho synthesis (không
lạc đề/trùng lặp).

## 15. Ưu điểm

Ghi nhận cải thiện đáng kể (case study thực tế cho thấy vượt trội rõ rệt so với agent đơn) trên các
bài toán nghiên cứu breadth-first phức tạp; giải quyết trực tiếp giới hạn context window của agent
đơn nhờ context isolation.

## 16. Nhược điểm

Chi phí token cao hơn đáng kể (nhiều lần) so với chạy agent đơn cho cùng câu hỏi — cần đánh giá kỹ
trade-off chi phí/chất lượng trước khi dùng, đặc biệt với hệ thống có volume request lớn.

## 17. Khi nên dùng

Câu hỏi nghiên cứu breadth-first thực sự cần khám phá nhiều hướng độc lập và tổng thông tin vượt
quá một context window — không phải mọi câu hỏi phức tạp đều thuộc loại này.

## 18. Khi không nên dùng

Với câu hỏi có thể trả lời tốt bằng agent đơn (kể cả câu hỏi cần nhiều bước nhưng tuần tự, không
độc lập với nhau) — chi phí Orchestrator-Worker không tương xứng lợi ích trong trường hợp này.

## 19. Pattern liên quan

Là dạng cụ thể hoá của Map-reduce agents kết hợp với Sub-agent context isolation (mục 24.3);
tương tự Committee-of-experts (file 16) nhưng khác ở việc worker khám phá **hướng khác nhau** thay
vì cùng đánh giá một vấn đề từ nhiều góc nhìn chuyên môn.

## 20. Ví dụ kiến trúc thực tế

Hệ thống nghiên cứu sâu đã được ghi nhận trong thực tế công nghiệp (2025): lead agent phân tích câu
hỏi nghiên cứu, lập chiến lược tìm kiếm, spawn 3-5 subagent chuyên biệt chạy song song, mỗi subagent
có context window/tool/hướng khám phá riêng; kết quả tổng hợp qua một bước synthesis tách biệt kèm
một pass kiểm tra citation riêng. Ghi nhận cải thiện chất lượng đáng kể so với agent đơn trên câu
hỏi nghiên cứu phức tạp, đổi lại chi phí token cao hơn nhiều lần so với một lượt chat thông thường
— minh hoạ rõ đánh đổi cốt lõi của pattern này.

---

*Nguồn tham khảo dùng khi biên soạn: tổng hợp và diễn giải lại từ các báo cáo kỹ thuật công khai về
hệ thống nghiên cứu đa-agent dạng orchestrator-worker trong công nghiệp (2025), không trích dẫn
nguyên văn.*
