#HAProxy
##1.Giới thiệu
HAProxy viết tắt của High Available Proxy, là một phần mềm mã nguồn mở được dùng phổ biến cho cân bằng tải TCP/HTTP và giải pháp proxy. Nó có thể chạy trên Linux,Solaris và FreeBSD. HAProxy được sử dụng phổ biến nhất để cài thiện hiệu suất (performance) và độ tin cậy (realiability) của môi trường máy chủ bằng cách phân phối công việc trên nhiều máy chủ (ví dụ: web, ứng dụng ,database). Rất nhiều công ty lớn đang sử dụng nó như GitHub, Imgur, Instagram và Twitter.
Trong hướng dẫn này chúng tôi sẽ cung cấp một cái nhìn tổng quan về HAProxy, thuật ngữ cân bằng tải cơ bản và những ví dụ về làm thế nào HAProxy có thể cải thiệt hiệu xuất và độ tin cậy trong môi trường máy chủ của bạn.

##2. Các thuật ngữ trong HAProxy
Có nhiều thuật ngữ và khái niệm rất quan trọng khi thảo luận về cân bằng tải và proxy. 
Trước khi nói về các kiểu cơ bản của load balancing chúng ta sẽ nói về ACLs, backends và frontends

###Access Control List
Trong một mối quan hệ cân bằng tải, ACLs được sử dụng để kiểm tra một số điều kiện và thực hiện một hành động (ví dụ như chọn một máy chủ hoặc chặn một yêu cầu) dựa trên kết quả kiểm tra. Sử dụng ACLs cho phép mạng linh hoạt chuyển tiếp lưu lượng truy cập dựa trên nhiều yếu tố như khớp mẫu(matched) và số kết nối đến backend.

Ví dụ 1 một ACL

`acl url_blog_path_beg /blog`

ACL này được khớp (matched) nếu đường dẫn người dùng request bắt đầu với /blog. Ví dụ: http://domain.com/blog/blog-entry-1

