# Bài tập: Dùng Agent với Pattern ReAct

## Tổng quan

Luyện tập xây dựng AI agent tự trị dùng pattern ReAct, cài đặt agent loop lặp cho đến khi giải quyết vấn đề, và tạo hệ thống đa tool nơi agent quyết định dùng tool nào và khi nào.

## Yêu cầu trước

- Đã hoàn thành [chương này](./README.md)
- Đã chạy tất cả ví dụ code trong chương
- Hiểu pattern ReAct và agent loop
- Đã hoàn thành chương Function Calling & Tools

---

## Thử thách: Research Agent 🔍

**Mục tiêu**: Xây dựng agent dùng `create_agent()` trả lời câu hỏi cần tìm kiếm web và tính toán.

**Nhiệm vụ**:
1. Tạo `research_agent.py` trong thư mục `05-agents/solution/`
2. Tạo hai tool:
   - **Search Tool**: Mô phỏng tìm kiếm web (trả về kết quả định sẵn cho query phổ biến)
   - **Calculator Tool**: Thực hiện phép tính
3. Xây dựng agent bằng `create_agent()` với cả hai tool
4. Test với query cần nhiều bước
5. Hiển thị output rõ ràng cho thấy agent đã dùng tool nào

**Ví dụ query**:
- "What is the population of Tokyo multiplied by 2?"
  - Bước 1: Tìm kiếm dân số Tokyo
  - Bước 2: Tính dân số * 2
  - Bước 3: Đưa ra câu trả lời
- "Search for the capital of France and tell me how many letters are in its name"
  - Bước 1: Tìm kiếm thủ đô của Pháp
  - Bước 2: Tính số chữ cái trong "Paris"
  - Bước 3: Đưa ra câu trả lời

**Tiêu chí thành công**:
- Agent dùng `create_agent()` (cách tiếp cận LangChain khuyến nghị)
- Cả hai tool được định nghĩa đúng chuẩn với mô tả rõ ràng
- Agent tự trị quyết định tool nào cần dùng cho mỗi query
- Agent xử lý đúng query nhiều bước cần dùng cả hai tool
- Console output rõ ràng cho thấy tool nào được dùng
- Agent đưa ra câu trả lời cuối cùng chính xác

**Gợi ý**:
```python
# 1. Import required modules
import os
from dotenv import load_dotenv
from langchain.agents import create_agent
from langchain_core.messages import AIMessage, HumanMessage
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

# 2. Define your search tool using @tool decorator:
#    Sample search data to get you started:
#    {
#      "population of tokyo": "Tokyo has a population of approximately 14 million...",
#      "capital of france": "The capital of France is Paris.",
#      "capital of japan": "The capital of Japan is Tokyo.",
#      "population of new york": "New York City has a population of approximately 8.3 million.",
#      # Add more...
#    }
#
#    Implementation tips:
#    - Create a dict with search results
#    - Convert the query to lowercase
#    - Loop through entries and check if query includes key or key includes query
#    - Return matching result or "No results found"
#
#    Use Pydantic BaseModel for args_schema with:
#    - query: str = Field(description="The search query...")

# 3. Define your calculator tool using @tool decorator:
#    - Use Python's eval() with restricted builtins for safe expression evaluation
#    - Example: eval(expression, {"__builtins__": {}}, {"abs": abs, ...})
#    - Return result as a string
#    - Handle errors with try/except
#
#    Schema should have:
#    - expression: str = Field(description="The mathematical expression...")

# 4. Create the ChatOpenAI model with your environment variables

# 5. Create agent using create_agent():
#    agent = create_agent(model, tools=[search_tool, calculator_tool])

# 6. Test with multi-step queries in a loop:
#    queries = ["What is the population of Tokyo multiplied by 2?", ...]
#    for query in queries:
#        response = agent.invoke({"messages": [HumanMessage(content=query)]})
#        last_message = response["messages"][-1]
#        print(last_message.content)

# 7. Optional: Display which tools were used:
#    tool_calls = []
#    for msg in response["messages"]:
#        if isinstance(msg, AIMessage) and msg.tool_calls:
#            tool_calls.extend([tc["name"] for tc in msg.tool_calls])
#    print(f"Tools used: {', '.join(set(tool_calls))}")
```

