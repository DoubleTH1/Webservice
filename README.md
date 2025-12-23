# Install LEMP-Stack-Ubuntu22.04-Wordpress
## Environments
- Nginx/1.18.0
- Mariadb-server
- PHP-FPM
- Wordpress
## Install nginx 
```sh
sudo apt install nginx
```
nginx -t kiểm tra syntax
<br>
nginx -T kiểm tra syntax nhưng in ra thứ tự mà nginx sẽ đọc các file config
#### note
Cấu hình Vitual hosts trên ubuntu nó sẽ nằm trong /etc/nginx/sites-available/ + /etc/nginx/sites-enabled/ và có cơ chế enable/disable site
<br>
**Kích hoạt site bằng**
```
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
```
**Hủy kích hoạt site**
```
sudo unlink /etc/nginx/sites-enabled/your_domain
```
<br>
Trên CentOS/RHEL sẽ không có thư mục sites-available / sites-enabled mặc định
Mỗi file sẽ là một file riêng trong /etc/nginx/conf.d/
<br>

## install mariadb-server 
```sh
sudo apt install mariadb-server -y
```
Truy cập vào db
```sh
sudo mysql -u root -p
```
Tạo db mới
```sh
CREATE DATABASE wordpress;
```
Tạo user
```sh
CREATE USE 'wordpress'@'localhost' IDENTIDIED BY 'password';
``` 
Phân quyền cho user
```sh
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'localhost';
```
localhost: chỉ cho phép truy cập db từ local
Xác nhận
```sh
FLUSH PRIVILEGES
```
Thoát khỏi db
```sh
exit
```
## install PHP-FPM
Cài đặt PHP-FPM
```sh
sudo apt install php-fpm
```
Cấu hình PHP-FPM pool
```sh
sudo nano /etc/php/8.1/fpm/pool.d/www.conf
```
Thêm vào cuối file
```conf
listen = 127.0.0.1:9000
listen.allowed_clients = 127.0.0.1
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
user = nginx
group = nginx
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
slowlog = /var/log/php-fpm/www-slow.log
php_admin_value[error_log] = /var/log/php-fpm/www-error.log
php_admin_flag[log_errors] = on
php_value[session.save_handler] = files
php_value[session.save_path] = /var/lib/php/session
security.limit_extensions = .php .php3 .php4 .php5 .php7
```
- listen.allowed_clients = 127.0.0.1: giới hạn client kết nối đến PHP-FPM qua TCP, nếu dùng unix socket thì dòng này bị bỏ qua
- slowlog = /var/log/php-fpm/www-slow.log: ghi lại các request PHP bị chậm
Cài đặt công cụ để truy vấn vào sql
```sh
sudo apt install php-curl php-gd php-intl php-mbstring php-soap php-xml php-xmlrpc php-zip  
```
Khởi động lại php-fpm
```sh
sudo systemctl restart php8.1-fpm
```
## Cài đặt wordpress
Truy cập vào thư mục chứa mã nguồn
```sh
cd /var/www/tth.com
```
Tải wordpress
```sh
sudo curl -LO https://wordpress.org/latest.tar.gz
```
Giải nén
```sh
sudo tar xzvf latest.tar.gz
sudo rm latest.tar.gz
```
Thiết lập wordpress
```sh
cd wordpress
sudo cp wp-config-sample.php wp-config.php
# Sửa file wp-config.php
sudo nano wp-config.php
```
Cập nhật các thông tin
```conf
define('DB_NAME', 'wordpress');
define('DB_USER', 'wordpress');
define('DB_PASSWORD', 'password');
define('DB_HOST', 'localhost');
```
Phân quyền cho thư mục wordpress
```sh
sudo chown -R www-data:www-data /var/www/tth.com/wordpress
sudo chmod 755 -R /var/www/tth.com/wordpress
```
## Cấu hình nginx 
Cài openssl để tạo chứng chỉ SSL 
```sh
sudo apt install openssl -y
```
Tạo thư mục lưu chứng chỉ và khóa
```sh
sudo mkdir /etc/nginx/ssl 
```
Tạo chứng chỉ SSL và khóa riêng self-signed
```
openssl req -x509 -nodes -newkey rsa:4096 -keyout /etc/nginx/ssl/tth-test2.com.key -out /etc/nginx/ssl/tth-test2.com.crt -days 365
```
-nodes cài mà không có passphrase
<br>

Cấu hình nginx truy cập được trong mạng LAN
```sh
sudo nano /etc/nginx/sites-available/tth-test.com
```
Thêm cấu hình sau
```conf
server {

    listen 80;  #lắng nghe yêu cầu trên port80 http

    server_name tth-test.com www.tth-test.com;

    root /var/www/tth-test.com;  #thư mục gốc chứa các tệp tĩnh như HTML, CSS, JavaScript, hình ảnh, video và các tài nguyên web khác.

    index index.html index.htm index.php; #thứ tự yêu tiên đọc file

    access_log /var/log/nginx/tth-test.com.access.log;

    error_log /var/log/nginx/tth-test.com.error.log;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }

    return 301 https://$host$request_uri; #Chuyển hướng toàn bộ HTTP sang HTTPS

    location / {                        #trong Nginx là một chỉ thị được sử dụng để xử lý tất cả các yêu cầu có đường dẫn bắt đầu bằng /.

        #try_files $uri $uri/ =404;  # kiểm tra và trả về tệp yêu cầu nếu có, nếu không thì trả về lỗi 404.
        try_files $uri $uri/ /index.php$is_args$args;
    }
}
server {
    listen 443 ssl;
    server_name tth-test.com www.tth-test.com;

    root /var/www/tth-test.com;
    index index.html index.htm index.php;

    ssl_certificate     /etc/nginx/ssl/tth-test.com.crt;
    ssl_certificate_key /etc/nginx/ssl/tth-test.com.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'HIGH:!aNULL:!MD5';
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/tth-test.com.access.log;
    error_log  /var/log/nginx/tth-test.com.error.log;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }
}
```
Kích hoạt site
```sh
sudo ln -s /etc/nginx/sites-available/tth-test.com /etc/nginx/sites-enabled/
```
Truy cập vào site theo địa chỉ IP server
