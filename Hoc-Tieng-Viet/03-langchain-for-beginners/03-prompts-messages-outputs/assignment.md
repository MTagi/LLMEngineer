# Bài tập: Prompts, Messages, and Structured Outputs

## Tổng quan

Luyện tập tạo prompt tái sử dụng, dễ bảo trì bằng template, few-shot learning, và kỹ thuật structured output. Bài tập này tập trung vào **template và structured output** (Phần 2 và 3 của chương).

## Yêu cầu trước

- Đã hoàn thành [chương này](./README.md)
- Đã chạy tất cả ví dụ code (bao gồm ví dụ structured output) trong chương
- Hiểu cú pháp và cách kết hợp template

---

## Thử thách: Few-Shot Format Teacher 🎓

**Mục tiêu**: Dùng few-shot prompting để dạy AI một định dạng output tuỳ chỉnh.

**Nhiệm vụ**:
1. Tạo `format_teacher.py`
2. Dạy AI chuyển mô tả sản phẩm thành định dạng JSON cụ thể:
   ```json
   {
     "name": "Product name",
     "price": "$XX.XX",
     "category": "Category",
     "highlight": "Key feature"
   }
   ```
3. Cung cấp 3-4 ví dụ chuyển đổi
4. Test với mô tả sản phẩm mới
5. Parse và validate output JSON

**Ví dụ để dạy** (cung cấp như few-shot example):
- Input: "Premium wireless headphones with noise cancellation, $199"
- Input: "Organic cotton t-shirt in blue, comfortable fit, $29.99"
- Input: "Gaming laptop with RTX 4070, 32GB RAM, $1,499"

**Tiêu chí thành công**:
- AI luôn cho ra JSON hợp lệ
- Định dạng khớp với ví dụ của bạn
- Hoạt động với nhiều mô tả sản phẩm khác nhau

**Gợi ý**:
```python
# 1. Import required modules
import json
import os
from dotenv import load_dotenv
from langchain_core.prompts import ChatPromptTemplate, FewShotChatMessagePromptTemplate
from langchain_openai import ChatOpenAI

# 2. Load environment variables and create model with temperature 0

# 3. Define your teaching examples list with input/output pairs
#    - Each example should show a product description as input
#    - And the corresponding JSON format as output (use json.dumps for formatting)

# 4. Create an example template using ChatPromptTemplate.from_messages
#    with ("human", "{input}") and ("ai", "{output}")

# 5. Create a FewShotChatMessagePromptTemplate with your examples

# 6. Build a final prompt that includes the few-shot template

# 7. Test with new product descriptions and parse the JSON output with json.loads()
```

> [!TIP]
> **🤖 Nhờ [GitHub Copilot](../docs/copilot.md) giúp đỡ:** Nếu cần hỗ trợ với thử thách này, mở file này trong editor và [dùng Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) để được hỗ trợ và giải thích cá nhân hoá.

---

## Thử thách Bonus: Product Data Extractor với Structured Output 🏷️

**Mục tiêu**: Xây dựng hệ thống trích xuất thông tin sản phẩm thành cấu trúc dữ liệu đã validate, có kiểu.

**Nhiệm vụ**:
1. Tạo `product_extractor.py`
2. Định nghĩa Pydantic model cho thông tin sản phẩm:
   ```python
   class Product(BaseModel):
       name: str
       price: float
       category: Literal["Electronics", "Clothing", "Food", "Books", "Home"]
       in_stock: bool
       rating: float  # 1-5
       features: list[str]
   ```
3. Dùng `with_structured_output()` để trích xuất dữ liệu sản phẩm
4. Test với mô tả sản phẩm ở nhiều định dạng khác nhau:
   - Danh sách sản phẩm trang trọng
   - Mô tả marketplace không trang trọng
   - Nội dung hỗn hợp (review + thông số kỹ thuật)
5. Validate rằng tất cả output khớp schema
6. Xử lý edge case (thiếu thông tin)

**Ví dụ Input**:
- "MacBook Pro 16-inch with M3 chip, $2,499. Currently in stock. Users rate it 4.8/5. Features: Liquid Retina display, 18-hour battery, 1TB SSD"
- "Cozy wool sweater, blue color, medium size. $89, available now! Customers love it - 4.5 stars. Hand-washable, made in Ireland"
- "The Great Gatsby by F. Scott Fitzgerald. Classic novel, paperback edition for $12.99. In stock. Rated 4.9 stars. 180 pages, published 1925"

**Tiêu chí thành công**:
- Tất cả output đúng kiểu
- Schema validation hoạt động chính xác
- Xử lý nhiều định dạng input khác nhau
- Phân loại sản phẩm chính xác
- Xử lý khéo léo dữ liệu bị thiếu

**Gợi ý**:
```python
# 1. Import required modules
import os
from typing import Literal
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

# 2. Load environment variables and create the ChatOpenAI model

# 3. Define a Pydantic model with all required fields:
#    - name: str with Field(description="...")
#    - price: float
#    - category: Literal["Electronics", "Clothing", "Food", "Books", "Home"]
#    - in_stock: bool
#    - rating: float with Field(ge=1, le=5)
#    - features: list[str]
#    Use Field(description="...") to add descriptions for each field

# 4. Create a structured output model using model.with_structured_output(Product)

# 5. Create a prompt template asking to extract product information

# 6. Create a chain by piping template | structured_model

# 7. Test with various product descriptions and handle edge cases
#    Access fields using result.name, result.price, etc.
#    Use result.model_dump_json(indent=2) for formatted JSON output
```

---

## Checklist nộp bài

Trước khi tiếp tục, đảm bảo bạn đã hoàn thành:

- [ ] Thử thách: Few-shot format teacher với output JSON
- [ ] Bonus: Product data extractor với structured output (tuỳ chọn)

---

## Lời giải

Lời giải cho tất cả thử thách có sẵn trong thư mục [`solution/`](./solution/). Hãy thử tự hoàn thành thử thách trước!

**Ví dụ bổ sung**: Xem thư mục [`samples/`](./samples/) để có thêm ví dụ bao gồm sinh email, hệ thống dịch thuật, prompt builder động, và thư viện template!

> **💡 Lưu ý**: Bài tập này tập trung vào template (Phần 2 của chương). Với pattern dựa trên message (Phần 1), bạn sẽ luyện tập xây dựng agent sau trong khoá học!

---

## Cần trợ giúp?

- **Cú pháp template**: Xem lại ví dụ trong [`code/`](./code/)
- **Vấn đề với few-shot**: Xem Ví dụ 5 ([`code/05_few_shot.py`](./code/05_few_shot.py))
- **Composition**: Xem lại Ví dụ 6 ([`code/06_composition.py`](./code/06_composition.py))
- **Câu hỏi bất kỳ**: Dùng [Challenge Tutor agent](../docs/copilot.md#challenge-tutor-agent) trong GitHub Copilot
- **Vẫn còn vướng**: Tham gia [cộng đồng Discord](https://aka.ms/foundry/discord)

---

## Bước tiếp theo

Sau khi hoàn thành các thử thách này, bạn đã sẵn sàng cho:

**[Function Calling & Tools](../04-function-calling-tools/README.md)**

Tiến bộ tuyệt vời! Bạn đang làm chủ prompt engineering! 🚀
