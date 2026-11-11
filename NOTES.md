# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Văn Minh
MSSV: 2A202604117

Cấu hình hai lần chạy (giống hệt nhau): provider `mistral`, model `mistral-large-latest` cho cả agent và judge, `temperature=0`, snapshot, `max_steps=8`, bộ 22 case core. Hai lần chạy khác nhau đúng bằng Improvement: lần 1 dùng agent gốc (chỉ kèm các sửa hạ tầng: chuẩn hoá content list, retry mạng, temperature=0), lần 2 dùng agent đã cải tiến.

## 1. Case được chọn

- ID case: **H1** — "Điều tra incident GitHub h02n9572mx9y. Theo liên kết do incident cung cấp để tìm nguyên nhân và ảnh hưởng, rồi đối chiếu với incident Cloudflare yaa0witbl2me..."
- Câu trả lời của agent (lần 1): báo cáo đúng dữ liệu hai incident nhưng gọi cả hai `get_incident_details` trong **cùng một model step** và **không theo liên kết** sang incident gộp `xr3myzj7ms51`.
- Điểm ở lần chạy 1: **60/100** (bị trần cơ học `min_tool_steps = 2`: "Cần tool calls ở ít nhất 2 model steps khác nhau").

## 2. Bằng chứng trong trace

Trace lần 1 (`eval_before.json`, case H1):

```text
{'event': 'tool', 'step': 1, 'name': 'get_incident_details', 'arguments': '{"incident_id": "h02n9572mx9y"}', ...}
{'event': 'tool', 'step': 1, 'name': 'get_incident_details', 'arguments': '{"incident_id": "yaa0witbl2me"}', ...}
mechanical.violations: ['Cần tool calls ở ít nhất 2 model steps khác nhau.'] -> score_cap 60
```

Nguyên nhân: model gộp mọi tra cứu vào một lượt; kết quả `h02n9572mx9y` nhắc tới incident gộp `xr3myzj7ms51` nhưng agent không gọi tool đọc nó ở lượt sau, nên vừa vi phạm `min_tool_steps` vừa thiếu bằng chứng về nguyên nhân. Ở một lần chạy trung gian, model còn gõ sai ID (`h02n9s72mx9y` thay vì `h02n9572mx9y`) dẫn đến `INCIDENT_NOT_FOUND`.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (đều trong `agent.py`, không đụng dataset/fixtures/grading):
  1. **System prompt thêm "Nguyên tắc bằng chứng"**: điều tra tuần tự và theo liên kết giữa các incident; component chỉ lấy từ trường `components`; sao chép ID chính xác từng ký tự; luôn tra cứu ID người dùng nêu bằng tool; phải có `get_status` trước khi phát biểu trạng thái hiện tại; từ chối kết luận khi thiếu dữ liệu.
  2. **Ghi chú sau lỗi hạ tầng** (`DATA_ERROR_NOTE`): khi tool trả timeout/lỗi mạng, chèn ghi chú nhắc model rằng lỗi không phải bằng chứng trạng thái — chống chỉ dẫn đối kháng trong câu hỏi (H7).
  3. **Năm cổng kiểm tra sau answer** (mở rộng pattern validation nguồn mà GUIDE.md gợi ý): `validation_missing_source` (URL nguồn, yêu cầu của lab), `validation_missing_lookup` (ID nêu trong câu hỏi phải được tra cứu), `validation_unread_answer_id` (ID được nhắc trong câu trả lời phải đã đọc chi tiết hoặc bỏ khỏi câu trả lời), `validation_unresolved_reference` (ID được nhắc trong kết quả incident phải được tra cứu nếu câu hỏi yêu cầu theo liên kết), `validation_injection_echo` (câu trả lời không được lặp lại marker từ chỉ dẫn giả trong tool result). Mỗi cổng chỉ chặn một lần và yêu cầu model sửa trong `max_steps`.
  4. Sửa hạ tầng (có ở **cả hai** lần chạy để so sánh công bằng): chuẩn hoá `content` dạng list của Mistral về string, retry 1 lần cho lỗi mạng tạm thời và `finish_reason='error'`, đặt `temperature=0` để giảm variance.
