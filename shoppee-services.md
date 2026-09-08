# Mô hình kiến trúc các service của Shopee

## 1. Tổng quan kiến trúc

Nguyên tắc chính:

- Mỗi service phụ trách một chức năng và sở hữu cơ sở dữ liệu riêng.
- Service khác không truy cập trực tiếp vào cơ sở dữ liệu của service đó; việc trao đổi phải đi qua API hoặc sự kiện.
- Giao tiếp đồng bộ: Dùng HTTP/REST hoặc gRPC cho các luồng cần phản hồi tức thì.
- Giao tiếp bất đồng bộ: Dùng hàng đợi/sự kiện (Kafka/RabbitMQ) cho các tác vụ xử lý nền

## 2. Đăng nhập và lưu mật khẩu

### 2.1. Lưu password_hash trong Auth DB
- **Auth Service** quản lý xác thực và lưu thông tin `password_hash` vào Auth DB (có kèm salt). Chỉ Auth service có thể truy cập
+ Không lưu mật khẩu thô, không mã hoá để giải mã ngược.
+ Dùng thuật toán băm 1 chiều, không thể dịch ngược về mật khẩu ban đầu
+ Mỗi mật khẩu cí 1 salt ngẫu nhiên riêng.

```text
Người dùng tạo mật khẩu
  -> HTTPS -> API Gateway -> Auth Service
  -> băm bằng thuật toán một chiều + salt riêng
  -> Auth DB: lưu password_hash, salt và tham số băm
```

### 2.2. Truy vấn tài khoản bằng email/số điện thoại
- Hệ thống dùng email hoặc số điện thoại (được lập chỉ mục) để tìm kiếm người dùng.
+ Auth service tìm bản ghi bằng email/số điện thoại
+ bản ghi trong đó có password_hash, salt và tham số băm

```text
Người dùng gửi email/số điện thoại + mật khẩu
  -> Gateway -> Auth Service
  -> Auth DB: tìm bản ghi bằng chỉ mục email/số điện thoại
  -> Lấy ra password_hash, salt, tham số băm
```

### 2.3. Băm và đối chiếu mật khẩu
- Khi có thông tin, hệ thống băm mật khẩu vừa nhận với salt đã lưu và đối chiếu với `password_hash` gốc.

```text
(Tiếp theo luồng trên)
  -> băm mật khẩu nhập vào với salt đã lưu
  -> so sánh với password_hash
```

### 2.4. Trả Session hoặc JWT
- Nếu xác thực thành công, hệ thống trả về Phiên (Session) hoặc Access Token (JWT).

```text
Xác thực thành công
  -> tạo Session ID và lưu phiên trong Redis
     hoặc ký access token JWT
  -> trả cookie/token cho ứng dụng
  -> các request sau gửi cookie/token để xác thực
```

**Luồng hoạt động đầy đủ:**
```text
Ứng dụng -> HTTPS -> API Gateway -> Auth Service
  -> Auth DB: tìm tài khoản và lấy password_hash
  -> băm + so sánh mật khẩu
  -> kiểm tra trạng thái tài khoản/MFA
  -> trả Session ID hoặc JWT
  -> các request sau: xác minh phiên/token -> xử lý nghiệp vụ
```

## 3. Tìm kiếm và kết quả tức thì

### 3.1. Cập nhật dữ liệu và tạo Inverted Index
- **Product Service cập nhật dữ liệu:** Lưu sản phẩm gốc ở DB và phát sự kiện đồng bộ.
- **Search Service xây dựng Inverted Index:** Dùng chỉ mục đảo ngược để ánh xạ từ khóa với danh sách sản phẩm.

```text
Product Service -> Product DB: lưu sản phẩm gốc
Product Service -> Event Bus/Kafka: ProductChanged
Event Bus -> Search Service: cập nhật tài liệu
Search Service -> Search Index: tách từ và tạo Inverted Index
```

### 3.2. Truy vấn, Cache và Xếp hạng
- **Client truy vấn Search Service:** Gửi yêu cầu tìm kiếm khi gõ nhanh (autocomplete) hoặc tìm toàn văn.
- **Redis xử lý cache/autocomplete:** Lấy kết quả lưu trữ nhanh cho các truy vấn phổ biến.
- **Search Index trả kết quả và xếp hạng:** Tìm tập hợp phù hợp nhất trong Index rồi xếp hạng, lọc.

```text
Người dùng gõ "điện tho" (Autocomplete)
  -> Gateway -> Search Service
  -> chuẩn hóa từ khóa
  -> Redis: kiểm tra cache
  -> nếu không có: truy vấn Search Index
  -> trả từ khóa và sản phẩm gợi ý
```

```text
Người dùng nhấn tìm kiếm (Client truy vấn Search Service)
  -> Gateway -> Search Service
  -> chuẩn hóa truy vấn và áp dụng bộ lọc
  -> Redis hoặc Search Index
  -> xếp hạng, phân trang và trả danh sách sản phẩm

Search Index (Trả kết quả và xếp hạng)
  -> tập sản phẩm phù hợp
  -> tính độ khớp từ khóa
  -> bộ lọc nghiệp vụ và tồn kho
  -> quảng cáo/cá nhân hóa
  -> chọn trang đầu và trả kết quả
```

## 4. Các service giao tiếp với nhau

| Luồng | Giao thức |
| :--- | :--- |
| Client → Gateway | HTTPS/REST |
| Gateway → Auth/Search | REST |
| Product → Search | Event/Kafka |
| Search → Redis/Search Index | Truy vấn nội bộ |
| Auth → Redis | Session |
| Auth → Client | Cookie/JWT |

---
