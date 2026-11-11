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
  3. **Ba cổng kiểm tra sau answer** (mở rộng pattern validation nguồn mà GUIDE.md gợi ý): `validation_missing_source` (URL nguồn, yêu cầu của lab), `validation_missing_lookup` (ID nêu trong câu hỏi phải được tra cứu), `validation_unresolved_reference` (ID được nhắc trong kết quả incident phải được tra cứu nếu câu hỏi yêu cầu theo liên kết). Mỗi cổng chỉ chặn một lần và yêu cầu model sửa trong `max_steps`.
  4. Sửa hạ tầng (có ở **cả hai** lần chạy để so sánh công bằng): chuẩn hoá `content` dạng list của Mistral về string, retry 1 lần cho lỗi mạng tạm thời và `finish_reason='error'`, đặt `temperature=0` để giảm variance.
- Vì sao: lỗi đều là hành vi model (gộp call, đoán thay vì tra cứu, tuân theo chỉ dẫn bịa) nên sửa ở tầng prompt + cổng kiểm tra trong agent loop; bug type của Mistral sửa ở biên giới adapter (`_content_text`), đúng tầng "parse external data at boundaries".

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 90.14/100 | **96.86/100** |
| Điểm case đã chọn (H1) | 60 | **100** |
| Số model calls (22 case) | 49 | 58 |
| Token agent (prompt/completion) | 38.069 / 18.178 | 88.587 / 21.528 |
| Token judge (prompt/completion) | 43.833 / 4.565 | 45.841 / 4.905 |

- Lỗi đã giải quyết: H1 60→100, H3 50→100, H6 60→100, H7 55→100, E4 75→100. Cả 5 case hết vi phạm mechanical và trả lời đủ bằng chứng theo link.
- Lỗi còn lại: A7/E1/E2/E5 giảm 9–10 điểm mà trace hành vi không đổi — nhiễu judge trên câu trả lời vẫn đúng dữ liệu; A1 91, A5/A6 90 do judge trù chi tiết trình bày. Không còn case nào dưới 90.

## 5. Giới hạn

- Điểm judge là ước lượng của LLM, mặc định cùng model với agent nên có thể thiên lệch; các case A7/E1/E2/E5 thay đổi −10 giữa hai lần chạy dù hành vi agent tương đương cho thấy nhiễu judge cỡ ±10 điểm/case.
- Chỉ một lần chạy mỗi phiên bản; `temperature=0` giảm variance nhưng không loại bỏ hoàn toàn (LLM API không đảm bảo deterministic tuyệt đối).
- Cổng kiểm tra là kiểm tra sự hiện diện (URL đã tra cứu, ID đã gọi), không chứng minh mọi claim đều có nguồn.
- Snapshot cố định (09/11/2026) không phản ánh trạng thái thực tế hiện tại; tool chỉ tìm trong 50 incident gần nhất nên kết quả rỗng không chứng minh incident không tồn tại.
- Improvement tăng số model calls (49→58) và token (~2,3× prompt) do các vòng tra cứu và revision thêm vào — đánh đổi chi phí để lấy bằng chứng.
