# Bài tập: Chat Models & Basic Interactions

## Tổng quan

Luyện tập hội thoại nhiều lượt, streaming, tham số, và xử lý lỗi để xây dựng ứng dụng AI vững chắc.

## Yêu cầu trước

- Đã hoàn thành [chương này](./README.md)
- Đã chạy tất cả ví dụ code trong chương
- Hiểu về quản lý lịch sử hội thoại

---

## Thử thách: Chatbot tương tác 🤖

**Mục tiêu**: Xây dựng chatbot duy trì lịch sử hội thoại qua nhiều lượt trao đổi.

**Nhiệm vụ**:
1. Tạo `chatbot.py` trong thư mục `02-chat-models/code/`
2. Cài đặt chatbot tương tác:
   - Nhận input của user trong vòng lặp
   - Duy trì lịch sử hội thoại
   - Cho phép user gõ "quit" để thoát
   - Hiển thị độ dài lịch sử hội thoại sau mỗi lượt trao đổi
3. Dùng SystemMessage để cho bot một tính cách (tuỳ bạn chọn!)

**Ví dụ tương tác**:
```
🤖 Chatbot: Hello! I'm your helpful assistant. Ask me anything!

You: What is Python?
🤖: Python is a versatile programming language...

You: Can you show me an example?
🤖: Sure! Here's a simple Python example...

You: quit
👋 Goodbye! We had 5 messages in our conversation.
```

**Tiêu chí thành công**:
- Bot nhớ message trước đó
- Lịch sử hội thoại được duy trì đúng
- User có thể thoát một cách trơn tru

**Gợi ý**:
```python
# 1. Import required modules
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from dotenv import load_dotenv
import os

# 2. Load environment variables
load_dotenv()

# 3. Create the ChatOpenAI model
model = ChatOpenAI(model=os.environ.get("AI_MODEL", "gpt-5-mini"))

# 4. Initialize conversation history list with a SystemMessage for personality
messages = [
    SystemMessage(content="You are a helpful assistant.")
]

# 5. Create a loop that:
#    - Prompts for user input using input()
#    - Adds HumanMessage to messages list
#    - Invokes model with messages list
#    - Adds AIMessage to messages list
#    - Displays the response

# 6. Check for "quit" to exit the loop

# 7. Show conversation history length on exit
```

> [!TIP]
> **🤖 Nhờ [GitHub Copilot](../docs/copilot.md) giúp đỡ:** Nếu cần hỗ trợ với thử thách này, mở file này trong editor và [dùng Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) để được hỗ trợ và giải thích cá nhân hoá.

---

## Thử thách Bonus: Thử nghiệm Temperature 🌡️

**Mục tiêu**: Hiểu cách temperature ảnh hưởng đến độ sáng tạo và tính nhất quán của AI.

**Nhiệm vụ**:
1. Tạo `temperature_lab.py`
2. Test cùng một prompt sáng tạo với 5 giá trị temperature khác nhau: 0, 0.5, 1, 1.5, 2
3. Chạy mỗi temperature 3 lần để thấy sự biến thiên
4. Hiển thị kết quả theo định dạng dễ đọc
5. Thêm phân tích của bạn về temperature nào phù hợp nhất cho từng use case

**Gợi ý Prompt sáng tạo**:
- "Write a tagline for a coffee shop"
- "Create a name for a tech startup"
- "Suggest a title for a mystery novel"

**Kết quả mong đợi**:
```
🌡️ Temperature: 0
─────────────────────────────────────
Try 1: "Brew Your Best Day"
Try 2: "Brew Your Best Day"
Try 3: "Brew Your Best Day"

🌡️ Temperature: 2
─────────────────────────────────────
Try 1: "Caffeinated Dreams Await"
Try 2: "Sip the Extraordinary"
Try 3: "Where Magic Meets Mocha"

📊 Analysis:
- Temperature 0: Perfect for factual, consistent responses
- Temperature 2: Great for creative brainstorming
```

**Tiêu chí thành công**:
- Test ít nhất 5 giá trị temperature
- Thể hiện rõ sự biến thiên
- Bao gồm phân tích của bạn về kết quả

**Gợi ý**:
```python
# 1. Import required modules
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
import os

# 2. Load environment variables
load_dotenv()

# 3. Define a list of temperature values to test [0, 0.5, 1, 1.5, 2]

# 4. Define your creative prompt

# 5. Loop through each temperature value:
#    - Create a NEW model instance with that temperature
#    - Run 3 trials with the same prompt
#    - Display the results for each trial

# 6. Add your analysis comparing the different temperature results
```

---

## Checklist nộp bài

Trước khi tiếp tục, đảm bảo bạn đã hoàn thành:

- [ ] Thử thách: Chatbot tương tác có lịch sử
- [ ] Bonus: Thử nghiệm so sánh temperature (tuỳ chọn)

---

## Lời giải

Lời giải cho tất cả thử thách có sẵn trong thư mục [`solution/`](./solution/). Hãy thử tự hoàn thành thử thách trước!

**Ví dụ bổ sung**: Xem thư mục [`samples/`](./samples/) để có thêm ví dụ minh hoạ về streaming, xử lý lỗi, và theo dõi token!

---

## Cần trợ giúp?

- **Vướng ở code**: Xem lại ví dụ trong [`code/`](./code/)
- **Lỗi**: Xem phần xử lý lỗi của chương
- **Khái niệm**: Đọc và học lại [Chương này](./README.md)
- **Câu hỏi bất kỳ**: Dùng [Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) trong GitHub Copilot
- **Vẫn còn vướng**: Tham gia [cộng đồng Discord](https://aka.ms/foundry/discord)

---

## Bước tiếp theo

Sau khi hoàn thành các thử thách này, bạn đã sẵn sàng cho:

**[Prompts, Messages, and Structured Outputs](../03-prompts-messages-outputs/README.md)**

Bạn đang tiến bộ rất tốt! 🚀
