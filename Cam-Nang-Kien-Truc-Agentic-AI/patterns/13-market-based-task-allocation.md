# 13. Market-based task allocation

[← Về mục lục chính](../README.md)

## 1. Tên pattern

Market-based task allocation — worker đưa ra bid cạnh tranh, coordinator chọn worker theo utility
tốt nhất, mượn nguyên lý thị trường để phân bổ công việc.

## 2. Vấn đề cần giải quyết

Khi có nhiều worker agent cùng khả năng đảm nhận một task (hoặc các phần khác nhau của một task
lớn), việc gán task theo rule cố định (ví dụ round-robin, hoặc luôn gán cho agent đầu tiên khớp
capability) bỏ lỡ thông tin quan trọng: agent nào đang rảnh, agent nào ước tính làm nhanh/rẻ hơn,
agent nào tự tin hơn với task cụ thể này. Market-based allocation để chính các worker "tự định
giá" khả năng của mình cho từng task, cho phép coordinator ra quyết định gán việc tối ưu hơn dựa
trên thông tin thời gian thực thay vì rule tĩnh.

## 3. Bối cảnh sử dụng

Hệ thống có nhiều worker agent tương đối đồng nhất về khả năng nhưng khác nhau về tải hiện tại,
chi phí vận hành (agent dùng model rẻ vs đắt), hoặc mức độ chuyên biệt hoá cho một số loại task
con.

## 4. Kiến trúc

```
Coordinator: "Cần thực hiện Task X"
        ↓ broadcast tới các worker khả dụng
Worker A: bid {capability: cao, cost: $, load hiện tại: thấp, ETA: 2 phút, confidence: 0.9}
Worker B: bid {capability: trung bình, cost: $$, load hiện tại: cao, ETA: 5 phút, confidence: 0.6}
Worker C: không bid (ngoài phạm vi capability)
        ↓
Coordinator chọn Worker A (utility tốt nhất theo tiêu chí kết hợp cost/ETA/confidence)
```

## 5. Thành phần

Cơ chế broadcast task tới worker (có thể qua event bus — liên hệ file 19); hàm tính bid của mỗi
worker (dựa trên capability, tải hiện tại, chi phí ước tính, độ tự tin); hàm utility của
coordinator (kết hợp các tiêu chí bid thành một quyết định chọn worker).

## 6. Luồng xử lý chi tiết

Coordinator công bố task cần thực hiện (broadcast, không gán trực tiếp cho một worker cụ thể); mỗi
worker khả dụng tự đánh giá và đưa ra bid gồm các tiêu chí: capability (mức độ phù hợp với task),
cost (chi phí ước tính — có thể tính bằng token/tiền/thời gian), current load (tải hiện tại, worker
đang bận nên bid kém hấp dẫn hơn), estimated completion time, và confidence (độ tự tin hoàn thành
đúng). Coordinator tổng hợp các bid, áp dụng hàm utility (có thể đơn giản là trọng số cố định hoặc
phức tạp hơn tuỳ ngữ cảnh) để chọn worker tối ưu, rồi gán task chính thức cho worker đó.

## 7. State và dữ liệu

Coordinator cần theo dõi trạng thái worker (danh sách worker khả dụng, capability mỗi worker) để
biết broadcast task tới ai; không cần lưu lịch sử bid lâu dài trừ khi muốn phân tích/tối ưu hàm
utility theo thời gian.

## 8. Thuật toán liên quan

Không có thuật toán chuẩn cố định — hàm utility có thể từ đơn giản (chọn bid có cost thấp nhất
thoả confidence tối thiểu) đến phức tạp (mô phỏng đấu giá kiểu combinatorial auction nếu nhiều
task cần gán đồng thời cho nhiều worker).

## 9. Cách triển khai

1. Định nghĩa rõ các tiêu chí bid worker cần cung cấp (capability, cost, load, ETA, confidence) —
   thống nhất định dạng để coordinator so sánh được công bằng giữa các worker khác nhau.
2. Bắt đầu với hàm utility đơn giản (ví dụ ưu tiên confidence cao nhất trong ngưỡng cost chấp
   nhận được) trước khi phức tạp hoá.
3. Giới hạn thời gian chờ bid (timeout) — nếu một worker chậm phản hồi bid, coordinator không nên
   chờ vô thời hạn trước khi quyết định.
