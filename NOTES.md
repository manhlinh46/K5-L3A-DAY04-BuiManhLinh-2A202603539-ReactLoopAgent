# Ghi chú phân tích · Day 04

Họ và tên: Bùi Mạnh Linh
MSSV: 2A202603539

## 1. Case được chọn

- ID case: H6
- Lý do chọn: Trong lần chẩn đoán ban đầu, H6 kết thúc với `max_steps` sau nhiều lần thiếu URL nguồn; trace của lần đó được ghi lại bên dưới. Kết quả `eval_before.json` hiện tại là một lần chạy lại hoàn chỉnh, trong đó H6 đạt 100/100. Không sửa điểm hay nội dung báo cáo eval.
- Câu trả lời ở lần lỗi ban đầu: `### 1. Trạng thái hiện tại của GitHub Actions - **Trạng thái thành`
- Điểm lần lỗi ban đầu: 0/100 (`max_steps`). Đây là điểm case ở lần chạy đó, không phải điểm tổng.
- Kết quả H6 trong hai báo cáo hoàn chỉnh hiện có: `eval_before.json` 100/100; `eval_after.json` 100/100.

## 2. Bằng chứng trong trace

Trace của lần lỗi ban đầu cho thấy agent đã gọi `get_incident_details` với `h02n9572mx9y`, theo liên kết tới `xr3myzj7ms51`, rồi gọi `get_status` cho GitHub Actions. Sau đó, ở các bước 4–8, agent đều trả lời nhưng phát sinh `validation_missing_source` với hai URL `https://stspg.io/7pwv28hvkmz8` và `https://stspg.io/pjgptg3cfc6n`. Ở bước 8 vẫn thiếu URL nên agent kết thúc với `max_steps`.

Nguyên nhân có thể xảy ra: yêu cầu sửa trước đây chỉ nêu URL còn thiếu, không hướng dẫn cách đưa URL vào câu trả lời hay giữ câu trả lời ngắn gọn. Model lặp lại câu trả lời mà không thỏa điều kiện nguồn, tiêu hao toàn bộ số bước. Lưu ý: trace lỗi này thuộc lần chạy ban đầu; báo cáo hiện tại đã được chạy lại và H6 đạt 100 ở cả before lẫn after.

```text
H6: status=max_steps, score=0
step 1: get_incident_details(h02n9572mx9y)
step 2: get_incident_details(xr3myzj7ms51)
step 3: get_status(Actions)
steps 4–8: validation_missing_source với
  https://stspg.io/7pwv28hvkmz8
  https://stspg.io/pjgptg3cfc6n
step 8: max_steps
```

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Khi thiếu nguồn, `agent.py` yêu cầu viết lại câu trả lời ngắn gọn và hoàn chỉnh, giữ tất cả URL incident đã quan sát trong cùng mục Nguồn, giữ các kết luận có bằng chứng và không gọi thêm tools. `tests/test_validation.py` kiểm tra contract URL và việc yêu cầu giữ nhiều URL cùng lúc.
- Vì sao chọn thay đổi này: Trace H6 ban đầu cho thấy model lặp lại khi thiếu nguồn. Một lần chạy A1 trước đó cũng cho thấy hai URL luân phiên bị thiếu; vì vậy nhắc lại toàn bộ tập URL thay vì chỉ URL còn thiếu. Trong hai báo cáo hoàn chỉnh hiện tại, H6 được 100 ở cả hai lần và không phát sinh `validation_missing_source`; do đó chưa thể quy mức điểm H6 cho thay đổi này. Kiểm tra URL chỉ xác minh sự hiện diện, không xác minh mọi claim.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Case được chấm | 22/22 | 22/22 |
| Điểm tổng | 99.86/100 | 99.73/100 |
| Điểm H6 | 100/100 | 100/100 |
| Agent model calls (tổng) | 61 | 60 |
| Judge calls | 22 | 22 |
| Agent tokens (tổng) | 62,581 | 63,176 |
| Judge tokens (tổng) | 52,432 | 49,464 |
| Tokens tổng (agent + judge) | 115,013 | 112,640 |
| Chi phí | Không có trong báo cáo | Không có trong báo cáo |

- Quan sát sau thay đổi: Hai báo cáo hiện tại đều hoàn tất đủ 22 case; H6 hoàn tất và đạt 100 ở cả hai. Ở H6, kiểm tra thiếu nguồn không được kích hoạt trong hai lần này, nên không thể kết luận thay đổi đã cải thiện riêng H6. A1 vẫn phát sinh một lần nhắc URL nhưng kết thúc thành công.
- Lỗi còn lại hoặc mới xuất hiện: Điểm tổng giảm 0.13 ở lần chạy after. Hai case đổi điểm là A2 (99 → 94) và H1 (98 → 100); các case còn lại không đổi. Riêng A1 vẫn có một sự kiện `validation_missing_source` ở bước 3 ở cả hai báo cáo, và đều hoàn tất với 100 điểm. Một lần chạy cho mỗi phiên bản cùng judge LLM không đủ để kết luận thay đổi gây ra biến động điểm; kết quả judge/model có thể khác nhau.

## 5. Giới hạn

- Điểm do LLM judge chấm là ước lượng, không phải bằng chứng tuyệt đối; judge có thể diễn giải khác nhau giữa các lần chạy.
- Snapshot là dữ liệu giả lập/cố định, không phản ánh trạng thái dịch vụ trực tiếp.
- Mỗi phiên bản chỉ chạy một lần; kết quả có thể dao động theo phản hồi của model và judge.
- Bộ kiểm tra URL chỉ kiểm tra URL đã quan sát có xuất hiện trong câu trả lời; không chứng minh mọi nhận định đều có nguồn hay chính xác.
- Báo cáo không cung cấp chi phí API, nên chỉ ghi được token usage.
