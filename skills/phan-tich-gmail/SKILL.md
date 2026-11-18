---
name: "phan-tich-gmail"
description: "Dùng khi cần phân tích Gmail công việc: thống kê thư theo khách/nhãn/người xử lý, thời gian phản hồi, báo giá, thư cần theo dõi."
---

# Phân tích Gmail công việc (Yoshita)

Người dùng không rành IT: trả lời bằng tiếng Việt, câu ngắn, đơn giản. Nêu việc đã làm và việc nên làm tiếp.

## Bước làm
1. Hỏi rõ (nếu chưa có): khoảng thời gian, tài khoản, mục tiêu (khách nào, người nào, nhãn nào).
2. Dùng kết nối Gmail để tìm thư (search_threads), đọc thư cần thiết (get_thread). Chỉ ĐỌC, không gửi, không xóa, không gắn nhãn trừ khi được yêu cầu.
3. Gom dữ liệu thành bảng: ngày, người gửi/nhận, khách, chủ đề, nhãn, người xử lý, trạng thái.
4. Phân tích theo mục tiêu:
   - Số thư theo nhãn (Hóa đơn, Quảng cáo, Business, Shopee...).
   - Thời gian phản hồi của Phương, Đoàn, Tiến với thư khách.
   - Thư khách chưa được trả lời hoặc chờ báo giá quá 2 ngày.
   - Khách hỏi lại nhiều lần (dấu hiệu trả lời chưa đủ).
   - Việc cần làm kèm hạn (ví dụ xác nhận đơn, họp, báo giá).
5. Kết quả: tóm tắt 5 dòng đầu (số liệu chính, việc cần chú ý), sau đó bảng chi tiết. Nếu cần file thì tạo Excel (skill xlsx) hoặc biểu đồ (skill data:create-viz).
6. Cuối cùng ghi: "Việc nên làm tiếp: ...".

## Lưu ý
- Thư có thông tin khách, giá: giữ riêng tư, không đưa ra dịch vụ ngoài.
- Không đoán: dữ liệu thiếu thì nói rõ là thiếu.
- Nếu hỏi nhanh/chuyển tin thì dùng Sonnet; phân tích tổng hợp dùng model mạnh hơn.