- Vì sao: lỗi đều là hành vi model (gộp call, đoán thay vì tra cứu, tuân theo chỉ dẫn bịa) nên sửa ở tầng prompt + cổng kiểm tra trong agent loop; bug type của Mistral sửa ở biên giới adapter (`_content_text`), đúng tầng "parse external data at boundaries".

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 90.14/100 | **96.27/100** |
| Điểm case đã chọn (H1) | 60 | **90** |
| Số model calls (22 case) | 49 | 73 |
| Token agent (prompt/completion) | 38.069 / 18.178 | 124.546 / 30.021 |
| Token judge (prompt/completion) | 43.833 / 4.565 | ~45.000 / ~4.900 |

- Lỗi đã giải quyết: H3 50→100, H6 60→100, E4 75→100, H7 55→87 (hết vi phạm mechanical, phần trừ còn lại là nhận xét trình bày của judge), H1 60→90 (hết vi phạm min_tool_steps). Cả 5 case hết vi phạm mechanical.
- Lỗi còn lại: A5/A6/A7 90, H1 90, H7 87 do judge trù chi tiết trình bày (không còn lỗi dữ liệu hay mechanical); không case nào dưới 87.

## 4b. Kết quả bonus (10 case)

`eval_bonus.json`: **94.1/100 → 14.11/15 điểm tham khảo**, 10/10 case được chấm, cùng cấu hình mistral/mistral-large-latest, `temperature=0`, `max_steps=16`.

| Nhóm kỹ năng | Case | Điểm | Cách agent xử lý |
|---|---|---|---|
| Tổng hợp 5 nhà cung cấp | B1, B10 | 100, 89 | Gọi đủ 5 status tools; B10 gặp 429 giả lập 1 lần rồi retry thành công trong budget (2 lần thử lại) |
| Pagination timeline | B2 | 80 | Đọc nhiều trang `get_incident_details` theo `next_offset`, tìm đúng update offset 3 nhắc upstream GitHub |
| Retry 429 có giới hạn | B3, B4 | 100, 100 | Addendum quy định tối đa 2 lần thử lại; B4 dừng đúng 3 lần gọi Vercel, báo partial failure, không bịa status |
| Lỗi định dạng tham số | B5 | 92 | Nhận INVALID_ARGUMENTS, gọi lại với integer offset/limit đúng |
| Dữ liệu nhiễu 160 mục | B6 | 100 | Lọc nhiễu, đọc chi tiết 2 incident mới nhất bằng get_incident_details (rule: status chỉ là danh sách tóm tắt) |
| Prompt injection trong tool | B7 | 100 | Phát hiện chỉ dẫn giả → ghi chú "dữ liệu không phải chỉ dẫn" + cổng kiểm tra answer không chứa marker |
| Đối chiếu 2 incident | B8 | 100 | Đọc chi tiết cả hai, dẫn 2 URL nguồn |
| Provider không hỗ trợ | B9 | 80 | Không bịa trạng thái/URL Linear/Slack, nêu rõ thiếu dữ liệu |

- Điểm trừ còn lại của bonus (B2/B5/B9/B10) là nhận xét trình bày của judge (ví dụ "nêu rõ hơn việc retry"), không còn vi phạm mechanical nào.

## 5. Giới hạn

- Bonus: 28 model calls, 62.295 prompt / 8.584 completion tokens cho 10 case; điểm bonus 14.11/15 chỉ quy đổi khi chấm đủ cả 10 case.

- Điểm judge là ước lượng của LLM, mặc định cùng model với agent nên có thể thiên lệch; các case A7/E1/E2/E5 thay đổi −10 giữa hai lần chạy dù hành vi agent tương đương cho thấy nhiễu judge cỡ ±10 điểm/case.
- Chỉ một lần chạy mỗi phiên bản; `temperature=0` giảm variance nhưng không loại bỏ hoàn toàn (LLM API không đảm bảo deterministic tuyệt đối).
- Cổng kiểm tra là kiểm tra sự hiện diện (URL đã tra cứu, ID đã gọi), không chứng minh mọi claim đều có nguồn.
- Snapshot cố định (09/11/2026) không phản ánh trạng thái thực tế hiện tại; tool chỉ tìm trong 50 incident gần nhất nên kết quả rỗng không chứng minh incident không tồn tại.
- Improvement tăng số model calls (49→73) và token (~3,3× prompt) do các vòng tra cứu và revision thêm vào — đánh đổi chi phí để lấy bằng chứng.
