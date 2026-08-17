# Course Setup (Thiết lập khoá học)

Chào mừng! Trước khi bắt đầu xây dựng ứng dụng AI với LangChain, hãy chuẩn bị môi trường phát triển. Chương này hướng dẫn bạn thiết lập môi trường phát triển và lấy GitHub token để truy cập AI model miễn phí (bạn cũng có thể dùng Microsoft Foundry). Kết thúc chương, bạn sẽ có đầy đủ mọi thứ cần thiết để bắt đầu xây dựng với LangChain.

## Yêu cầu trước

- Tài khoản GitHub (miễn phí)
- Kiến thức command line cơ bản
- Text editor hoặc IDE

---

## 📖 Ví von về xưởng làm việc (Workshop Analogy)

**Giống như việc chuẩn bị một xưởng làm việc trước khi đóng đồ nội thất, bạn cần chuẩn bị môi trường phát triển trước khi xây dựng ứng dụng AI.**

Bạn sẽ thiết lập môi trường phát triển, lấy quyền truy cập AI model, và cấu hình công cụ. Điều này đảm bảo bạn có nền tảng vững chắc và trải nghiệm phát triển mượt mà. Hãy chuẩn bị "xưởng làm việc" của bạn — chỉ mất 15 phút!

## Truy cập Model: GitHub Models vs Microsoft Foundry

Với khoá học này, bạn có thể dùng GitHub Models hoặc Microsoft Foundry để truy cập LLM. Để bắt đầu, chúng tôi khuyến nghị dùng GitHub Models.

### Vì sao chọn GitHub Models?

- ✅ **Miễn phí**: Không cần thẻ tín dụng
- ✅ **Mạnh mẽ**: Truy cập GPT-5, GPT-5-mini, và các model khác
- ✅ **Dễ dùng**: Dùng tài khoản GitHub sẵn có
- ✅ **Học tập**: Hoàn hảo cho khoá học này!

