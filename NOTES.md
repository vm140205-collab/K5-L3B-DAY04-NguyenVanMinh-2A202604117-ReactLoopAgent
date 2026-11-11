# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Văn Minh
MSSV: 2A202604117

Cấu hình hai lần chạy (giữ nguyên): provider `mistral`, model `mistral-large-latest` cho cả agent và judge, snapshot, `max_steps=8`, bộ 22 case core.

## 1. Case được chọn

- ID case: **H1** — "Điều tra incident GitHub h02n9572mx9y. Theo liên kết do incident cung cấp để tìm nguyên nhân và ảnh hưởng, rồi đối chiếu với incident Cloudflare yaa0w..."
- Câu trả lời của agent (lần 1): báo cáo đúng trạng thái/thời gian của hai incident nhưng chỉ gọi 2 tool call trong **cùng một model step** và **không theo liên kết** tới incident gộp `xr3myzj7ms51`.
- Điểm ở lần chạy 1: **60/100** (bị trần cơ học `min_tool_steps = 2`).

## 2. Bằng chứng trong trace

Trace lần 1 (`eval_before.json`, case H1) — cả hai call ở step 1:

```text
{'event': 'tool', 'step': 1, 'name': 'get_incident_details', 'arguments': '{"provider": "github", "incident_id": "h02n9572mx9y"}', ...}
{'event': 'tool', 'step': 1, 'name': 'get_incident_details', 'arguments': '{"provider": "cloudflare", "incident_id": "yaa0witbl2me"}', ...}
mechanical.violations: ['Cần tool calls ở ít nhất 2 model steps khác nhau.'] -> score_cap 60
```

Nguyên nhân có thể xảy ra: model gộp mọi tra cứu vào một lượt parallel thay vì điều tra tuần tự; kết quả `h02n9572mx9y` nhắc tới incident gộp `xr3myzj7ms51` nhưng agent không gọi tiếp ở lượt sau, nên vừa vi phạm `min_tool_steps` vừa bỏ sót bằng chứng về nguyên nhân.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: bổ sung 2 câu vào `SYSTEM_PROMPT` trong `agent.py` — hướng dẫn điều tra tuần tự: sau mỗi tool call, đọc kết quả rồi mới quyết định bước tiếp theo; khi một incident dẫn tới incident khác thì gọi incident đó ở lượt kế tiếp, không gộp mọi tra cứu vào cùng một lượt.
- Vì sao chọn thay đổi này: lỗi này ảnh hưởng trực tiếp 2 case (H1, H6) đều bị trần 60 điểm vì cùng vi phạm `min_tool_steps`; đây là lỗi hành vi gọi tool, sửa ở system prompt là đúng tầng (không phải sửa dataset hay grading).

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 91.73/100 | 88.32/100 |
| Điểm case đã chọn (H1) | 60 | **90** |
| Số model calls (tổng 22 case) | 48 | 51 |
| Token agent (prompt/completion) | 35.122 / 15.770 | 43.548 / 13.611 |
| Token judge (prompt/completion) | 42.190 / 4.831 | 38.054 / 4.785 |

- Lỗi đã giải quyết: H1 (60→90) — sau cải tiến, trace cho thấy step 1 đọc `h02n9572mx9y` và `yaa0witbl2me`, step 2 theo liên kết đọc `xr3myzj7ms51`; hết vi phạm `min_tool_steps`, câu trả lời có đủ nguyên nhân/ảnh hưởng của incident gộp và nguồn từng bên.
- Lỗi còn lại hoặc mới xuất hiện:
  - H6 vẫn 60: model vẫn gộp `get_status` + `get_incident_details` vào cùng step 1 ở case này; prompt chưa đủ mạnh để thay đổi hành vi khi danh sách việc ngắn.
  - H2 giảm 100→50: lần này agent chỉ gọi `get_incident_details`, bỏ `get_status` (vi phạm `requires_tools`) dù nội dung trả lời đúng — biến đổi hành vi, không do lỗi logic agent.
  - E6 giảm 100→55: khi gặp timeout giả lập, agent retry `get_status` 3 lần rồi trả lời lửng lơ, hỏi lại người dùng mã incident thay vì kết luận theo bằng chứng đã có.
  - E2/E3 giảm 10 điểm mỗi case: cùng đúng dữ liệu nhưng judge trừ vì diễn giải khác — biến động judge.

## 5. Giới hạn

- Điểm judge là ước lượng của LLM: cùng câu trả lời đúng dữ liệu có thể được chấm khác nhau giữa hai lần (E2, E3); judge mặc định dùng cùng model với agent nên có thể thiên lệch.
- Điểm tổng giảm (91.73→88.32) dù cải tiến đạt mục tiêu (H1 +30, A4 +13, E1 +10): phần giảm đến từ biến đổi hành vi/judge ở các case khác (E6 −45, H2 −50), cho thấy một lần chạy mỗi phiên bản không chứng minh độ ổn định — cần nhiều lần chạy để phân biệt cải tiến thật và nhiễu loạn ngẫu nhiên.
- Snapshot cố định (09/11/2026) không phản ánh trạng thái thực tế hiện tại; tool chỉ tìm trong 50 incident gần nhất nên kết quả rỗng không chứng minh incident không tồn tại.
- Hai lần chạy dùng cùng cấu hình, nhưng nhiệt độ mẫu của model không cố định bằng 0 nên hành vi có thể khác nhau giữa các lần chạy.
