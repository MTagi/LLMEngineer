# Phụ lục (Appendix)

## Thiết lập Microsoft Foundry

Với khoá học này, bạn có thể dùng GitHub Models hoặc Microsoft Foundry để truy cập LLM. GitHub Models là lựa chọn tốt cho nhiều người dùng, nhưng nếu bạn có Azure subscription, bạn có thể dùng Microsoft Foundry cho ứng dụng AI cấp production với các tính năng doanh nghiệp. Nếu bạn mới dùng Azure, bạn có thể lấy tài khoản Azure MIỄN PHÍ với 200 USD, bằng cách đăng ký [tại đây](https://azure.microsoft.com/pricing/purchase-options/azure-account/?cid=msft_devrel-yt). Phụ lục này sẽ hướng dẫn bạn thiết lập một project Microsoft Foundry và deploy model cho khoá học.

## Thiết lập từng bước

Bạn có thể làm theo các bước dưới đây để deploy model trong Microsoft Foundry, hoặc xem [Deploy an Azure OpenAI model quickstart](https://learn.microsoft.com/azure/ai-foundry/quickstarts/get-started-code?tabs=azure-ai-foundry) cũng hướng dẫn quy trình này.

### 1. Tạo Project Microsoft Foundry

1. Truy cập [Microsoft Foundry portal](https://ai.azure.com/)
2. Đăng nhập bằng tài khoản Azure
3. Bấm **+ New project**
4. Điền thông tin project:
   - **Project name**: `langchain-course` (hoặc tên bạn thích)
   - **Subscription**: Chọn Azure subscription của bạn
   - **Resource group**: Tạo mới hoặc chọn cái có sẵn
   - **Region**: Chọn region gần bạn (vd. East US, West Europe)
5. Bấm **Create** (portal sẽ tự động thiết lập các resource cần thiết)

### 2. Deploy các Model cần thiết

Bạn cần deploy hai model cho khoá học này:

**Deploy gpt-5-mini & gpt-5 (Chat Models):**

1. Trong project của bạn, vào **Models + endpoints** ở thanh điều hướng bên trái
2. Bấm **+ Deploy model** → **Deploy base model**
3. Tìm và chọn **gpt-5-mini**
4. Bấm **Confirm**
5. Cấu hình deployment:
   - **Deployment name**: `gpt-5-mini` (giữ nguyên tên này để nhất quán)
   - **Model version**: Chọn phiên bản mới nhất có sẵn
   - **Deployment type**: Global Standard
   - Bấm **Deploy**
6. Đợi deployment hoàn tất
7. Làm tương tự để deploy `gpt-5`

> **Vì sao deploy cả hai model?** `gpt-5-mini` được dùng xuyên suốt khoá học cho hầu hết ví dụ (nhanh hơn và tiết kiệm chi phí hơn). `gpt-5` được dùng ở Chương 1 cho bài tập so sánh model, để minh hoạ sự khác biệt về hiệu năng và khả năng giữa các model.

**Deploy Text Embedding Model:**

1. Bấm **+ Deploy model** → **Deploy base model** lần nữa
2. Tìm và chọn **text-embedding-3-small**
3. Bấm **Confirm**
4. Cấu hình deployment:
   - **Deployment name**: `text-embedding-3-small` (giữ nguyên tên này)
   - **Model version**: Chọn phiên bản mới nhất có sẵn
   - **Deployment type**: Global Standard
   - Bấm **Deploy**
5. Đợi deployment hoàn tất

### 3. Lấy các giá trị cấu hình

Sau khi deploy model, bạn cần hai thông tin:

1. **API Key**:
   - Trong project của bạn, vào **Overview** ở thanh điều hướng bên trái
   - Tìm **Endpoints and keys**
   - Xác định **API Key**

2. **Endpoint URL**:
   - Xác định giá trị **Azure OpenAI** → **Azure OpenAI endpoint** (dạng: `https://your-resource.openai.azure.com`)

### 4. Thêm API Key và Endpoint vào file `.env`

Đảm bảo bạn thêm `/openai/v1` vào cuối endpoint URL.

```bash
# Cấu hình Microsoft Foundry
AI_API_KEY=your_azure_api_key_here
AI_ENDPOINT=https://your-resource.openai.azure.com/openai/v1
AI_MODEL=gpt-5-mini
```

**Thay `your_azure_api_key_here` bằng API key Azure thực tế và cập nhật endpoint URL!**

### Vì sao chọn Microsoft Foundry?

- ✅ **Sẵn sàng cho production**: Hạ tầng và SLA cấp doanh nghiệp
- ✅ **Giới hạn cao hơn**: Nhiều request/phút hơn so với gói miễn phí
- ✅ **Tính năng bổ sung**: Private endpoint, content filtering, monitoring
- ✅ **Tích hợp Azure**: Hoạt động liền mạch với các dịch vụ Azure khác

---

## Quay lại Course Setup

Sau khi hoàn thành thiết lập Azure, quay lại [hướng dẫn thiết lập chính](./README.md) và tiếp tục kiểm tra thiết lập.

## 🐛 Xử lý sự cố

**Giải pháp**:

1. Đảm bảo file `.env` tồn tại ở thư mục gốc project
2. Kiểm tra `.env` có đủ các biến bắt buộc:
   - `AI_API_KEY=your_key`
   - `AI_ENDPOINT=your_endpoint_url`
   - `AI_MODEL=gpt-5-mini`
3. Không cần dấu ngoặc kép quanh giá trị
4. Không có khoảng trắng trước/sau dấu `=`

### Lỗi: "401 Unauthorized" hoặc "Invalid token"

**Giải pháp**:

1. Tạo GitHub Personal Access Token mới
2. Đảm bảo bạn đã copy toàn bộ token
3. Token nên bắt đầu bằng `ghp_` hoặc `github_pat_`
4. Kiểm tra khoảng trắng thừa trong file `.env`

### Lỗi: Rate limit

**Giải pháp**: GitHub Models có giới hạn rate limit. Nếu bạn chạm giới hạn:

- Đợi vài phút
- Giới hạn sẽ reset nhanh
- Bạn có thể dùng Microsoft Foundry thay thế nếu đã làm theo phần thiết lập tuỳ chọn ở trên
