# Cấu hình keepalive VRRP với nginx revered proxy
## Tổng quan về keepalive
Keepalived là một daemon chạy trên Linux, được dùng để:
- Tạo môi trường dịch vụ có tính sẵn sàng cao (High Availability – HA).
- Cân bằng tải (Load Balancing) cho các hệ thống web hoặc ứng dụng.
- Giám sát tình trạng (Health Checking) của các dịch vụ backend.
Gồm 2 thành phần chính <br>
**VRRP (Virtual Router Redundancy Protocol)**
- giúp tạo ra một địa chỉ IP ảo (VIP - Virtual IP) chia sẻ giữa các node trong cụm.
- Một node sẽ được bầu làm MASTER, node khác là BACKUP, nếu MASTER gặp sự cố BACKUP tiếp quản VIP -> dịch vụ không bị gián đoạn
**Health Checking**
- kiểm tra trạng thái của các backend server
## Cài đặt keepalived VRRP
- Cài đặt thêm một nginx revered proxy load balancing
- 2 server nginx proxy (thông mạng)
Cài đặt keepalive trên cả 2 server
```sh
sudo apt install keepalived nginx -y
sudo systemctl enable --now keepalived
```
### Cấu hình keepalive trên master
File cấu hình sẽ nằm trong /etc/keepalived/ <br>
Tạo file cấu hình
```sh
sudo nano /etc/keepalived/keepalived.conf
```
Thêm cấu hình 
```conf
vrrp_instance VI_1 {
    state MASTER                # Router hiện tại là Master
    interface ens160             # Giao diện mạng sử dụng (ens160)
    virtual_router_id 51        # ID của nhóm VRRP, phải giống nhau giữa các router trong nhóm
    priority 101                # Mức độ ưu tiên của router này (giá trị càng cao thì càng ưu tiên làm Master)
    advert_int 1                # Thời gian giữa các lần quảng bá gói tin VRRP (tính bằng giây)
    authentication {
        auth_type PASS          # Loại xác thực (PASS là mật khẩu)
        auth_pass 1234          # Mật khẩu xác thực
    }
    virtual_ipaddress {
        172.16.1.200/24           # Địa chỉ IP ảo được chia sẻ giữa các router
    }
}
```
Trong file cấu hình này
- state MASTER: Thiết lập router này là router chủ (Master). Nếu router này bị ngừng hoạt động, một router khác có thể trở thành Master.
- interface ens160: Chỉ định giao diện mạng mà VRRP sẽ chạy trên đó.
- virtual_router_id 51: Xác định ID của nhóm VRRP. Các router trong cùng một nhóm phải có ID này giống nhau.
- priority 101: Mức độ ưu tiên của router này. Router với mức độ ưu tiên cao sẽ trở thành Master.
- advert_int 1: Thời gian giữa mỗi lần gửi gói tin quảng bá VRRP tính bằng giây, tức là tần suất các router trong nhóm sẽ gửi gói tin VRRP để thông báo trạng thái của chúng.
- authentication: Thiết lập phương thức xác thực. Trong trường hợp này, PASS được sử dụng và mật khẩu xác thực là 1234.
- virtual_ipaddress: Địa chỉ IP ảo mà các máy chủ sẽ sử dụng làm gateway. IP này sẽ được chuyển giao giữa các router trong nhóm khi có sự cố.
Có thể kiểm tra bằng xem địa chỉ IP ảo đã được chuyển giao đến IP nào bằng
```sh
sudo ip show ens160
```
### Cấu hình keepalive trên Backup 
Thêm cấu mới vào trong file /etc/keepalived/keepalived.conf
```conf
vrrp_instance VI_1 {
    state BACKUP                # Router hiện tại là Master
    interface ens160             # Giao diện mạng sử dụng (eth0)
    virtual_router_id 51        # ID của nhóm VRRP, phải giống nhau giữa các router trong nhóm
    priority 100                # Mức độ ưu tiên của router này (giá trị càng cao thì càng ưu tiên làm Master)
    advert_int 1                # Thời gian giữa các lần quảng bá gói tin VRRP (tính bằng giây)
    authentication {
        auth_type PASS          # Loại xác thực (PASS là mật khẩu)
        auth_pass 1234          # Mật khẩu xác thực
    }
    virtual_ipaddress {
        172.16.1.200/24           # Địa chỉ IP ảo được chia sẻ giữa các router
    }
}
```
mặc định keepalive sẽ chạy với giao thức IP multicast , có thể cấu hình sử dụng unicast như sau:
```sh
unicast_peer {
        192.168.32.136  # Địa chỉ IP của router BACKUP
    }
```
- multicast là cơ chế truyền gói tin trong mạng từ một node đến nhiều node chỉ gửi cho một nhóm node đã đăng kí nhất định
- unicast sẽ gửi VRRP packet đến IP của gói còn lại dùng trong trường hợp multicast bị chặn bởi firewall, nếu có trên 2 node phải liệt kê tất cả các IP
#### Luồng hoạt động
Khi keepalive khởi chạy 
- Mỗi node (MASTER / BACKUP) sẽ đọc file cấu hình /etc/keepalived/keepalived.conf, và khởi tạo VRRP instance (vrrp_instance VI_1).
- Cả hai node đều tham gia cùng một nhóm VRRP "virtual_router_id"
- MASTER gửi gói tin VRRP multicast (hoặc unicast nếu cấu hình unicast_peer) advert_int mỗi giây.
- Node backup lắng nghe có nhận được vrrp packet mỗi giây không nếu nó nhận đều đặn → nó giữ nguyên trạng thái BACKUP, nếu không nó kích hoạt failover và chuyển sang trạng thái MASTER.
- Chuyển VIP: Khi MASTER hoạt động, nó gán VIP (Virtual IP) vào interface, khi BACKUP phát hiện MASTER chết, nó bind VIP này lên interface của mình và bắt đầu gửi gói VRRP mới, các client truy cập dịch vụ qua VIP không bị gián đoạn.
- Khi node MASTER cũ quay lại, nó sẽ: Gửi gói tin VRRP với priority cao hơn (ví dụ 101 > 90). Node hiện tại (BACKUP tạm thời) tự động nhường lại VIP. VIP chuyển ngược về node MASTER → quá trình “failback” hoàn tất. <br>
**Note**
- Failover: Đảm bảo dịch vụ luôn sẵn sàng bằng cách chuyển đổi giữa các máy chủ chính (master) và dự phòng (backup), Khi server chính ngừng hoạt động, Keepalived phát hiện lỗi, tự động chuyển địa chỉ IP ảo (VIP) sang server dự phòng -> dịch vụ hoạt động bình thường như chưa có sự cố 
- Failback: quá trình chuyển ngược lại vai trò MASTER cho máy chủ ban đầu sau khi nó đã khôi phục hoạt động bình thường. Tự động giành lại quyền MASTER (nếu bật preempt). Giữ nguyên trạng thái BACKUP (nếu tắt preempt bằng nopreempt).
