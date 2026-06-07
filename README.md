I. Thông tin chung
Học viên: Nguyễn Tiến Đạt
Giảng viên hướng dẫn: TS. Đặng Ngọc Hùng — Bộ môn CNPM, Khoa CNTT1
Định hướng: Quản trị hệ thống / Vận hành → DevOps
Thời gian: 08/06/2026 – 06/07/2026 (04 tuần)
Repository: https://github.com/ntiendat897/PTIT.git


II. CHỦ ĐỀ
Đề tài: Tự động hóa triển khai phần mềm kết hợp giám sát logs tập trung

1. Hiện trạng
Quy trình hiện tại mang tính thủ công: lập trình viên (dev) sau khi hoàn thành code sẽ push lên Git; việc đưa lên môi trường production do người vận hành thực hiện bằng tay — đăng nhập máy chủ, pull code mới về và chạy các bước triển khai thủ công. Log của ứng dụng nằm phân tán trên từng máy chủ và chỉ người quản trị có quyền đăng nhập mới xem được.

2. Mô tả vấn đề
Các hạn chế:
 - Triển khai chậm, tốn công, dễ sai: mỗi lần lên production phải thao tác tay qua nhiều bước, mất thời gian, dễ thiếu/nhầm bước.
 - Rollback chậm: khi bản mới gặp lỗi, việc rollback về phiên bản cũ làm thủ công, gây dịch vụ gián đoạn kéo dài.
 - Khó truy vết và troubleshoot: log phân tán, chỉ quản trị viên xem được, nên tốn nhiều thời gian tìm nguyên nhân qua lại giữa dev và vận hành.
 - Thiếu kiểm soát chất lượng: không có build/kiểm thử/quét bảo mật tự động trước khi lên production.

3. Mục tiêu đề tài
Xây dựng quy trình CI/CD tự động kết hợp giám sát log tập trung nhằm giải quyết trực tiếp các vấn đề trên:
 - Tự động hóa chuỗi build → quét bảo mật → đóng gói → triển khai, giảm thao tác tay và sai sót.
 - Thiết lập cơ chế rollback nhanh để rút ngắn thời gian khôi phục khi sự cố.
 - Gom log tập trung trên ELK để dev và vận hành cùng tra cứu, giảm thời gian troubleshoot và không phụ thuộc một người.
 - Bổ sung cổng kiểm thử và quét bảo mật tự động để giảm tỉ lệ lỗi lọt ra production.

=> Tạo tiền đề cho AIOps: khi log/metric đã tập trung, có thể áp dụng học máy để tự phát hiện bất thường và hỗ trợ chẩn đoán/kiểm thử lỗi.

4. Lý do chọn đề tài
Đề tài xuất phát từ một bài toán có thật và cấp thiết tại nơi thực tập phù hợp với định hướng nghề nghiệp cá nhân (quản trị hệ thống/vận hành tiến tới DevOps và AIOps)
Khả thi để thực hiện và hoàn thành đưa ra một kết quả tốt trong 4 tuần với nền tảng kỹ năng hiện có, cho kết quả đo lường.


III. KẾ HOẠCH LÀM VIỆC

Thời gian: 04 tuần, 08/06/2026 – 06/07/2026. 

Tuần 0 (đến 08/06) — Chuẩn bị
 - Công việc: Chốt chủ đề
 - Kết quả đầu ra dự kiến: README (chủ đề + kế hoạch)
 - Mốc cần đạt: 08/06 - nộp URL repo

Tuần 1 (08/06 – 14/06)
 - Công việc: Khảo sát hiện trạng, thiết kế kiến trúc, viết đề cương
 - Kết quả đầu ra dự kiến: Tài liệu hiện trạng, sơ đồ kiến trúc, viết đề cương
 - Mốc cần đạt: Chốt baseline & thiết kế, viết đề cương
 
Tuần 2 (15/06 – 21/06)
 - Công việc: Dựng pipeline CI: build → quét bảo mật → đóng gói và đẩy image lên registry
 - Kết quả đầu ra dự kiến: Pipeline CI chạy tự động, image trên registry
 - Mốc cần đạt: CI chạy end-to-end

Tuần 3 (22/06 – 28/06)
 - Công việc: Xây dựng CD + auto-rollback; dựng ELK gom log tập trung
 - Kết quả đầu ra dự kiến: Pipeline CD hoàn chỉnh, rollback hoạt động, dashboard Kibana
 - Mốc cần đạt: Triển khai tự động từ 1 commit

Tuần 4 (29/06 – 05/07)
 - Công việc: Đo lại các chỉ số trước vào sau để so sánh, hoàn thiện báo cáo
 - Kết quả đầu ra dự kiến: Bảng so sánh ở báo cáo cuối
 - Mốc cần đạt: Nộp báo cáo