**Hành vi mong đợi**:
- Query: "What is the population of Tokyo multiplied by 2?"
- Agent tự động:
  1. Dùng search tool để tìm dân số Tokyo (≈14 triệu)
  2. Dùng calculator tool để nhân 2
  3. Trả về "The population of Tokyo multiplied by 2 is 28 million."

**Gợi ý**:
- Làm theo pattern từ Ví dụ 1 và 2 trong chương
- Dùng create_agent() - nó tự động xử lý vòng lặp ReAct
- Tập trung tạo tool được mô tả tốt để agent biết khi nào dùng chúng
- Agent sẽ lặp qua các tool cho đến khi có đủ thông tin để trả lời

> [!TIP]
> **🤖 Nhờ [GitHub Copilot](../docs/copilot.md) giúp đỡ:** Nếu cần hỗ trợ với thử thách này, mở file này trong editor và [dùng Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) để được hỗ trợ và giải thích cá nhân hoá.

---

## Thử thách Bonus: Multi-Step Planning Agent 🎯

**Mục tiêu**: Xây dựng agent với nhiều tool chuyên biệt dùng `create_agent()`, cần suy luận nhiều bước để giải quyết query phức tạp.

**Nhiệm vụ**:
1. Tạo `planning_agent.py`
2. Tạo bốn tool chuyên biệt:
   - **Search Tool**: Tìm thông tin sự kiện
   - **Calculator Tool**: Thực hiện phép tính
   - **Unit Converter Tool**: Chuyển đổi giữa các đơn vị (dặm/km, USD/EUR, v.v.)
   - **Comparison Tool**: So sánh hai giá trị và xác định cái nào lớn hơn/nhỏ hơn
3. Tạo agent bằng `create_agent()` với cả bốn tool
4. Thêm console output hữu ích cho thấy:
   - Tool nào đã được dùng
   - Tóm tắt cuối cùng cho thấy tổng số lần gọi tool
5. Test với query phức tạp nhiều bước

**Ví dụ Query phức tạp**:
- "What's the distance between London and Paris in miles, and is that more or less than 500 miles?"
  - Bước 1: Tìm kiếm khoảng cách (nhận: ~343 km)
  - Bước 2: Chuyển km sang dặm (nhận: ~213 dặm)
  - Bước 3: So sánh với 500 dặm (nhận: ít hơn)
  - Bước 4: Trả lời với thông tin đầy đủ

- "Find the population of New York and Tokyo, calculate the difference, and tell me the result in millions"
  - Bước 1: Tìm dân số NY
  - Bước 2: Tìm dân số Tokyo
  - Bước 3: Tính hiệu số
  - Bước 4: Chuyển sang đơn vị triệu
  - Bước 5: Trả lời

**Tiêu chí thành công**:
- Cả bốn tool được định nghĩa đúng chuẩn với mô tả rõ ràng
- Agent dùng `create_agent()` để xử lý việc chọn đa tool
- Agent tự trị dùng nhiều tool theo trình tự
- Xử lý được query cần 3+ lần gọi tool
- Output rõ ràng cho thấy tool nào được dùng
- Tóm tắt hiển thị tổng số lần dùng tool

