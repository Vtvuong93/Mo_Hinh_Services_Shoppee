## 2. Các Thành Phần Xử Lý Cốt Lõi

hệ thống sử dụng chiến lược **"Phòng thủ nhiều lớp"** và tuyệt đối **không** cho người dùng truy cập trực tiếp vào cơ sở dữ liệu. 

### 1) CDN & Cloudflare (Mạng Phân Phối Trợ Tốc & Bảo Mật)
* **Khái niệm:** Mạng lưới các máy chủ lưu trữ bản sao nội dung tĩnh kiêm hệ thống phân giải tên miền (DNS) toàn cầu, hoạt động dưới dạng máy chủ ẩn danh trung gian (Reverse proxy), được đặt rải rác ở nhiều vị trí địa lý.
* **Tác dụng:** 
  - **(CDN):** Mạng phân phối nội dung. CDN lưu sẵn các tài nguyên tĩnh (ảnh, video, giao diện) tại trung tâm dữ liệu (lưu trữ đệm) gần người dùng về mặt vật lý nhất để tăng tốc tải trang.
  - **Bảo mật mạng (WAF & Proxy):** Nằm giữa người dùng và máy chủ gốc để ẩn IP thật. Nó kiểm tra, lọc mọi luồng lưu lượng và chặn các truy cập độc hại, chống tấn công DDoS hiệu quả. Phân giải DNS cũng được xử lý siêu tốc.
* **Ví dụ:** Một tài khoản ở Hà Nội khi lướt web tải ảnh thì bức ảnh đó được lấy từ máy phân phối nằm ngay tại VN (hoặc Singapore).
### 2) Cân bằng tải (Load Balancer)
* **Khái niệm:** Một phần cứng hoặc phần mềm chuyên điều tiết giao thông mạng. Nó nhận một địa chỉ IP logic ở lối vào và định tuyến tới một tập hợp nhiều máy chủ vật lý ở đằng sau phòng khi có cả triệu request ùa tới.
* **Tác dụng:** Nó đánh giá tình trạng khối lượng công việc hiện hành của hàng nghìn máy chủ. Từ đó thông minh chia đều các yêu cầu đến các máy chủ web có công suất khả dụng cao nhất và trống việc nhất. Lợi ích lớn lao nhất là ngay cả khi một số máy chủ bất ngờ mất điện hư máy, tải sẽ lập tức được tự động san sẻ ra các máy còn sống, giúp quá trình dùng web của người dùng không hề bị nghẽn đứt.
* **Ví dụ:** Lễ tân trạm y tế: Có 10,000 bệnh nhân đổ bộ vào. Lễ tân (bộ cân bằng tải) sẽ tự động xếp 5,000 người vào phòng khám của Nhóm tư vấn A, 5,000 người gặp Nhóm B. Luôn đảm bảo không bác sĩ nào bị quá tải đứng tim.

### 3) Kiến Trúc Microservices
* **Khái niệm:** Phương pháp chia nhỏ một ứng dụng nguyên khối  thành hàng trăm module nhỏ chuyên biệt lo 1 nghiệp vụ độc lập.
* **Ví dụ:** Ứng dụng Shopee: Nếu hôm đó phân hệ thẻ "Thanh Toán" bị treo giật, người dùng vẫn có thể "Tìm kiếm hàng", "Nhắn tin hỏi Shop" thoải mái vì các chức năng đó nằm ở cụm máy chủ khác.

### 4) In-memory Cache (Bộ Nhớ Đệm Redis)
* **Khái niệm:** Cơ sở dữ liệu tạm thời chạy truy xuất dữ liệu trực tiếp trên Bộ nhớ trong (RAM) của máy chủ với thời gian ngắn.
* **Tác dụng:** Giải pháp là đem toàn bộ dữ liệu thường xuyên được hàng dài người dùng gọi nhiều (lượt like, danh sách món hàng sale) để sẵn trên bộ đệm RAM. Lớp cache này hấp thụ tới hơn 80-90% lượt tải đáng nhẽ giáng thẳng xuống Database truyền thống.
* **Ví dụ:** Mỗi lần vào profile Facebook, thay vì phải vào cơ sở dữ liệu để tính toán lấy ra con số lượt theo dõi, hệ thống rẽ nhánh nhặt luôn số đếm có sẵn đó nằm ngay ngắn trên RAM.

### 5) Message Queue (Hàng Đợi Kafka)
* **Khái niệm:** Nền tảng luồng thông điệp được thiết kế lập các hàng đệm trung chuyển chờ đợi giao dịch để ứng dụng làm việc lệch pha (bất đồng bộ).
* **Tác dụng:** tạo "Phòng chờ xếp hàng". Vào dịp lễ tết có triệu người dùng tương tác thả tim cực mạnh tại một mốc giờ vàng. Nếu hệ thống ép DB phục vụ ngay lập tức bằng mọi giá thì Server chết sặc liền. Kafka sẽ làm phễu đẩy hết luồng yêu cầu ồ ạt khổng lồ cất kho vào Hàng Đợi. Các server ở hậu phương cứ túc tắc từ từ rút từng mẩu ra xử lý theo năng lực vật lý thật sự.

### 6) Database Scaling (Phân Mảnh & Nhân Bản Cơ Sở Dữ Liệu)
* **Tác dụng:**
  - **Tạo bản sao & Chia Đọc/Ghi (Read/Write Replica):** Phần lớn hành vi người dùng là Lướt (Đọc). Ta quy hoạch đúng 1 DB hệ trọng (Master) chuyên sâu phục vụ nhu cầu Ghi/Sửa. Bên cạnh đó, dựng lập hàng chục chiếc máy bản sao nhân bản DB Master sang (Replica) nhằm chia lửa riêng rẽ tiếp đón toàn bộ khách ghé thăm xin yêu cầu Đọc.
  - **Phân mảnh Sharding :** Chặt và xé nhỏ cái túi database khổng lồ thành nhiều ngăn phần cứng vật lý hoàn toàn tách biệt. Thường dùng "Khóa phân vùng" qua giá trị băm mã ID. Từ khối lượng tỉ người gom một mối giờ đây việc ghi chép tra tìm được bẻ nhỏ, chạy song song với nhau trên hàng trăm máy.

