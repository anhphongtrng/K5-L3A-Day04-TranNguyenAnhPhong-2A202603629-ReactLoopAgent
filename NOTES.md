# Ghi chú phân tích · Day 04

Họ và tên: TRẦN NGUYỄN ANH PHONG
MSSV: 2A202603629

## 1. Case được chọn

- ID case: A1
- Câu hỏi: Đối chiếu GitHub Actions và Cloudflare Spectrum, nêu incident, trạng thái và phạm vi, không suy luận hai sự cố có cùng nguyên nhân, đồng thời dẫn URL cho cả hai.
- Câu trả lời của agent lần chạy đầu: Mô tả GitHub Actions operational và các incident đã resolved; Cloudflare Spectrum có incident Phoenix đang monitoring. Câu trả lời có dẫn nguồn nhưng một số URL incident không khớp URL chính xác được trả về từ `get_incident_details`.
- Điểm ở lần chạy 1: 78/100.

## 2. Bằng chứng trong trace

Trace lần chạy đầu có kết quả lấy chi tiết incident cho cả GitHub và Cloudflare. Ví dụ, tool trả URL incident GitHub `t89g2q1800fr` là `https://stspg.io/lv4q3xm9r0rk`; tool trả URL incident Spectrum Phoenix là `https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1`.

Trong trace `eval_after.json`, sau khi model tạo câu trả lời ở bước 3, agent ghi nhận:

```text
{"event":"validation_missing_source","step":3,"urls":["https://stspg.io/lv4q3xm9r0rk"]}
```

Agent gọi model thêm lần nữa ở bước 4 để sửa câu trả lời và thêm URL đã được tool quan sát. Cả hai lần chạy đều dùng evidence cho thấy GitHub Actions operational, các incident GitHub được nêu đã resolved, còn Spectrum Availability Issue in Phoenix đang monitoring. Trace không cho thấy hai dịch vụ có cùng nguyên nhân.

## 3. Thay đổi đã thực hiện

- Trong `agent.py`, lưu URL từ kết quả `get_incident_details` thành công. Khi câu trả lời cuối thiếu một URL incident đã quan sát, ghi event `validation_missing_source`, đưa yêu cầu sửa cho model và tiếp tục trong giới hạn `max_steps`.
- Chọn thay đổi này để tăng khả năng câu trả lời có URL trực tiếp cho incident đã tra cứu. Đây là kiểm tra sự hiện diện của URL trong câu trả lời, không xác minh toàn bộ nội dung hoặc mọi claim đều chính xác.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 83.45/100 | 83.73/100 |
| Điểm A1 | 78/100 | 85/100 |
| Số model calls của A1 | 3 | 4 |
| Tool calls của A1 | 8 | 8 |

- Điểm A1 tăng 7 điểm. Ở lần chạy sau, trace cho thấy thiếu URL được phát hiện và model nhận thêm một lượt để sửa câu trả lời.
- Lỗi được nhắm đến là thiếu URL chính xác từ incident detail trong câu trả lời. Sau validation, agent tạo thêm câu trả lời có các URL incident.
- Lỗi còn lại: điểm judge vẫn góp ý cần làm rõ trạng thái hiện tại của cả hai dịch vụ và nguồn cho mọi thông tin. Validation chỉ kiểm tra sự hiện diện của URL; nó không chứng minh nội dung được URL hỗ trợ đầy đủ hoặc ngăn mọi sai sót thực tế.

## 5. Giới hạn

- Điểm LLM judge là ước lượng và có thể dao động; chênh lệch 7 điểm trong một lần chạy không tự nó chứng minh toàn bộ cải thiện đến từ thay đổi trong agent.
- Hai file dùng cấu hình cùng model, judge, dataset và `max_steps`, nhưng câu trả lời sinh ra có thể thay đổi giữa các lần chạy.
- Snapshot phản ánh dữ liệu tại thời điểm được ghi trong trace. Danh sách incident giới hạn theo dữ liệu tool trả về; không tìm thấy incident không chứng minh rằng incident đó chưa từng tồn tại.
- Kiểm tra URL chỉ xác nhận URL đã quan sát có mặt trong câu trả lời; không xác nhận chất lượng nguồn, tính đúng của trạng thái, phạm vi, thời gian hay quan hệ nhân quả.
