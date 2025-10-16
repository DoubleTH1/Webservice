# Cài đặt nginx revered proxy
## Tổng quan 
Reverse Proxy là một server trung gian giữa Client và Server. Nó kiểm soát các request từ Client, và điều phối những request đó tới Server phù hợp, để Server xử lí request đó. Khi Server xử lí xong, sẽ trả về response cho Reverse Proxy, và Reverse Proxy có trả về response đó cho Client.
## Cài wordpress
Cài thêm một backend wordpress dùng chung database
Trên server này tiến hành cài đặt nginx, PHP-FPM và wordpress
Trên server có mariadb tiến hành mở port 3306 
```sh
sudo ufw allow 3306/TCP
```
Truy cập vào db 
```sh
sudo mysql -u root -p
```
Tạo user wordpress cho phép kết nối từ mọi IP (% là willcard)
```sh
CREATE USER 'wordpress'@'%' IDENTIFIED BY '123456';
```
Phân quyền user
```sh
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'%';
```
Xác nhận
```sh
FLUSH PRIVILEGS;
```
Trên server wordpress mới cập nhật thông tin trong file /var/www/wp-config.php
```conf
define('DB_NAME', 'wordpress');
define('DB_USER', 'wordpress');
define('DB_PASSWORD', 'password');
define('DB_HOST', 'localhost');
```
Phần quyền cho thư mục wordpress
```sh
sudo chown -R www-data:www-data /var/www/tth.com/wordpress
sudo chmod 755 -R /var/www/tth.com/wordpress
```
## Cài đặt nginx revered proxy 
Cấu hình nginx proxy trong /etc/nginx/conf.d/
```sh
sudo nano /etc/nginx/conf.d/proxy.conf
```
Thêm cấu hình sau
```conf
upstream backend_servers {    #định ngĩa một nhóm server backend
    server 172.16.1.232:443;
    server 172.16.1.231:443;
}

server {
        listen 80;
        server_name proxytest.com;
        return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name proxytest.com;

    # Đường dẫn đến chứng chỉ và khóa riêng
    ssl_certificate /etc/nginx/ssl/nginx-proxy.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-proxy.key;

    #Cấu hình SSL
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'HIGH:!aNULL:!MD5';    #Lựa chọn thuật toán mã hóa loại bỏ các bản mã(cipher) không xác thực danh tính và loại bỏ các cipher sử dụng thuật toán MD5 hash
    ssl_prefer_server_ciphers on;      #Đảm bảo chỉ dùng cipher mạnh an toàn nếu off giảm bảo mật của kết nối

    access_log /var/log/nginx/proxyaccess.log upstreamlog;               #Upstreamlog định dạng log custom tự định nghĩa bằng log_format

    location / {
        # Chuyển tiếp yêu cầu của client tới nhóm máy chủ backend/chỉ thị chuyển tiếp response từ client đến server backend
        proxy_pass https://backend_servers;

        # Thiết lập các header cần thiết cho reverse proxy
        proxy_set_header Host $host;                                     # Truyền thông tin domain tới backend
        proxy_set_header X-Real-IP $remote_addr;                         # Truyền địa chỉ IP của client
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;     # Truyền thông tin của các proxy trung gian
        proxy_set_header X-Forwarded-Proto $scheme;                      # Truyền giao thức (HTTP hoặc HTTPS) tới backend
        }
}
```
Truy cập qua ip của proxy nginx nginx sẽ định tuyến tới một trong hai backend
Thêm cấu hình sau để xem log các server đã load balandcing chưa bằng cách định nghĩa log_format trong file cấu hình /etc/nginx/nginx.conf
```conf
log_format upstreamlog
            '$remote_addr - $remote_user [$time_local] '
            '"$request" $status $body_bytes_sent '
            '"$http_referer" "$http_user_agent" '
            'to: $upstream_addr';
```
- $remote_addr: Địa chỉ IP của client request tới
- $remote_user: Tên user nếu có dùng basic auth
- $time_local: Thời gian Nginx nhận request, theo múi giờ local của server
- $request: Dòng request HTTP đầu tiên (phương thức + URL + HTTP version).
- $status: Mã trạng thái HTTP trả về cho client
- $body_bytes_sent: số bytes mà được gửi đến cho client, không tính header
- $http_referer: Client có được chuyển hướng từ web trước đó không
- $http_user_agent: thông tin trình duyệt của client
- $upstream_addr: IP máy chủ backend mà nginx proxy request đến
