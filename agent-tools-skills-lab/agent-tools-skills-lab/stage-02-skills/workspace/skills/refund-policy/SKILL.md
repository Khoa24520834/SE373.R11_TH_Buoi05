---
name: refund-policy
description: Tra cứu chính sách hoàn tiền trong data/policies/ và xác định một yêu cầu hoàn tiền có đủ điều kiện không, kèm phí và tài liệu làm căn cứ. Dùng khi người dùng hỏi có được hoàn tiền không, điều kiện, thời hạn hoặc phí hoàn tiền của một đơn hàng.
---

# Refund policy

Hướng dẫn trả lời câu hỏi hoàn tiền dựa trên tài liệu chính sách trong workspace. Chỉ dùng nội dung đọc được từ tài liệu; không dùng kiến thức sẵn có hay tài liệu đọc ở cuộc trò chuyện khác.

## Các bước

1. Kiểm tra câu hỏi có đủ 3 thông tin: **ngày mua**, **ngày yêu cầu hoàn**, **trạng thái kích hoạt** của sản phẩm (đã/chưa kích hoạt).
   - Thiếu bất kỳ thông tin nào: hỏi lại đúng thông tin còn thiếu và dừng. Không giả định (ví dụ không tự coi là "chưa kích hoạt"), không đưa kết luận tạm.
2. Tìm tài liệu: gọi `list_files` với `data/policies`. Không đoán tên file; tên file có thể thay đổi. Nếu có thư mục con, liệt kê tiếp. Nếu thư mục không tồn tại hoặc rỗng, báo không tìm thấy tài liệu chính sách và dừng.
3. Đọc bằng `read_file` các file chính sách tìm được để biết **phạm vi hiệu lực** của từng file (dòng "Áp dụng cho ngày mua ..."). Đọc cả các file để chắc chắn chọn đúng, không chọn theo tên file.
4. Chọn đúng một chính sách có phạm vi hiệu lực chứa **ngày mua** (chú ý "trước" là không bao gồm, "từ ..., bao gồm ngày này" là bao gồm). Nếu không có hoặc có nhiều hơn một chính sách khớp, báo mâu thuẫn và dừng.
5. Tính **số ngày đã qua** = chênh lệch ngày lịch giữa ngày yêu cầu hoàn và ngày mua (ngày yêu cầu − ngày mua). Dùng ngày trong câu hỏi, không dùng ngày hiện tại của máy. Ngày dạng dd/mm/yyyy.
6. Kết luận theo chính sách đã chọn:
   - Sản phẩm đã kích hoạt mà chính sách cấm → không đủ điều kiện.
   - Số ngày đã qua lớn hơn giới hạn → không đủ điều kiện. Bằng đúng giới hạn vẫn đủ điều kiện về thời gian.
   - Còn lại → đủ điều kiện; nêu phí theo chính sách (hoặc không thu phí).
7. Đọc `references/answer-template.md` (tức `skills/refund-policy/references/answer-template.md`) và trả lời đúng theo mẫu. Đường dẫn căn cứ phải là `path` trong kết quả `read_file` của chính sách đã chọn.
