# Bài tập: Model Context Protocol (MCP)

## Tổng quan

Trong bài tập này, bạn sẽ luyện tập kết nối AI agent với dịch vụ bên ngoài bằng Model Context Protocol (MCP). Bạn sẽ làm việc với MCP server, tích hợp tool của chúng với agent, và xây dựng ứng dụng đa dịch vụ.

---

## Thử thách 1: Kết nối đến Context7 MCP Server

**Mục tiêu**: Thiết lập và dùng MCP server Context7 để lấy tài liệu thư viện.

**Yêu cầu**:
1. Kết nối đến server Context7 công khai tại `https://mcp.context7.com/mcp`
2. Liệt kê tất cả tool khả dụng từ server
3. Tạo agent dùng tool Context7
4. Truy vấn tài liệu về một thư viện JavaScript bạn chọn (vd. "How do I use Express.js middleware?")
5. Hiển thị response với định dạng phù hợp

**Kết quả mong đợi**:
```
🔧 Available Tools from Context7:
   • resolve-library-id: Convert library name to Context7 ID
   • get-library-docs: Retrieve library documentation

👤 User: How do I use Express.js middleware?
🤖 Agent: [Detailed documentation about Express.js middleware from Context7]
```

**Gợi ý**:
- Dùng `MultiServerMCPClient` từ `langchain_mcp_adapters.client`
- Dùng transport `streamable_http` cho Context7
- Nhớ dùng `async/await` vì thao tác MCP là bất đồng bộ

> [!TIP]
> **🤖 Nhờ [GitHub Copilot](../docs/copilot.md) giúp đỡ:** Nếu cần hỗ trợ với thử thách này, mở file này trong editor và [dùng Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) để được hỗ trợ và giải thích cá nhân hoá.

---

## Thử thách 2: Xây dựng Multi-Tool Agent với MCP

**Mục tiêu**: Kết hợp MCP tool với tool tự tạo thủ công trong một agent.

**Yêu cầu**:
1. Kết nối đến Context7 MCP server (cho tài liệu)
2. Tự tạo tool calculator tuỳ chỉnh thủ công (giống như bạn đã làm với agent)
3. Tạo agent có quyền truy cập cả MCP tool VÀ tool tuỳ chỉnh của bạn
4. Test với query cần tool khác nhau:
   - "What is 125 * 8?" (nên dùng calculator)
   - "How do I use React hooks?" (nên dùng Context7)
   - "Calculate 50 + 25, then look up documentation for the result" (nên dùng cả hai)

**Kết quả mong đợi**:
```
🎛️  Multi-Tool Agent (MCP + Custom Tools)

👤 User: What is 125 * 8?
🤖 Agent: 125 × 8 = 1000

👤 User: How do I use React hooks?
🤖 Agent: [Documentation from Context7 about React hooks]

👤 User: Calculate 50 + 25, then look up docs for that number
🤖 Agent: 50 + 25 = 75. [Searches for "75" in documentation if relevant]
```

**Gợi ý**:
- Kết hợp tool: `all_tools = [*mcp_tools, calculator_tool]`
- Agent sẽ tự động chọn tool đúng dựa trên query
- Mô tả tool rõ ràng giúp agent chọn tốt hơn

---

## Thử thách 3 (Bonus): Tích hợp đa Server

**Mục tiêu**: Kết nối đến nhiều MCP server cùng lúc và dùng tool từ tất cả chúng.

**Yêu cầu**:
1. Kết nối đến Context7 cho tài liệu
2. Kết nối đến một MCP server khác bạn chọn (xem MCP Registry: https://github.com/mcp)
3. Tạo agent có thể dùng tool từ cả hai server
4. Minh hoạ agent dùng tool từ các server khác nhau trong cùng một hội thoại

**Ví dụ cấu hình**:
```python
client = MultiServerMCPClient(
    {
        "context7": {
            "transport": "streamable_http",
            "url": "https://mcp.context7.com/mcp"
        },
        # Add another server here
        "myServer": {
            "transport": "streamable_http",
            "url": "https://your-server-url.com/mcp"
        }
    }
)
```

**Kết quả mong đợi**:
```
🌐 Multi-Server MCP Agent

📋 Available Tools:
   From context7:
   • resolve-library-id
   • get-library-docs

   From myServer:
   • [list of tools from your second server]

👤 User: [Query that uses tools from different servers]
🤖 Agent: [Coordinated response using multiple servers]
```

**Gợi ý**:
- Tất cả tool từ tất cả server đều khả dụng với agent
- Agent chọn tool dựa trên mô tả, bất kể server nào cung cấp
- Bạn có thể tìm MCP server khả dụng tại [MCP Registry](https://github.com/mcp)

---

## 🎯 Mục tiêu học tập đã bao quát

Sau khi hoàn thành các thử thách này, bạn sẽ đã:
- ✅ Kết nối đến MCP server bên ngoài bằng Streamable HTTP transport
- ✅ Tích hợp MCP tool với LangChain agent
- ✅ Kết hợp MCP tool với tool tự tạo thủ công
- ✅ Làm việc với nhiều MCP server cùng lúc
- ✅ Xây dựng tích hợp MCP sẵn sàng cho production

---

## 📂 Lời giải

Xem [`solution/`](./solution/) để có cài đặt tham khảo cho tất cả thử thách.

**Lưu ý**: Hãy thử tự giải các thử thách trước khi xem lời giải!

---

## Cần trợ giúp?

- **Kiến thức nền tảng về agent**: Xem lại [Getting Started with Agents](../05-agents/README.md)
- **Câu hỏi bất kỳ**: Dùng [Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) trong GitHub Copilot
- **Vẫn còn vướng**: Tham gia [cộng đồng Discord](https://aka.ms/foundry/discord)

---

## 💡 Mẹo để thành công

1. **Bắt đầu đơn giản**: Bắt đầu với Thử thách 1 để hiểu kiến thức cơ bản
2. **Mô tả rõ ràng**: Mô tả tool tốt giúp agent chọn đúng
3. **Xử lý lỗi**: Luôn bọc lệnh gọi MCP trong khối try-except
4. **Async/Await**: Nhớ rằng thao tác MCP là bất đồng bộ
5. **Test từng phần**: Test từng tool riêng lẻ trước khi kết hợp chúng

---

## 🐛 Vấn đề thường gặp

### "Failed to connect to MCP server"
- Kiểm tra kết nối internet
- Xác nhận URL server đúng
- Đảm bảo server đang chạy (với server local)

### "Tool not found"
- Xác nhận tool đã được lấy: `print(await client.get_tools())`
- Kiểm tra tên tool khớp với những gì server cung cấp
- Đảm bảo MCP client được khởi tạo đúng

### "Agent doesn't use the right tool"
- Cải thiện mô tả tool để cụ thể hơn
- Kiểm tra tool được bind đúng cho agent
- Xác nhận query thể hiện rõ tool nào cần dùng

---

## 🚀 Đi xa hơn

Sau khi hoàn thành các thử thách này, hãy thử:
- Xây dựng MCP server của riêng bạn (xem [MCP for Beginners](https://github.com/microsoft/mcp-for-beginners))
- Kết nối đến dịch vụ MCP production (GitHub, Slack, database)
- Cài đặt xác thực cho MCP server
- Dùng stdio transport để phát triển local

---

[← Về Model Context Protocol (MCP)](./README.md)