**Gợi ý**:
```python
# 1. Import required modules
import os
from typing import Literal
from dotenv import load_dotenv
from langchain.agents import create_agent
from langchain_core.messages import AIMessage, HumanMessage
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

# 2. Define your four specialized tools:

# Search Tool (reuse from Challenge 1)
# Calculator Tool (reuse from Challenge 1)

# Unit Converter Tool - sample conversion data:
#    conversions = {
#      "km": {"miles": {"rate": 0.621371, "unit": "miles"}},
#      "miles": {"km": {"rate": 1.60934, "unit": "kilometers"}},
#      "usd": {"eur": {"rate": 0.92, "unit": "EUR"}},
#      "eur": {"usd": {"rate": 1.09, "unit": "USD"}},
#    }
#
#    Schema needs: value (float), from_unit (str), to_unit (str)

# Comparison Tool - handle operations:
#    "less" -> check if value1 < value2
#    "greater" -> check if value1 > value2
#    "equal" -> check if value1 == value2
#    "difference" -> return abs(value1 - value2)
#
#    Schema needs: value1 (float), value2 (float), operation (Literal enum)

# 3. Create the ChatOpenAI model

# 4. Create agent using create_agent():
#    Pass model and all four tools

# 5. Test with complex queries in a loop and display results

# 6. Display which tools were used:
#    Filter messages for AIMessage with tool_calls
#    Extract tool names and show unique tools + total count
```

**Tính năng bổ sung** (Tuỳ chọn):
- Thêm console output chi tiết cho mỗi lần gọi tool
- Hiển thị tóm tắt tất cả tool đã dùng sau khi agent hoàn thành
- Theo dõi và hiển thị tổng thời gian thực thi
- Thêm xử lý lỗi cho trường hợp tool thất bại

**Ví dụ output**:
```
🤖 Planning Agent: Multi-Step Query

Query: "What's the distance from London to Paris in miles, and is that more or less than 500 miles?"

🤖 Agent: The distance from London to Paris is approximately 213 miles, which is less than 500 miles.

─────────────────────────────────────────────
📊 Agent Summary:
   • Tools used: search, unit_converter, comparison_tool
   • Total tool calls: 3
   • Query solved successfully!
```

**Lưu ý**: Agent xử lý vòng lặp ReAct nội bộ, vậy nên bạn sẽ không thấy từng lần lặp riêng lẻ trừ khi thêm logging tuỳ chỉnh.

---

## Checklist nộp bài

Trước khi tiếp tục, đảm bảo bạn đã hoàn thành:

- [ ] Thử thách: Research agent với vòng lặp ReAct
- [ ] Bonus: Multi-step planning agent (tuỳ chọn)

---

## Lời giải

Lời giải cho tất cả thử thách có sẵn trong thư mục [`solution/`](./solution/). Hãy thử tự hoàn thành thử thách trước!

---

## Cần trợ giúp?

- **create_agent() cơ bản**: Xem lại Ví dụ 1 trong [`code/01_create_agent_basic.py`](./code/01_create_agent_basic.py)
- **Agent đa tool**: Xem Ví dụ 2 trong [`code/02_create_agent_multi_tool.py`](./code/02_create_agent_multi_tool.py)
- **Pattern ReAct**: Đọc lại [phần ReAct](./README.md#🧠-pattern-react) trong README
- **Vòng lặp agent thủ công**: Xem thư mục [`samples/`](./samples/) để có cài đặt vòng lặp thủ công
- **Câu hỏi bất kỳ**: Dùng [Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) trong GitHub Copilot
- **Vẫn còn vướng**: Tham gia [cộng đồng Discord](https://aka.ms/foundry/discord)

---

## Bước tiếp theo

Giờ bạn đã có thể xây dựng agent tự trị, bạn đã sẵn sàng kết nối chúng với dịch vụ bên ngoài!

**[Model Context Protocol (MCP)](../06-mcp/README.md)**

Ở chương tiếp theo, bạn sẽ học cách dùng MCP để kết nối agent của bạn với nhà cung cấp tool bên ngoài mà không cần viết tích hợp tuỳ chỉnh. Sau đó, bạn sẽ thêm khả năng tìm kiếm và truy xuất tài liệu!

Làm tốt lắm! 🚀
