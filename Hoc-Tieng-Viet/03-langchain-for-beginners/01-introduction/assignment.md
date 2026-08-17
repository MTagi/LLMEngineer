# Bài tập: Introduction to LangChain

## Tổng quan

Giờ bạn đã học các kiến thức cơ bản của LangChain, đến lúc luyện tập! Các thử thách này sẽ giúp củng cố những gì bạn đã học về model, message, và thực hiện lệnh gọi LLM đầu tiên.

## Yêu cầu trước

- Đã hoàn thành [Course Setup](../00-course-setup/README.md)
- Đã đọc và học [chương này](./README.md)
- Đã chạy tất cả ví dụ code trong lesson này

---

## Thử thách: Thử nghiệm với System Prompt 🎭

**Mục tiêu**: Học cách system message ảnh hưởng đến hành vi AI.

**Nhiệm vụ**:
1. Tạo file tên `personality_test.py`
2. Test cùng một câu hỏi với ba system prompt khác nhau:
   - Tính cách cướp biển
   - Chuyên viên phân tích kinh doanh chuyên nghiệp
   - Giáo viên thân thiện dạy trẻ em
3. Hiển thị cả ba response song song

**Ví dụ System Prompt**:
- Cướp biển: `"You are a pirate. Answer all questions in pirate speak with 'Arrr!' and nautical terms."`
- Chuyên viên phân tích: `"You are a professional business analyst. Give precise, data-driven answers."`
- Giáo viên: `"You are a friendly teacher explaining concepts to 8-year-old children."`

**Câu hỏi để test**: "What is artificial intelligence?"

**Tiêu chí thành công**:
- Cùng một câu hỏi cho ra ba phong cách response rất khác nhau
- Bạn hiểu cách SystemMessage định hình tính cách của AI

**Gợi ý**:
```python
# 1. Import required modules
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage
from dotenv import load_dotenv

# 2. Load environment variables
load_dotenv()

# 3. Create the ChatOpenAI model (reuse for all personalities)

# 4. Define a list of personalities with name and system prompt

# 5. Define the question to test

# 6. Loop through each personality:
#    - Create messages list with SystemMessage and HumanMessage
#    - Invoke the model with the messages
#    - Display the response with personality name
```

> [!TIP]
> **🤖 Nhờ [GitHub Copilot](../docs/copilot.md) giúp đỡ:** Nếu cần hỗ trợ với thử thách này, mở file này trong editor và [dùng Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) để được hỗ trợ và giải thích cá nhân hoá.

---

## Thử thách Bonus: So sánh hiệu năng Model 🔬

**Mục tiêu**: So sánh nhiều model trên cùng một task.

**Nhiệm vụ**:
1. Tạo file tên `model_performance.py`
2. Test ít nhất 2 model có trên GitHub Models:
   - `gpt-5`
   - `gpt-5-mini`
3. Với mỗi model, đo:
   - Thời gian response
   - Độ dài response (số ký tự)
   - Chất lượng response (đánh giá chủ quan của bạn)
4. Tạo một bảng đơn giản hiển thị kết quả

**Câu hỏi test**: "Explain the difference between machine learning and deep learning."

**Kết quả mong đợi**:
```
📊 Model Performance Comparison
─────────────────────────────────────────────
Model          | Time    | Length | Quality
─────────────────────────────────────────────
gpt-5-mini    | 567ms   | 234ch  | ⭐⭐⭐⭐
gpt-5         | 1234ms  | 456ch  | ⭐⭐⭐⭐⭐
```

**Tiêu chí thành công**:
- Script so sánh ít nhất 2 model
- Kết quả hiển thị rõ ràng, dễ đọc
- Bạn có thể giải thích sẽ chọn model nào cho các use case khác nhau

**Gợi ý**:
```python
# 1. Import required modules
import time
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv

# 2. Load environment variables
load_dotenv()

# 3. Define question and models list
question = "Explain the difference between machine learning and deep learning."

models = [
    {"name": "gpt-5", "description": "Most capable"},
    {"name": "gpt-5-mini", "description": "Fast and efficient"},
]

# 4. Create a function to test each model:
#    - Accept model_name as parameter
#    - Create ChatOpenAI instance with that model
#    - Measure start time with time.time()
#    - Invoke the model with the question
#    - Measure end time and calculate duration
#    - Return a dict with name, time, length, and response

# 5. Loop through models list:
#    - Call test_model() for each model
#    - Display results in a formatted table
#    - Use .ljust() for consistent column widths
```

---

## Checklist nộp bài

Trước khi tiếp tục, đảm bảo bạn đã hoàn thành:

- [ ] Thử thách: Thử nghiệm system prompt cho thấy sự khác biệt về tính cách
- [ ] Bonus: So sánh model hiển thị kết quả (tuỳ chọn)

---

## Lời giải

Lời giải cho tất cả thử thách có sẵn trong thư mục [`solution/`](./solution/). Hãy thử tự hoàn thành thử thách trước khi xem lời giải!

**Ví dụ bổ sung**: Xem thư mục [`samples/`](./samples/) để có thêm ví dụ minh hoạ các khái niệm hữu ích khác!

---

## Cần trợ giúp?

- **Vướng ở code**: Xem lại ví dụ trong [`code/`](./code/)
- **Thông báo lỗi**: Xem phần xử lý sự cố ở [Course Setup](../00-course-setup/README.md)
- **Khái niệm chưa rõ**: Đọc và học lại [Chương này](./README.md)
- **Câu hỏi bất kỳ**: Dùng [Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) trong GitHub Copilot
- **Vẫn còn vướng**: Tham gia [cộng đồng Discord](https://aka.ms/foundry/discord)

---

## Bước tiếp theo

Sau khi hoàn thành các thử thách này, bạn đã sẵn sàng cho:

**[Chương 2: Chat Models & Basic Interactions →](../02-chat-models/README.md)**

Làm tốt lắm! Bạn đã có những bước đầu tiên với LangChain! 🎉