GitHub Models là lựa chọn tốt cho khoá học này, nhưng nếu bạn có Azure subscription, bạn có thể dùng Microsoft Foundry cho ứng dụng AI cấp production với các tính năng doanh nghiệp. Nếu bạn mới dùng Azure, bạn cũng có thể lấy tài khoản Azure MIỄN PHÍ với 200 USD, bằng cách đăng ký [tại đây](https://azure.microsoft.com/pricing/purchase-options/azure-account/?cid=msft_devrel-yt).

---

## Các lựa chọn thiết lập

Chọn một trong các lựa chọn sau để thiết lập môi trường phát triển. Cách dễ nhất để bắt đầu là GitHub Codespaces, vì nó tự thiết lập mọi công cụ cho bạn, nhưng bạn cũng có thể thiết lập ở local:

1. [**GitHub Codespaces**](#github-codespaces): Dùng môi trường phát triển trên cloud (**khuyến nghị**)
2. [**Local Development**](#local-development): Thiết lập môi trường trên máy của bạn.

---

## GitHub Codespaces

### Bước 1: Mở Codespace cho khoá học này

Bạn có thể mở khoá học này trong GitHub Codespaces. Nút bên dưới sẽ đưa bạn đến trang có nút xanh ghi `CREATE A CODESPACE`. Bấm nút đó và đợi codespace tải xong. Một VS Code chạy trên web sẽ mở trong trình duyệt khi tải xong.

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/microsoft/langchain-for-beginners)

Terminal sẽ tự động mở và bạn sẽ thấy `(.venv)` trước tên user. Điều này cho thấy Python virtual environment đã được kích hoạt. Môi trường này đã cài sẵn tất cả package Python cần cho khoá học.

---

### Bước 2: Kiểm tra file `.env`

Tất cả biến môi trường được tự động thiết lập cho bạn trong Codespaces.
GitHub Models được dùng mặc định và cung cấp quyền truy cập AI model miễn phí. Để dùng GitHub Models bạn cần Personal Access Token. Trong Codespaces, token này, endpoint và model đúng sẽ tự động được thêm vào file `.env`.

Đảm bảo file `.env` tồn tại và kiểm tra xem nó có đủ các biến bắt buộc sau không.

**Với GitHub Models (Miễn phí):**

```bash
AI_API_KEY=your_personal_token_will_show_up_here
AI_ENDPOINT=https://models.inference.ai.azure.com
AI_MODEL=gpt-5-mini
```

**Lựa chọn khác: Microsoft Foundry:**

Nếu bạn có Azure subscription, bạn có thể dùng Microsoft Foundry cho ứng dụng AI cấp production và nhiều token hơn. Xem [Phụ lục thiết lập Azure](./APPENDIX.md#microsoft-foundry-setup) để có hướng dẫn chi tiết về việc deploy model và cấu hình môi trường.

Vậy là xong 🎉 Giờ bạn có thể chuyển sang [kiểm tra thiết lập](#test-your-setup).

---

## Local Development (Phát triển tại local)

### Bước 1: Cài Python

Bạn cần **Python 3.10 trở lên** để chạy ứng dụng LangChain v1.

#### Kiểm tra xem Python đã được cài chưa

```bash
python --version
# hoặc
python3 --version
```

Nếu bạn thấy Python 3.10 trở lên, vậy là ổn! Bỏ qua đến [Bước 2](#bước-2-clone-repository). Nếu chưa:

#### Cài Python

1. Truy cập [python.org](https://www.python.org/downloads/)
2. Tải Python 3.10+ cho hệ điều hành của bạn
3. Làm theo hướng dẫn cài đặt (nhớ tick "Add Python to PATH" trên Windows)
4. Kiểm tra lại việc cài đặt:

```bash
python --version  # Nên hiện 3.10 trở lên
pip --version     # Nên hiện phiên bản pip
```

---

### Bước 2: Clone Repository

```bash
# Clone repository của khoá học
git clone https://github.com/microsoft/langchain-for-beginners

# Di chuyển vào thư mục project
cd langchain-for-beginners

# Tạo virtual environment
python -m venv venv

# Kích hoạt virtual environment
# Trên Mac/Linux:
source venv/bin/activate
# Trên Windows:
venv\Scripts\activate

# Cài đặt dependencies
pip install -r requirements.txt
```

Lệnh này sẽ cài tất cả package cần thiết, bao gồm:

- `langchain-openai` - Tích hợp model tương thích OpenAI
- `langchain-core` - Chức năng cốt lõi của LangChain
- `langchain` - Package chính của LangChain kèm tiện ích bổ sung
- `langchain-azure-ai` - Tích hợp LangChain riêng cho Azure
- `python-dotenv` - Quản lý biến môi trường cho API key

---

### Bước 3: Tạo GitHub Personal Access Token

GitHub Models cung cấp quyền truy cập miễn phí đến các AI model mạnh mẽ — bạn chỉ cần một Personal Access Token.

#### Tạo Token của bạn

1. **Truy cập**: https://github.com/settings/tokens/new
2. **Tên token**: `langchain-course` (hoặc tên bất kỳ bạn thích)
3. **Thời hạn**: Chọn theo ý bạn (khuyến nghị 90 ngày cho việc học)
4. **Scopes/Permissions**:
   - ✅ Không cần scope nào cho GitHub Models!
   - Bạn có thể để trống tất cả checkbox
5. **Bấm**: "Generate token"
6. **⚠️ QUAN TRỌNG**: Copy token ngay và lưu tạm vào file text! Bạn sẽ cần nó ở bước tiếp theo.

### Bước 4: Cấu hình biến môi trường

#### Tạo file `.env`

**Mac, Linux, WSL trên Windows:**

```bash
cp .env.example .env
```

**Windows Command Prompt:**

```bash
# Windows Command Prompt
copy .env.example .env

# Windows PowerShell
Copy-Item .env.example .env
```

#### Chỉnh sửa file `.env`

Mở `.env` trong text editor và cấu hình AI provider của bạn.

**Với GitHub Models (Miễn phí):**

```bash
AI_API_KEY=ghp_your_github_token_here
AI_ENDPOINT=https://models.inference.ai.azure.com
AI_MODEL=gpt-5-mini
```

**Thay `ghp_your_github_token_here` bằng token GitHub thực tế của bạn!**

**Lựa chọn khác: Microsoft Foundry:**

Nếu bạn có Azure subscription, bạn có thể dùng Microsoft Foundry cho ứng dụng AI cấp production. Xem [Phụ lục thiết lập Azure](./APPENDIX.md#microsoft-foundry-setup) để có hướng dẫn chi tiết về việc deploy model và cấu hình môi trường.

## Kiểm tra thiết lập của bạn

Hãy xác minh mọi thứ hoạt động!

### Chạy bài test

Chạy lệnh sau trong terminal, từ thư mục gốc của project:

```bash
python scripts/test_setup.py
```

**Kết quả mong đợi:**

```bash
🚀 Testing AI provider connection...

✅ SUCCESS! Your AI provider is working!
   Provider: https://models.inference.ai.azure.com
   Model: gpt-5-mini

Model response: Setup successful!

🎉 You're ready to start the course!
```

Nếu bạn thấy kết quả này, vậy là xong! Nếu không, xem phần xử lý sự cố bên dưới.

---

## ✅ Checklist thiết lập

Trước khi bắt đầu khoá học, đảm bảo bạn đã có:

Với Codespaces:

- [ ] Codespace và terminal đang mở
- [ ] File `.env` với các biến cần thiết
- [ ] Script test chạy thành công

Với Local development:

- [ ] Đã cài Python 3.10+
- [ ] Đã clone project và tạo virtual environment
- [ ] Đã cài dependencies (`pip install -r requirements.txt`)
- [ ] Đã tạo GitHub Personal Access Token nếu dùng GitHub Models. Nếu dùng Microsoft Foundry, đảm bảo model đã được deploy và bạn có API key + endpoint.
- [ ] File `.env` đã cấu hình token (hoặc key nếu dùng Microsoft Foundry) và endpoint
- [ ] Script test chạy thành công
- [ ] Đã cài VS Code (tuỳ chọn nhưng khuyến nghị)

---

## 🎯 Bước tiếp theo?

Xong rồi! Đến lúc xây dựng ứng dụng AI đầu tiên của bạn.

**👉 Tiếp tục với [Introduction to LangChain](../01-introduction/README.md)**

---

## 📚 Tài nguyên bổ sung

- [GitHub Models Documentation](https://github.com/marketplace/models)
- [Python Documentation](https://docs.python.org/3/)
- [LangChain Python Documentation](https://python.langchain.com/)

---

## 🗺️ Điều hướng

[Về trang chính](../README.md) | [Tiếp: Introduction to LangChain →](../01-introduction/README.md)

---

## 💬 Có thắc mắc?

Nếu gặp khó khăn hoặc có câu hỏi khi xây dựng ứng dụng AI, hãy tham gia:

[![Microsoft Foundry Discord](https://img.shields.io/badge/Discord-Azure_AI_Foundry_Community_Discord-blue?style=for-the-badge&logo=discord&color=5865f2&logoColor=fff)](https://aka.ms/foundry/discord)

Nếu bạn có góp ý sản phẩm hoặc gặp lỗi khi xây dựng, hãy truy cập:

[![Microsoft Foundry Developer Forum](https://img.shields.io/badge/GitHub-Azure_AI_Foundry_Developer_Forum-blue?style=for-the-badge&logo=github&color=000000&logoColor=fff)](https://aka.ms/foundry/forum)

Nếu bạn gặp vấn đề với tài liệu khoá học, hãy mở issue trong repo GitHub:

[![Course Issues](https://img.shields.io/badge/GitHub-LangChain_for_Beginners_Issues-blue?style=for-the-badge&logo=github&color=green&logoColor=fff)](https://github.com/microsoft/langchain-for-beginners/issues)
