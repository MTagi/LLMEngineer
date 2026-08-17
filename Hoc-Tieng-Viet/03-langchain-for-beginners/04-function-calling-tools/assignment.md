# Bài tập: Function Calling & Tooling

## Tổng quan

Luyện tập tạo tool type-safe với Pydantic schema, cài đặt pattern thực thi tool hoàn chỉnh, và xây dựng hệ thống đa tool mở rộng khả năng AI.

## Yêu cầu trước

- Đã hoàn thành [chương này](./README.md)
- Đã chạy tất cả ví dụ code trong chương
- Hiểu về tạo, bind, và thực thi tool
- Đã hoàn thành chương Prompts, Messages & Outputs

---

## Thử thách: Weather Tool với vòng lặp thực thi hoàn chỉnh ⛅

**Mục tiêu**: Xây dựng tool thời tiết và cài đặt pattern thực thi 3 bước hoàn chỉnh (sinh → thực thi → phản hồi).

**Nhiệm vụ**:
1. Tạo `weather_tool.py` trong thư mục `04-function-calling-tools/solution/`
2. Xây dựng tool thời tiết với Pydantic schema nhận:
   - `city` (string, bắt buộc) - Tên thành phố
   - `units` (Literal["celsius", "fahrenheit"], tuỳ chọn, mặc định: "fahrenheit") - Đơn vị nhiệt độ
3. Cài đặt tool trả về dữ liệu thời tiết mô phỏng cho ít nhất 5 thành phố
4. Cài đặt pattern thực thi 3 bước hoàn chỉnh:
   - **Bước 1**: Lấy tool call từ LLM
   - **Bước 2**: Thực thi tool
   - **Bước 3**: Gửi kết quả trở lại LLM cho response cuối cùng
5. Test với nhiều truy vấn dùng thành phố và đơn vị khác nhau

**Ví dụ truy vấn**:
- "What's the weather in Tokyo?"
- "Tell me the temperature in Paris in celsius"
- "Is it raining in London?"

**Tiêu chí thành công**:
- Tool dùng Pydantic schema đúng chuẩn với `Field(description=...)` cho tham số
- Xử lý được cả đơn vị celsius và fahrenheit
- Cài đặt đủ 3 bước thực thi tool
- LLM sinh response ngôn ngữ tự nhiên dựa trên kết quả tool
- Console output rõ ràng thể hiện từng bước

**Gợi ý**:
```python
# 1. Import required modules
import os
from typing import Literal, Optional
from dotenv import load_dotenv
from langchain_core.messages import AIMessage, HumanMessage, ToolMessage
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

# 2. Create input schema with Pydantic BaseModel
#    - city: str with Field(description="...")
#    - units: Optional[Literal["celsius", "fahrenheit"]] with default

# 3. Create a weather tool using the @tool decorator:
#    - Use args_schema parameter to specify the Pydantic model
#    - Implement function to return simulated weather data
#    - Add detailed docstring as the tool description

# 4. Bind the tool to the model using model.bind_tools()

# 5. Implement the 3-step execution pattern:
#    Step 1: Invoke model with user query, check for tool_calls
#    Step 2: Execute the tool with tool.invoke(tool_call["args"])
#    Step 3: Create messages list with HumanMessage, AIMessage, and ToolMessage
#            Then invoke model again for final natural language response
```

> [!TIP]
> **🤖 Nhờ [GitHub Copilot](../docs/copilot.md) giúp đỡ:** Nếu cần hỗ trợ với thử thách này, mở file này trong editor và [dùng Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) để được hỗ trợ và giải thích cá nhân hoá.

---

## Thử thách Bonus: Multi-Tool Travel Assistant 🌍

**Mục tiêu**: Xây dựng hệ thống với nhiều tool nơi LLM tự động chọn tool phù hợp cho truy vấn liên quan đến du lịch.

**Nhiệm vụ**:
1. Tạo `travel_assistant.py`
2. Xây dựng ba tool chuyên biệt:
   - **Currency Converter**: Chuyển đổi số tiền giữa các đơn vị tiền tệ (USD, EUR, GBP, JPY)
   - **Distance Calculator**: Tính khoảng cách giữa hai thành phố theo dặm hoặc km
   - **Time Zone Tool**: Lấy giờ hiện tại ở một thành phố và tính chênh lệch múi giờ
