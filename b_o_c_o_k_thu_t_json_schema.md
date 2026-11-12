## skill_name: query_store_sales_schema
description: Đặc tả Function Calling JSON Schema cho công cụ truy vấn cơ sở dữ liệu WinMart.
author: AI301 Student
version: 1.0.0
tags: [JSON Schema, Function Calling, Data Analytics, WinCommerce]

# Báo cáo Kỹ thuật: Định nghĩa Function Calling JSON Schema cho công cụ truy vấn cơ sở dữ liệu WinMart

## 1. Bối cảnh dự án
Với hệ thống bán lẻ quy mô lớn gồm hơn 3.600 siêu thị WinMart và WinMart+ thuộc Tập đoàn Masan, việc khai thác dữ liệu bán hàng tự động đòi hỏi độ chính xác tuyệt đối. Để Data Analytics Agent có thể tự động truy vấn dữ liệu từ hệ thống nội bộ, chúng ta cần định nghĩa một đặc tả công cụ (Tool Specification) chặt chẽ bằng JSON Schema, làm cầu nối để Mô hình ngôn ngữ lớn (LLM) hiểu và gọi đúng hàm nghiệp vụ `query_store_sales`.

## 2. Nhiệm vụ 1: Soạn thảo JSON Schema cho công cụ
Dưới đây là tài liệu đặc tả JSON Schema tiêu chuẩn để cấu hình cho LLM, bao gồm các ràng buộc chặt chẽ về mặt định dạng nhằm tránh lỗi truy vấn cơ sở dữ liệu:

```json
{
  "name": "query_store_sales",
  "description": "Truy vấn dữ liệu doanh số và thông tin tồn kho tại các siêu thị WinMart/WinMart+ dựa trên mã cửa hàng và thời gian bắt đầu.",
  "parameters": {
    "type": "object",
    "properties": {
      "store_id": {
        "type": "string",
        "description": "Mã định danh duy nhất của siêu thị (ví dụ: WMT_TIMES_CITY)."
      },
      "start_date": {
        "type": "string",
        "description": "Ngày bắt đầu truy vấn dữ liệu. Bắt buộc phải tuân thủ định dạng năm-tháng-ngày (YYYY-MM-DD).",
        "pattern": "^\\d{4}-\\d{2}-\\d{2}$"
      },
      "category_id": {
        "type": "string",
        "description": "Mã định danh của ngành hàng cần kiểm tra (ví dụ: FRESH_FOOD). Không bắt buộc."
      }
    },
    "required": ["store_id", "start_date"]
  }
}
```

*Phân tích thiết kế:* Thiết kế này đánh dấu rõ ràng `store_id` và `start_date` ở mảng `required`. Đặc biệt, trường `start_date` sử dụng khóa `pattern` (Biểu thức chính quy - Regex) để áp đặt quy tắc định dạng bắt buộc, ngăn chặn ảo giác (hallucination) của LLM về cách ghi ngày tháng.

## 3. Nhiệm vụ 2: Mô phỏng lời gọi hàm từ LLM
Khi người dùng đưa ra Prompt: *"Kiểm tra doanh thu ngành hàng Thực phẩm tươi sống tại siêu thị WinMart Times City từ ngày 01/10 đến 07/10/2024"*.

Mặc dù người dùng đề cập đến một khoảng thời gian, nhưng dựa vào định nghĩa schema chỉ chấp nhận `start_date`, Agent sẽ tự động suy luận để trích xuất ngày bắt đầu hợp lý nhất, đồng thời ánh xạ ngôn ngữ tự nhiên thành các mã định danh tương ứng. 

Đoạn mã JSON giả lập (Function Argument) mà LLM sẽ sinh ra để gọi hàm như sau:

```json
{
  "name": "query_store_sales",
  "arguments": {
    "store_id": "WMT_TIMES_CITY",
    "start_date": "2024-10-01",
    "category_id": "FRESH_FOOD"
  }
}
```

## 4. Nhiệm vụ 3: Đánh giá cơ chế xác thực tham số (Parameter Validation)
Dù LLM rất thông minh, vẫn luôn tồn tại rủi ro mô hình sinh ra định dạng ngày tháng theo thói quen của người dùng (ví dụ: `"start_date": "01/10/2024"`). Đây là lúc cơ chế xác thực tham số tại môi trường thực thi (Runtime) phát huy tác dụng.

*   **Bắt lỗi (Error Trapping):** Trước khi payload JSON được gửi xuống tầng Cơ sở dữ liệu (Database), trình xác thực (như Pydantic của Python) sẽ đối chiếu với JSON Schema. Chuỗi "01/10/2024" ngay lập tức vi phạm regex `^\d{4}-\d{2}-\d{2}$`.
*   **Chặn rủi ro:** Runtime sẽ đình chỉ việc thực thi hàm (ngăn ngừa lỗi sập hệ thống hoặc tốn tài nguyên truy vấn vô nghĩa).
*   **Cơ chế tự sửa chữa (Self-Correction):** Hệ thống sẽ trả lại một mã lỗi nguyên bản cho Agent, ví dụ: `ValidationError: start_date does not match YYYY-MM-DD`. Nhờ khả năng hiểu ngữ cảnh, Agent sẽ nhận ra lỗi của mình và tiến hành sinh lại một đoạn JSON mới đã được sửa đổi thành `"2024-10-01"` để gọi lại hàm (Retry) mà không cần sự can thiệp của con người.

**Kết luận:**
Việc thiết lập JSON Schema chặt chẽ là lớp áo giáp bảo vệ quan trọng nhất của kiến trúc Agent. Nó giúp Data Analytics Agent giao tiếp an toàn, chuẩn xác với kho dữ liệu đồ sộ của WinCommerce.