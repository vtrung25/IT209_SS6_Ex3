# Bài 3 — Cấu hình UFW và chẩn đoán cổng mạng

## Mục tiêu cấu hình

- Mặc định chặn kết nối vào và cho phép kết nối đi ra.
- Cho phép SSH trên `22/tcp` và ứng dụng web trên `8080/tcp`.
- Bật UFW, kiểm tra chính sách và xác nhận dịch vụ có thực sự lắng nghe trên cổng dự kiến.

## Thực hiện trên Ubuntu Server

Trước khi bật tường lửa, thêm luật SSH để giữ kết nối quản trị. Thực hiện các lệnh sau với quyền `sudo`:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp
sudo ufw enable
sudo ufw status verbose
```

Trả lời `y` khi UFW hỏi xác nhận bật. Trạng thái cần có `Status: active`, chính sách `deny (incoming)` và `allow (outgoing)`, cùng luật cho `22/tcp` và `8080/tcp` là `ALLOW IN`.

## Chẩn đoán cổng dịch vụ

```bash
sudo ss -tlnp
sudo ss -tlnp | grep -E ':(22|8080)\b'
curl -I --max-time 5 http://127.0.0.1:8080/
```

`ss` kiểm tra socket TCP đang ở trạng thái listen; `curl` gửi HTTP request tới ứng dụng trên cùng máy. Nếu cổng 8080 chưa xuất hiện trong `ss` hoặc `curl` báo connection refused, kiểm tra ứng dụng đã khởi động và bind đúng địa chỉ/cổng chưa. Luật UFW cho phép lưu lượng qua firewall nhưng không tự khởi chạy ứng dụng.

## Xác minh từ máy bên ngoài

Sau khi xác nhận firewall của nhà cung cấp cloud cũng cho phép TCP 22 và 8080, truy cập `http://<IP_MAY_CHU>:8080/` hoặc chạy:

```bash
curl -I --max-time 5 http://<IP_MAY_CHU>:8080/
```

Đầu ra mong đợi của `ufw status verbose` là trạng thái active và các luật `22/tcp ALLOW IN` cùng `8080/tcp ALLOW IN`. Kết quả của `ss` phụ thuộc vào tiến trình thực tế đang chạy trên máy chủ; `curl` thành công khi ứng dụng HTTP hoạt động.

## Kết quả kiểm tra trên VPS

UFW đã active với mặc định `deny (incoming)` và `allow (outgoing)`. Máy chủ có sẵn luật SSH `22/tcp` và HTTP `80/tcp`; luật 80 được giữ nguyên để không làm gián đoạn Nginx có sẵn. Đã thêm `8080/tcp ALLOW IN` cho IPv4 và IPv6.

Ban đầu `ss -tlnp` chỉ hiển thị SSH và Nginx, không có listener trên 8080. Khi chạy một HTTP server tạm thời, `ss` xác nhận listener `0.0.0.0:8080` và `curl -I http://127.0.0.1:8080/` trả về `HTTP/1.0 200 OK`; server kiểm tra đã được dừng sau đó. Request từ máy cá nhân tới địa chỉ công khai `:8080` timeout, nên cần kiểm tra inbound rule 8080 ở Cloud Firewall/NSG của nhà cung cấp để xác nhận đường truy cập ngoài Internet.
