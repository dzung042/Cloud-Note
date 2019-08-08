Thành phần cấu tạo của OpenStack
OpenStack không phải dự án nhỏ và đơn lẻ mà chính là nhiều dự án mã nguồn mở nhằm mục tiêu cung cấp các dịch vụ cloud hoàn chỉnh. Open Stack có rất nhiều thành phần:

OpenStack Compute
Là những module quản lý và tạo ra máy ảo. Tên phát hành của nó Nova. Nó support nhiều hypervisors gồm ảo hóa LXC, KVM, XenServer, QEMU,… Compute là một tools rất mạnh giúp điều khiển toàn bộ các công việc:  mạng (networking), chip xử lý (CPU), tài nguyên đĩa (storage), Bộ nhớ đệm(memory), create, điều khiển và delete máy ảo, Bảo mật (security), access control. Người quản lý có thể điều khiển hầu hết bằng lệnh hoặc từ giao diện người dùng trên web.

OpenStack Glance
Nó là OpenStack Image Service, dis manager images. Glance hỗ trợ các ảnh Hyper-V (VHD), Raw, VirtualBox (VDI), VMWare (VMDK, OVF) và Qemu (qcow2). Quản trị có thể thực hiện: update thêm các virtual disk images, cài đặt các private image và public điều khiển việc truy cập vào chúng, và dễ dàng tạo và xóa chúng.

OpenStack Object Storage
Chuyên dùng để quản lý lưu trữ dữ liệu. Đây là một hệ thống lưu dữ phân tán cho phép quản lý tất cả các dạng của lưu trữ như: archives, virtual machine image, user data, … Có nhiều class redundancy và copy phiên bản được thực hiện tự động, khi đó có node bị lỗi thì cũng không làm mất dữ liệu toàn bộ, và việc khôi phục được thực hiện tự động.

Identity Server
nhiệm vụ để quản lý xác minh cho user và projects.

OpenStack Netwok
Đây là thành phần quản lý mạng cho các máy chủ ảo. Cung cấp các chức năng network as a service. Đây là hệ thống có các tính chất pluggable, API-driven và scalable.

OpenStack dashboard
Mang đến giao diện trực quan cho người quản trị cũng như người dùng đồ họa dễ truy cập thao tác, cung cấp và tự động tạo tài nguyên theo thao tác. Việc giao diện có thể mở rộng giúp dễ dàng thêm các chức năng ngoài, giúp việc tái cấu trúc nhanh chóng.