3. Mỗi tool nên có:
   - Tên rõ ràng, mang tính mô tả
   - Docstring chi tiết giải thích khi nào dùng nó
   - Pydantic schema đúng chuẩn với mô tả tham số
4. Bind cả ba tool vào model
5. Test với các truy vấn cần tool khác nhau:
   - "Convert 100 USD to EUR"
   - "What's the distance between New York and London?"
   - "What time is it in Tokyo right now?"
   - "If it's 3pm in Seattle, what time is it in Paris?"

**Tiêu chí thành công**:
- Cả ba tool hoạt động đúng
- LLM tự động chọn đúng tool cho mỗi truy vấn
- Mô tả tool đủ rõ ràng để hướng dẫn LLM chọn
- Trả về kết quả mô phỏng chính xác
- Xử lý edge case (mã tiền tệ không hợp lệ, thành phố không xác định)

**Gợi ý**:
```python
# 1. Import required modules
import os
from typing import Literal, Optional
from dotenv import load_dotenv
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

# 2. Create Pydantic input schemas for each tool

# 3. Create three tools using @tool decorator:
#    Currency Converter - with amount, from_currency, to_currency parameters
#    Distance Calculator - with from_city, to_city, and units parameters
#    Time Zone Tool - with city parameter
#    Make sure each has:
#    - Clear, descriptive docstring
#    - Proper Pydantic schema with Field(description="...") on parameters
#    - Simulated implementation returning appropriate data

# 4. Bind all three tools to the model with model.bind_tools([...])

# 5. Test with different queries and observe which tool the LLM selects

# 6. Create a tools_map dict to look up tool functions by name
#    Execute with: tool_fn.invoke(tool_call["args"])
```

**Tính năng bổ sung** (Tuỳ chọn):
Thêm xử lý lỗi trả về thông báo hữu ích khi:
- Mã tiền tệ không hợp lệ
- Tên thành phố không xác định
- Định dạng input không hợp lệ

**Ví dụ output**:
```
Query: "Convert 50 EUR to JPY"
→ LLM chose: currency_converter
→ Args: { "amount": 50, "from": "EUR", "to": "JPY" }
→ Result: "50 EUR equals approximately 8,100 JPY"

Query: "What's the distance from Paris to Rome?"
→ LLM chose: distance_calculator
→ Args: { "from": "Paris", "to": "Rome", "units": "kilometers" }
→ Result: "The distance from Paris to Rome is approximately 1,430 kilometers"
```

---

## Checklist nộp bài

Trước khi tiếp tục, đảm bảo bạn đã hoàn thành:

- [ ] Thử thách: Weather tool với đủ 3 bước thực thi
- [ ] Bonus: Multi-tool travel assistant (tuỳ chọn)

---

## Lời giải

Lời giải cho tất cả thử thách có sẵn trong thư mục [`solution/`](./solution/). Hãy thử tự hoàn thành thử thách trước!

---

## Cần trợ giúp?

- **Tạo tool**: Xem lại Ví dụ 1 trong [`code/01_simple_tool.py`](./code/01_simple_tool.py)
- **Pattern thực thi**: Xem Ví dụ 3 trong [`code/03_tool_execution.py`](./code/03_tool_execution.py)
- **Nhiều tool**: Xem Ví dụ 4 trong [`code/04_multiple_tools.py`](./code/04_multiple_tools.py)
- **Pydantic schema**: Xem lại [phần Pydantic](./README.md#🛠️-tạo-tool-với-decorator-tool) trong README
- **Câu hỏi bất kỳ**: Dùng [Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) trong GitHub Copilot
- **Vẫn còn vướng**: Tham gia [cộng đồng Discord](https://aka.ms/foundry/discord)

---

## Bước tiếp theo

Sau khi hoàn thành các thử thách này, bạn đã sẵn sàng cho:

**[Getting Started with Agents](../05-agents/README.md)**

Làm tốt lắm, bạn đã làm chủ function calling và tooling! 🚀