Để cấu hình chi tiết ACL các bạn tham khảo tại đây
[HAProxy Configuration Manual](https://cbonte.github.io/haproxy-dconv/configuration-1.5.html#7)

###Backend

Backend là tập các server nhận các yêu cầu được forward từ server HAProxy. Backend được định nghĩa trong mục backend trong file cấu hình HAProxy. - -

Backend có thể định nghĩa bởi:

<ul>

<li>Thuật toán cân bằng được sử dụng</li>

<li>Một danh sách các server và port</li>

</ul>

Một Backend có thể chứa một hoặc nhiều server. Thêm nhiều server vào backend sẽ làm tăng khả năng chịu tải của ứng dụng mà bạn triển khai trên nó.

Dưới đây là một ví dụ về hai cấu hình backend gồm có web-backend và blog-backend với 2 web server trên mỗi backend lắng nghe trên cổng 80

```
backend web-server
       mode http
       balance roundrobin
       server web-1 web-1.domain.com:80 check
       server web-2 web-2.domain.com:80 check
backend blog-server
       mode http 
       balance source 
       server blog-1 blog-1.domain.com:80 check
       server blog-2 blog-2.domain.com:80 check

```

- `balance roundrobin` và `balance source ` dòng này định nghĩa thuật toán cân bằng tải được sử dụng

- `mode http` định nghĩa cân bằng tải trên layer 7

- option `check` ở cuối của `server` dùng để kiểm tra tình trạng (sức khỏe) của các server (down hay up)

###Frontend

Một Frontend được định nghĩa làm thế nào để request có thể forward tới backend. Frontend được định nghĩa trong mục frontend trong file config HAProxy. 
- Các định nghĩa gồm các thành phần sau:
<ul>

<li>Địa chỉ IP và port (ví dụ *:80, 10.14.55.28:443,..)</li>

<li>ACLs</li>

<li>*use_backend* định nghĩa điều kiện ACLs trên backend nào</li>

<li>*default_backend* xử lý mọi trường hợp khác</li>

</ul>

Một Backend có thể được cấu hình theo nhiều kiểu traffic network (sẽ được giải thích trong phần sau)

##3.Các kiểu cân bằng tải

###Không có load balancing

Một ứng dụng web không có load balancing sẽ giống như sau

<img src="http://i.imgur.com/cad1GDu.png">

Người dùng kết nối trực tiếp đến web server của bạn thông qua tên miền. Nếu server down người dùng không thể truy cập vào web server.

###Layer 4 Loadbalancing

Cách đơn giản nhất để chuyển traffic tới các server là sử dụng layer (transport layer). Load balancing theo cách này sẽ forward user traffic dựa trên IP và Port đến backend.

Dưới đây là mô hình cho layer 4 load balancing

<img src="http://i.imgur.com/BSmJy1f.png">

Người dùng truy cập vào máy chủ load balancer, request sẽ được forward tới backend. Tất cả các server trong backend phải có nội dung giống nhau nếu không người dùng có thể nhận được những nội dung không phù hợp. Lưu ý: cả 2 server đều kết nối tới cùng 1 database

###Layer 7 Load Balancing

Một cách phức tạp hơn là cân bằng tải traffic sử dụng tại layer 7 (application layer). Sử dụng layer 7 cho phép server load balancer forward request tới các backend khác nhau dựa trên nội dung người dùng request. Đây là kiểu load balancing cho phép ban chạy nhiều ứng dụng web trên cùng 1 domain và port. 

Dưới đây là mô hình load balancing cho layer 7:

<img src="http://i.imgur.com/zxhxTCA.png">

Ví dụ: Nếu người dùng request tới domain.com/blog các request sẽ được forward tới blog-backend (gồm tập các server chạy ứng dung blog). Mặt khác request sẽ được forward tới web-backend (chạy ứng dụng khác). Cả 2 backend sử  dụng chung một database server

Dưới đây là một đoạn cấu hình trong file cấu hình HAProxy cho ví dụ trên

```
frontend http-layer7
         bind *:80
         mode http

         acl  url_blog path_beg /blog
         use_backend blog-backend if url_blog

         default_backend web-backend

```

- Frontend có tên http-layer7 xử lý traffic đến trên cổng 80

- `acl url_blog path_beg /blog` khới với một request nếu đường dẫn người dùng request bắt đầu với /blog

- `use_backend blog-backend if url_blog` sử dụng ACL để forward traffic tới blog-backend

- `default_backend web-backend` quy định tất cả traffic khác sẽ được forward web-backend


##4.Thuật toán Load Balancing


Thuật toán cân bằng tải được sử dụng để xác định server trong một backend sẽ được chọn khi load balancing. HAProxy cung cấp một số tùy chọn cho các thuật toán. Ngoài các thuật toán cân bằng tải máy chủ có thể được chỉ định một tham số weight để thao tác thường xuyên như thế nào khi server đó được chọn.

Các thuật toán thường được sử dụng:

###roundrobin

Round robin chọn các máy chủ lần lượt. Đây là thuật toán mặc định

###leasteconn

Chọn máy chủ với số lượng connection ít nhất - recommended cho các ứng dụng có session lâu. Các máy chủ trong backend cũng được luân chuyển trong kiểu round-robin

###source

Việc lựa chọn server dựa trên hash IP nguồn (IP người dùng). Phương pháp này để chắc chắn 1 user sẽ kết nối đến cùng một server

##5.Kết luận
 
Bây giờ thì bạn đã có những hiểu biết cơ bản về load balancing và biết được cách mà HAProxy làm cho load balancing dễ dàng. Bạn đã có nền tảng vững chắc để bắt đầu vào việc cải thiện hiệu suất và độ tin cậy của môi trường các máy chủ của bạn

**Các bài viết sau mình sẽ hường dẫn cấu hình trên layer 4 và layer 7 và HAProxy cho MySQL Loadbalancing** 