4. Log lại các quyết định gán việc và kết quả thực tế để đánh giá xem hàm utility có đang chọn
   đúng worker hay không theo thời gian.

## 10. Tham số cần tuning

Timeout chờ bid; trọng số giữa các tiêu chí trong hàm utility (cost vs confidence vs ETA); ngưỡng
confidence tối thiểu để một bid được coi là hợp lệ.

## 11. Failure modes

- **Không worker nào bid**: task ngoài phạm vi capability của mọi worker hiện có — cần fallback
  (escalate lên con người hoặc báo lỗi rõ ràng) thay vì treo vô thời hạn.
- **Race condition khi nhiều task broadcast cùng lúc**: cùng một worker bid cho nhiều task song
  song dựa trên "load hiện tại" đã lỗi thời (chưa tính các bid đang chờ khác) — dẫn tới overload
  thực tế dù bid ban đầu trông hợp lý.
- **Bid không trung thực**: worker luôn báo confidence cao để được chọn (nếu không có cơ chế đối
  chiếu ngược kết quả thực tế với bid), làm hàm utility mất ý nghĩa theo thời gian.

## 12. Security considerations

Nếu Worker thuộc các domain quyền khác nhau (liên hệ Federated multi-agent, file 12), cần đảm bảo
việc bid/broadcast task không làm lộ nội dung task cho worker không có quyền truy cập dữ liệu liên
quan, ngay cả khi worker đó cuối cùng không được chọn.

## 13. Observability

Log toàn bộ bid nhận được cho mỗi task (không chỉ worker được chọn) — cần thiết để phân tích sau
này liệu hàm utility có đang chọn đúng, và để phát hiện worker bid không trung thực.

## 14. Evaluation metrics

Tỷ lệ task hoàn thành đúng ETA đã bid; độ lệch giữa confidence đã bid và kết quả thực tế (worker
liên tục over-confident cần được điều chỉnh trọng số thấp hơn trong hàm utility); tổng chi phí hệ
thống so với phân bổ theo rule cố định làm baseline so sánh.

## 15. Ưu điểm

Phân bổ task thích nghi theo tải và khả năng thời gian thực thay vì rule tĩnh, có thể cải thiện
đáng kể hiệu quả sử dụng tài nguyên khi có nhiều worker không đồng nhất về tải/chi phí.

## 16. Nhược điểm

Phức tạp hơn đáng kể so với gán task trực tiếp; cần cơ chế tin cậy giữa các worker (hoặc cơ chế
đối chiếu để phát hiện bid không trung thực); overhead giao tiếp (broadcast + chờ bid) làm tăng
latency so với gán trực tiếp.

## 17. Khi nên dùng

Khi có đủ nhiều worker (thường ≥ 3-4) với sự khác biệt thực sự về tải/chi phí/chuyên môn hoá đáng
để đánh đổi overhead phối hợp lấy hiệu quả phân bổ tốt hơn.

## 18. Khi không nên dùng

Với ít worker hoặc worker gần như đồng nhất về khả năng — Contract-net pattern (file 14, đơn giản
hơn) hoặc gán trực tiếp theo rule là đủ, không cần cơ chế thị trường phức tạp.

## 19. Pattern liên quan

Gần với Contract-net pattern (file 14) — cả hai đều dựa trên cơ chế "công bố việc, nhận đề xuất,
chọn người thực hiện", khác biệt chính ở chỗ market-based nhấn mạnh nhiều tiêu chí cạnh tranh
(cost/load/confidence) còn contract-net thường đơn giản hơn ở vòng đề xuất-trao hợp đồng.

## 20. Ví dụ kiến trúc thực tế

Hệ thống xử lý yêu cầu dịch thuật với nhiều worker agent chuyên các cặp ngôn ngữ và mức độ phức
tạp khác nhau: coordinator broadcast mỗi yêu cầu dịch, các worker phù hợp bid theo cặp ngôn ngữ họ
xử lý tốt, tải hiện tại, và độ tự tin với loại văn bản (kỹ thuật/pháp lý/thông thường); coordinator
chọn worker có utility tốt nhất, cân bằng giữa chất lượng dự kiến và thời gian hoàn thành.

---

*Nguồn tham khảo dùng khi biên soạn: kiến thức chung về cơ chế phân bổ dựa trên đấu giá (market-
based allocation) trong hệ thống multi-agent. Nội dung là tổng hợp và diễn giải lại.*
