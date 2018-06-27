# IPTABLES 

1. [Tổng quan](#overview)
	* Iptables là gì? để làm gì?
	* Lịch sử phát triển
	* Các khái niệm

2. [Kiến trúc](#Arch)
	* Quá trình sử lý gói tin

3. [Iptables command](#ipcmd)
	* Các option command thông dụng

<a name="overview"></a>
## 1. Tổng quan

Iptables là firewall được cấu hình để lọc gói dữ liệu rất mạnh, miễn phí và có sẵn trên Linux. 

Tích hợp tốt với kernel của Linux. Có khả năng phân tích package hiệu quả. Lọc package dựa vào MAC và một số cờ hiệu trong TCP Header. Cung cấp chi tiết các tùy chọn để ghi nhận sự kiện hệ thống. Cung cấp kỹ thuật NAT. Có khả năng ngăn chặn một số cơ chế tấn công theo kiểu DoS.

Gồm 2 phần là Netfilter ở trong nhân linux và iptables nằm ngoài nhân. Iptables làm nhiệm vụ giao tiếp giữa người dùng và Netfilter để đẩy các luật từ người dùng vào cho netfilter xử lý. Netfilter làm việc trong kernel nhanh mà không ảnh hưởng đến tốc độ hệ thống.



<a name="Arch"></a>
## 2. Kiến trúc
### 2.1 Các khái niệm cơ bản
Iptables cơ bản gồm 3 thành phần chính: Table, chain, target

* Table 

Các table được sử dụng để định nghĩa các các rules cụ thể cho các gói tin. Có 4 loại table khác nhau: 

|Name| |
|----|--|
|Filter table|Quyết định gói tin có được gửi tới đích hay không|
|Mangle table|Bảng quyết định việc sửa head của gói tin (các giá trị của các trường TTL, MTU, Type of Service)|
|Nat table|Cho phép route các gói tin đến các host khác nhau trong mạng NAT, cách đổi IP nguồn IP đích của gói tin. |
|Raw table| Một gói tin có thể thuộc một kết nối mới hoặc cũng có thể là của 1 một kết nối đã tồn tại. Table raw cho phép bạn làm việc với gói tin trước khi kernel kiểm tra trạng thái gói tin.|

* Chain

Mỗi table được tạo ra mới một số chain nhất định, Chain cho phép lọc gói tin tại các "hook points" khác nhau. Netfilter định nghĩa ra 5 "hook points" trong quá trình xử lý gói tin của kernel: PREROUTING, INPUT, FORWARD, POSTROUTING, OUTPUT.

Các build-int chains được gán vào các hook point và có thể add một loạt các rules cho mỗi hook points.

Chains không chỉ nằm trên một table và một table không chỉ chứa một chain.

|Hook points| Cho phép xử lý các packet...|
|----|----|
|PREROUTING| vừa mới tiến vào từ network interface. Nó sẽ được thực thi trước khi quá trình routing diễn ra, thường dùng cho DNAT (destination NAT)|
|INPUT| có địa chỉ đích đến là server của bạn|
|FORWARD| có đích là một server khác nhưng không được tạo từ server của bạn. Chain này là cách cwo bản để cấu hình server của bạn để route các request tới một thiết bị khác.|
|OUTPUT| được tạo bởi server của bạn|
|POSTROUTING| đi ra ngoài hoặc được forward sau khi quá trình routing hoàn tất, chỉ trước khi nó tiến vào đường truyền, thường dùng cho SNAT (source NAT)|


* Target 

Target đơn giản là các hành động áp dụng cho các gói tin. Đối với những gói tin đúng theo rule mà chúng ta đặt ra thì các hành động (target) có thể thực hiện được đó là:
	- Accept: Chấp nhận gói tin
	- Drop: loại bỏ gói tin, không có gói tin trả lời, phía nguồn sẽ không biết đích có tồn tại hay không.
	- Reject: loại bỏ gói tin nhưng có trả lời table gói tin khác, ví dụ trả lời table 1 gói tin "connection reset" đối với gói TCP hoặc bản tin “destination host unreachable” đối với gói UDP và ICMP.
	- Log: chấp nhận gói tin và ghi log lại

Khi có nhiều rule, gói tin sẽ đi qua tất cả các rule chứ không chỉ dừng lại khi đã đúng với một rule đầu. Đối với gói tin, không khớp với rule nào thì mặc định sẽ được chấp nhận.

* Bảng NAT

Bảng NAT có 3 chain được xây dựng sẵn trong table NAT là 
	+ Chain PREROUTING: Ví dụ đối với gói tin tcp có port đích là 80 thì sẽ được đổi địa chỉ đích thành 10.0.30.100: `iptables -t NAT -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.30.100:80`
	+ Chain POSTROUTING: Ví dụ đổi địa chỉ nguồn với gói tin tcp thành 10.0.30.200: `iptables -t NAT -A POSTROUTING -p tcp -j SNAT --to-source 10.0.30.200`
	+ Chain Output

* Bảng filter

Các chain được xây dựng sẵn trong bảng filter:
	- Chain Input
	- Chain Output 
	- Chain Forward

* Bảng Mangle 

Bảng này bao gồm tất cả các chain được xây dựng sẵn (5 chain)

### 2.2 Quá trình xử lý gói tin


<a name="ipcmd"></a>
## 3. Iptables command

#### Để khởi động iptables mỗi khi máy khởi động

	chkconfig iptables on

#### Để xem các rule đang có trong iptables

	iptables -L -v

Trong đó: 

* TARGET: Hành động sẽ thực thi.
* PROT: Là viết tắt của chữ Protocol, nghĩa là giao thức. Tức là các giao thức sẽ được áp dụng để thực thi quy tắc này. Ở đây chúng ta có 3 lựa chọn là all, tcp hoặc udp. Các ứng dụng như SSH, FTP, sFTP,..đều sử dụng giao thức kiểu TCP.
* IN: chỉ ra rule sẽ áp dụng cho các gói tin đi vào từ interface nào, chẳng hạn như lo, eth0, eth1 hoặc any là áp dụng cho tất cả interface.
* OUT: Tương tự như IN, chỉ ra rule sẽ áp dụng cho các gói tin đi ra từ interface nào.
* DESTINATION: Địa chỉ của lượt truy cập được phép áp dụng quy tắc.

Ví dụ:

	ACCEPT    all    --   lo   any   anywhere   anywhere

Nghĩa là chấp nhận toàn bộ gói tin từ interface lo, lo ở đây nghĩa là “Loopback Interface“, chẳng hạn như IP 127.0.0.1 là kết nối qua thiết bị này.

	ACCEPT    all    --   any  any   anywhere   anywhere    ctstate  RELATED,ESTABLISHED

Chấp nhận toàn bộ gói tin của kết nối hiện tại. Nghĩa là khi bạn đang ở trong SSH và sửa đổi lại Firewall, nó sẽ không đá bạn ra khỏi SSH nếu bạn không thỏa mãn quy tắc

	ACCEPT    tcp    --   any  any   anywhere   anywhere    tcp      dpt:http

Cho phép kết nối vào cổng 80, mặc định sẽ biểu diễn thành chữ http.

	ACCEPT    tcp    --   any  any   anywhere   anywhere    tcp      dpt:https

Cho phép kết nối vào cổng 443, mặc định nó sẽ biểu diễn thành chữ https.

	DROP      all    --   any  any   anywhere   anywhere

Loại bỏ tất cả các gói tin nếu không khớp với các rule ở trên

#### Tạo một rule mới
	
	iptables -A INPUT -i lo -j ACCEPT

Trong đó:

* -A INPUT: khai báo kiểu kết nối sẽ được áp dụng (A nghĩa là Append).

* -i lo: Khai báo thiết bị mạng được áp dụng (i nghĩa là Interface).

* -j ACCEPT: khai báo hành động sẽ được áp dụng cho quy tắc này (j nghĩa là Jump).

gõ `iptables -L -v` bạn sẽ thấy xuất hiện một rule mới.

Sau khi thêm mới hoặc thay đổi bất cứ gì thì cần lưu và khởi động lại service

	service iptables save
	service iptables restart

Tiếp tục bây giờ chúng ta thêm một rule mới để cho phép lưu lại các kết nối hiện tại để tránh hiện tượng tự block bạn ra khỏi máy chủ.

	iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

Trong đó: 
* `-m conntrack`: Áp dụng cho các kết nối thuộc module tên là “Connection Tracking“. Module này sẽ có 4 kiểu kết nối là NEW, ESTABLISHED, RELATED và INVALID. Cụ thể là ở quy tắc này chúng ta sẽ sử dụng kiểu RELATED và ESTABLISHED để lọc các kết nối đang truy cập.
* `–ctstate RELATED,ESTABLISHED`: Khai báo loại kết nối được áp dụng của cái module Connection Tracking mà mình đã nói ở trên.


Ví dụ cho phép các cổng được truy cập từ bên ngoài vào qua giao thức tcp: SSH(22), HTTP(80), HTTPS(443)

	iptables -A INPUT -p tcp --dport 22 -j ACCEPT

Trong đó:

* `-p tcp`: Giao thức được áp dụng (tcp, udp, all)
* `--dport 22`: Cổng cho phép áp dụng. 22 là cho SSH

Cuối cùng chặn toàn bộ các kết nối truy cập từ bên ngoài vào nếu không thỏa mãn các rule trên. 

	iptables -A INPUT -j DROP

#### Thêm một rule mới

Nếu muốn chèn một rule mới vào một hàng nào đó, ví dụ hàng 2 thì thay tham số `-A` thành tham số `INSERT -I`

	iptables -I INPUT 2 -p tcp --dport 8080 -j ACCEPT

#### Xóa một rule

Để xóa 1 rule mà bạn đã tạo ra tại vị trí 4, ta sẽ sử dụng tham số -D

	iptables -D INPUT 4

Xóa toàn bộ các rule chứa hành động DROP có trong iptables:

	iptables -D INPUT -j DROP

Kiểm tra lại xem đã xóa thành công chưa

#### Chặn một ip

Nếu muốn chặn một ip 59.45.175.62 thì bạn cần thêm một rule mới vào INPUT chain của table filter, sử dụng lệnh sau:

	iptables -t filter -A INPUT -s 59.45.175.62 -j REJECT

Ngoài ra chúng ta cũng hoàn toàn có thể chặn cả dải địa chỉ IP với việc sử dụng CIDR.

	iptables -A INPUT -s 59.45.175.0/24 -j REJECT

Tương tự bạn có thể chặn traffic đi tới một IP hoặc 1 dải IP nào đó bằng cách sử dụng OUTPUT chain:

	iptables -A OUTPUT -d 31.13.78.35 -j DROP

### Một số các tùy chọn thường sử dụng

|Tùy chọn|	Ý nghĩa|s
|--|--|
|-A chain rule	|Thêm Rule vào chain|
|-D [chain] [index]	|Xóa rule có chỉ số trong chain đã chọn|
|-E [chain][new chain]|	đổi tên cho chain|
|-F [chain]	|Xóa tất cả các rule trong chain đã chọn, nếu ko chọn chain mặc định sẽ xóa hết rule trong tất cả các chain|
|-L [chain]	|Hiển thị danh sách tất cả các rule trong chain, nếu ko chọn chain thì mặc định nó sẽ hiện hết chain trong một table|
|-P [chain][target]	|Áp dụng chính sách đối với chain.|
|-Z [chain]|	Xóa bộ đếm của chain đi|
|-N [name new chain]|	Tạo một chain mới|
|-j [target]|	dùng để chỉ rõ gói tin sau khi thoải mãn rule sẽ được nhảy đến taret để xử lý|
|-m [match]	|dùng để mở rộng rule đối với với một gói tin (*)|
|-t [table]	|dùng để chọn bảng. nếu bạn không chọn thì mặc định iptable sẽ chọn bảng filter|
|-p [protocol]	|chỉ ra gói tin thuộc loại nào: tcp, udp, icmp,...|