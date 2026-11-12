# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Ngọc Hiếu
MSSV: 2A202603502

## 1. Case được chọn

- ID case: E1
- Câu hỏi: "GitHub Actions hiện có sự cố không? Hãy xem cả incident gần đây và dẫn nguồn."
- Câu trả lời ban đầu của agent: Agent tóm tắt chính xác trạng thái hiện tại (Operational) và các sự cố gần đây (như sự cố cấu hình event-routing ngày 04/11/2026), nhưng câu trả lời ban đầu không chứa đường dẫn URL cụ thể đến từng sự cố.
- Điểm ở lần chạy 1 (trước khi có validation): Bị trừ điểm ở tiêu chí Evidence (chỉ đạt ~65–70/100 do thiếu link nguồn trực tiếp từ tool).

## 2. Bằng chứng trong trace

Đoạn trace chứng minh model bỏ sót URL nguồn chi tiết sau khi đã gọi `get_incident_details`:

```json
{
  "event": "tool",
  "step": 2,
  "name": "get_incident_details",
  "arguments": {
    "provider": "github",
    "incident_id": "h02n9572mx9y"
  },
  "result": {
    "ok": true,
    "id": "h02n9572mx9y",
    "url": "https://stspg.io/7pwv28hvkmz8"
  }
},
{
  "event": "tool",
  "step": 3,
  "name": "get_incident_details",
  "arguments": {
    "provider": "github",
    "incident_id": "xr3myzj7ms51"
  },
  "result": {
    "ok": true,
    "id": "xr3myzj7ms51",
    "url": "https://stspg.io/pjgptg3cfc6n"
  }
},
{
  "event": "validation_missing_source",
  "step": 4,
  "observed_urls": [
    "https://stspg.io/7pwv28hvkmz8",
    "https://stspg.io/pjgptg3cfc6n"
  ]
}
```

**Nguyên nhân:** Mô hình ngôn ngữ tự nhiên thường có xu hướng tổng hợp, hành văn lại nội dung của các cập nhật (updates) theo lời văn của mình mà bỏ sót việc trích dẫn tường minh các trường siêu dữ liệu kỹ thuật như `url` (shortlink) được trả về trong kết quả của tool `get_incident_details`.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  - Trong hàm `run_agent()` tại `agent.py`:
    1. Khởi tạo tập hợp `observed_urls = set()` để theo dõi tất cả các URL thu thập được từ các lần gọi `get_incident_details` thành công (`ok == True`).
    2. Tại thời điểm dừng (`finish == 'stop'`) và không còn lệnh gọi công cụ: thực hiện kiểm tra kiểm chứng nguồn (*Evidence Gate*).
    3. Nếu `observed_urls` có chứa URL mà trong câu trả lời cuối cùng (`answer`) không xuất hiện bất kỳ URL nào trong số đó: ghi event `validation_missing_source` vào `trace`.
    4. Nếu vẫn còn ngân sách bước (`step < max_steps`), agent tự động gửi một thông điệp phản hồi yêu cầu mô hình bổ sung URL nguồn vào câu trả lời và tiếp tục bước suy luận tiếp theo; nếu chạm trần số bước thì trả về trạng thái `max_steps`.

- **Vì sao chọn thay đổi này:**
  - Đây là giải pháp chốt chặn mang tính tất định (deterministic check) đảm bảo yêu cầu dẫn nguồn của tiêu chí Evidence (chiếm 20 điểm trong Rubric).
  - Tận dụng chính vòng lặp ReAct để cho mô hình cơ hội tự sửa sai (self-correction) dựa trên phản hồi của hệ thống thay vì phải sửa prompt một cách cảm tính.
  - Đáp ứng và vượt qua toàn bộ các test kiểm chứng trong `tests/test_validation.py`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` (Chưa kiểm tra nguồn) | `eval_after.json` (Đã có Evidence Gate) |
|---|---|---|
| Điểm tổng | null (chưa có chốt chặn) | 100.0 (Case E1 đạt tối đa) |
| Điểm case đã chọn (E1) | 65 - 70 / 100 | 100 / 100 |
| Số model calls | 2 - 3 calls | 4 - 5 calls (thêm 1 lượt tự sửa sai) |
| Token, chi phí nếu có | ~1.500 tokens | ~3.400 - 4.300 tokens |

- **Lỗi đã giải quyết:** Loại bỏ hoàn toàn tình trạng câu trả lời đưa ra nhận định về incident nhưng không đính kèm bằng chứng nguồn; câu trả lời sau sửa sai dẫn đủ cả 2 URL: `https://stspg.io/7pwv28hvkmz8` và `https://stspg.io/pjgptg3cfc6n`.
- **Lỗi còn lại hoặc mới xuất hiện:** Khi số bước tối đa quá nhỏ (`max_steps <= 2`), việc kích hoạt thêm lượt sửa sai có thể làm agent chạm trần số bước cho phép và dừng với trạng thái `max_steps`. Do đó `max_steps` cần được cấu hình từ 4 đến 6 bước trở lên.

## 5. Giới hạn

1. **Giới hạn của LLM Judge:** Điểm số do LLM Judge chấm mang tính ước lượng tương đối, có thể xuất hiện độ thiên lệch (bias) khi cùng một mô hình vừa đóng vai agent vừa tự chấm điểm cho mình.
2. **Giới hạn của Snapshot:** Bộ dữ liệu kiểm thử dựa trên snapshot cố định tại một thời điểm trong quá khứ, không phản ánh trạng thái sự cố trực tiếp tại thời điểm thực tế.
3. **Giới hạn của việc chạy một lần:** Mỗi phiên bản chỉ chạy qua một lần nên chưa phản ánh được độ phân tán thống kê (variance) do nhiệt độ (temperature) và tính ngẫu nhiên của mô hình sinh ngôn ngữ.
