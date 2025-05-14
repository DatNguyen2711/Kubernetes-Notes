# Hành Trình Của Một Packet trong VXLAN

Mục tiêu của VXLAN là tạo ra ảo giác rằng tất cả các Pod đều nằm trên cùng một mạng phẳng lớn, ngay cả khi chúng phân tán trên nhiều node vật lý khác nhau. Nó làm điều này bằng cách "đóng gói" (encapsulate) packet gốc của Pod bên trong một packet UDP/IP bên ngoài. Packet UDP/IP này sau đó được định tuyến qua mạng vật lý giữa các node.

![alt text](image.png)

## Giả định cấu hình VXLAN điển hình:

- Mỗi Pod có một cặp giao diện mạng ảo **veth pair**. Một đầu của **veth pair** nằm trong network namespace của Pod, đầu còn lại nằm trong network namespace gốc của Node.
- Trên mỗi Node, đầu **veth** của Pod nằm trong namespace gốc được gắn vào một **Linux bridge** ảo (ví dụ: `cbr0`, `docker0`, hoặc bridge do CNI tạo). Bridge này hoạt động như một switch ảo, kết nối tất cả các Pod trên cùng Node.
- Trên mỗi Node cũng có một giao diện **VXLAN** (ví dụ: `flannel.1`, `weave`, `vxlan0`). Giao diện này chịu trách nhiệm đóng gói và giải đóng gói packet VXLAN.
- Hệ thống định tuyến trên mỗi Node được cấu hình để biết rằng Pod CIDR của các Node khác có thể truy cập được thông qua giao diện VXLAN của Node đó.

---

## Đường đi chi tiết của Packet (Mode VXLAN):

### 1. Packet rời Pod A (Node 1):

- Pod A muốn gửi packet đến Pod B (địa chỉ IP ví dụ: `10.244.2.5`).

```bash 
$ kubectl exec -it ubuntu-pod3  -- ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
44: eth0@if45: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 36:35:53:59:22:f0 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.2.1/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::3435:53ff:fe59:22f0/64 scope link 
       valid_lft forever preferred_lft forever

# eth0 chính là card mạng chính của pod được nối với 1 đầu veth trong Node       
```       

- **Packet được tạo ra với**:
  - **Source IP**: IP của Pod A (ví dụ: `10.244.1.3`).
  - **Destination IP**: IP của Pod B (`10.244.2.5`).
  - **Source MAC**: MAC của giao diện `eth0` (hoặc tên tương tự) trong Pod A.
  - **Destination MAC**: MAC của gateway mặc định của Pod A, chính là đầu **veth** của nó nằm trong namespace gốc của Node 1.
- Packet rời khỏi **network namespace** của Pod A thông qua đầu **veth** bên trong Pod.

### 2. Packet đi vào Node 1 (qua veth pair và Bridge):

- Packet xuất hiện ở đầu **veth** còn lại của Pod A nằm trong **network namespace** gốc của Node 1.
- Đầu **veth** này được gắn vào **Linux bridge** (ví dụ: `cbr0`).

> Lý do để biết tại sao VETH gắn vào Linux bridge, khi check thông tin card veth thì chúng ta thấy có đoạn 10: vetheaa97948@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master **cni0** state UP mode DEFAULT group default

```bash 

# Inspect the network configuration on the minikube host
$ ip link sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:37:09:5c brd ff:ff:ff:ff:ff:ff
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:ea:1b:4a brd ff:ff:ff:ff:ff:ff
4: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:7e:1b:3c:46 brd ff:ff:ff:ff:ff:ff
6: cni-podman0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether aa:8b:b8:3b:ae:0d brd ff:ff:ff:ff:ff:ff
8: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether 3e:7d:76:2a:39:82 brd ff:ff:ff:ff:ff:ff
9: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether c6:ae:d0:e6:66:cc brd ff:ff:ff:ff:ff:ff
    
$ ethtool -S vetheaa97948
NIC statistics:
     peer_ifindex: 4
10: vetheaa97948@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP mode DEFAULT group default 
    link/ether 0e:52:e0:c4:14:74 brd ff:ff:ff:ff:ff:ff link-netnsid 0

```
- Bridge tra cứu bảng MAC của nó để tìm MAC address của địa chỉ IP đích (`10.244.2.5`). Tuy nhiên, vì `10.244.2.5` nằm ngoài bridge này, bridge sẽ gửi packet lên lớp mạng cao hơn để xử lý định tuyến.

### 3. Routing trên Node 1:

- Hệ điều hành Node 1 nhận packet từ bridge.
- Hệ điều hành tra cứu bảng định tuyến của Node 1.
- Bảng định tuyến của Node 1 có một route cho dải IP của Pod B (`10.244.2.0/24`). Route này chỉ ra rằng để tới dải IP này, packet cần được gửi qua giao diện **VXLAN** của Node 1 (ví dụ: `flannel.1`).

> Check bằng lệnh **ip route sh** để thấy routing từ pod đến card mạng của CNI (chạy lệnh trên node mà chứa pod cần check), ví dụ khi chạy lệnh thì output sẽ ra như này:


```bash 
$ ip route sh
default via 192.168.122.1 dev eth1 proto dhcp src 192.168.122.215 metric 1024 
10.88.0.0/16 dev cni-podman0 proto kernel scope link src 10.88.0.1 linkdown 
10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1 
10.244.2.0/24 via 10.244.1.0 dev flannel.1 onlink 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
192.168.61.0/24 dev eth0 proto kernel scope link src 192.168.61.95 
192.168.122.0/24 dev eth1 proto kernel scope link src 192.168.122.215 
192.168.122.1 dev eth1 proto dhcp scope link src 192.168.122.215 metric 1024
```

### 4. Đóng gói VXLAN (Encapsulation) trên Node 1:

- Packet gốc (`Source IP: 10.244.1.3, Dest IP: 10.244.2.5`) được chuyển đến giao diện **VXLAN** (`flannel.1`).
- Giao diện VXLAN biết rằng Pod CIDR `10.244.2.0/24` thuộc về Node 2 (địa chỉ IP vật lý ví dụ: `192.168.100.102`).

> Lý do mà VXLAN có thể biết rằng CIDR `10.244.2.0/24` thuộc về Node 2 (`192.168.100.102`) là do khi CNI sử dụng mode VXLAN thì các CIDR của Pod sẽ có 1 tunnel được nối đến các worker-node. Chạy lệnh **cilium bpf tunnel list** ở trong 1 Pod cilium để có thông tin **Trong trường hợp dùng cilium**

```golang 
$ kubectl exec -n kube-system cilium-4bk46 -- cilium bpf tunnel list
                                                                                
> Here is the output:                                                             
                                                                                
  TUNNEL     VALUE                                                              
  10.0.4.0   192.168.10.168:0                                                   
  10.0.3.0   192.168.10.169:0                                                   
  10.0.1.0   192.168.10.204:0                                                   
  10.0.0.0   192.168.10.162:0     
```

- Giao diện VXLAN đóng gói packet gốc bằng cách thêm các header mới:
  - **Inner Header**: Packet gốc của Pod (IP header, TCP/UDP header, data).
  - **VXLAN Header**: Thêm một VXLAN header nhỏ, chứa **VNI** (VXLAN Network Identifier) để phân biệt các mạng overlay khác nhau nếu có.
  - **Outer UDP Header**: Thêm một UDP header với port đích là 8472 (port của VXLAN).
  - **Outer IP Header**:
    - **Source IP**: IP vật lý của Node 1 (ví dụ: `192.168.100.101`).
    - **Destination IP**: IP vật lý của Node 2 (ví dụ: `192.168.100.102`).
  - **Outer Ethernet Header**: Thêm một Ethernet header mới với MAC đích là MAC của router/gateway tiếp theo trên mạng vật lý để đến Node 2.
- Lúc này, packet trông giống như một packet UDP thông thường đi từ IP vật lý của Node 1 đến IP vật lý của Node 2.

### 5. Packet đi qua Mạng Vật lý:

- Packet đã được đóng gói VXLAN được gửi ra card mạng vật lý của Node 1 (ví dụ: `ens133`).
- Packet được định tuyến qua mạng vật lý, sử dụng các router và switch thông thường, giống như bất kỳ packet IP nào khác đi giữa Node 1 và Node 2.
- Packet cuối cùng đến card mạng vật lý của Node 2.

### 6. Packet đi vào Node 2 (qua card mạng vật lý và Giao diện VXLAN):

- Card mạng vật lý của Node 2 nhận packet UDP có đích là IP vật lý của Node 2 và port 8472.
- Kernel của Node 2 nhận diện packet này dành cho giao diện **VXLAN** (`flannel.1`) vì nó đến trên port 8472.
- Packet được chuyển đến giao diện VXLAN.

### 7. Giải đóng gói VXLAN (Decapsulation) trên Node 2:

- Giao diện VXLAN trên Node 2 nhận packet đóng gói.
- Nó kiểm tra VXLAN header và ID mạng (VNI).
- Nó bóc tách (remove) **Outer Ethernet**, **Outer IP**, **Outer UDP** và **VXLAN headers**.
- Packet gốc của Pod (với `Source IP: 10.244.1.3, Dest IP: 10.244.2.5`) được phục hồi.

### 8. Routing và đi vào Pod B (Node 2):

- Packet gốc đã được giải đóng gói được chuyển đến stack mạng của Node 2.
- Kernel của Node 2 tra cứu bảng định tuyến cho packet có đích là `10.244.2.5`.
- Bảng định tuyến của Node 2 cho thấy địa chỉ `10.244.2.5` nằm trong dải Pod CIDR của chính Node 2 (`10.244.2.0/24`).
- Packet được chuyển đến **Linux bridge** (`cbr0` hoặc tương tự) mà tất cả các Pod trên Node 2 được gắn vào.
- Bridge tra cứu bảng MAC của nó và tìm thấy rằng địa chỉ MAC tương ứng với IP `10.244.2.5` có thể truy cập được thông qua đầu **veth** của Pod B gắn vào bridge.
- Bridge đẩy packet ra đầu **veth** này.

### 9. Packet đi vào Pod B (Node 2):

- Packet đi qua **veth pair** và đi vào **network namespace** của Pod B.
- Pod B nhận được packet gốc từ Pod A.

---

![alt text](image-1.png)

**Flannel**

```bash 
Pod -> veth -> bridge (cni0) -> VXLAN (flannel.1) -> eth0

Dùng IPAM, routing, bridging truyền thống.
```

> Đối với Cilium dùng VXLAN mode, Routing sẽ được xử lý ở eBPF sâu bên trong Kernel. Về cơ bản packet vẫn sẽ đi từ card eth0 của pod -> card LXC.... (nằm trên mỗi node mà pod chạy) -> xử lý routing = eBPF (đoạn này card cilium_vxlan sẽ xử lý) đẩy gói tin lên card mạng eth0 hoặc ens... của node -> traffic đi dần dần vào Node khác rồi decapselated packet chuyển dần vào pod 2

**Khác so với calico, flannel hay canal sẽ routing từ VETH đến 1 CNI bridge network thì cilium sẽ xử lý eBPF thay vì chuyển packet qua nhiều lớp mạng**

**Cilium**

```c++ 
pod -> veth -> eBPF:from-container -> cilium-vxlan -> eth0

eBPF xử lý từng bước:

    from-container

    to-overlay

    to-netdev

=> Không cần routing table / iptables / bridges.
```
![alt text](image-3.png)

# So sánh Cilium vs Calico vs Flannel (Về Routing & Xử lý Packet)

## 1. Kiến trúc xử lý Packet (Data Path)

| Thành phần             | Flannel/Calico (veth/bridge/iproute)                        | Cilium (eBPF-based)                                          |
|------------------------|-------------------------------------------------------------|--------------------------------------------------------------|
| Packet flow            | Pod → veth → bridge (cni0) → routing table → VXLAN          | Pod → veth → eBPF (from-container) → overlay/netdev          |
| Routing                | Qua Linux routing table + iptables                          | Dùng eBPF map lookup trực tiếp trong kernel                  |
| Overlay encapsulation  | Thực hiện trong user/kernel space, thường dùng VXLAN        | Có thể dùng VXLAN/Geneve hoặc native routing (no overlay)    |
| Cơ chế chính           | Linux Bridge, iptables, conntrack                           | eBPF programs gắn trực tiếp lên interface                    |

---

## 2. Debug và Visibility

| Thành phần         | Flannel/Calico                                      | Cilium                                                        | 
|--------------------|-----------------------------------------------------|---------------------------------------------------------------|
| Quan sát lưu lượng | Phải xem `iptables`, `conntrack`, `ip route`, etc.  | Dùng `cilium monitor`, `cilium bpf`, `cilium policy trace`    |
| Debug rule policy  | Iptables rule hoặc Calico policy log                | BPF maps theo từng policy, kiểm soát đến từng packet          |
| Tracing            | Gặp giới hạn của netfilter, khó trace chi tiết      | Có thể trace từng bước trong eBPF pipeline dễ dàng            |

---

## 3. Hiệu năng & Mở rộng (Performance & Scalability)

| Thành phần             | Flannel/Calico                                 | Cilium                                                            |
|------------------------|------------------------------------------------|-------------------------------------------------------------------|
| Overhead xử lý         | Context switch nhiều giữa user/kernel space    | Xử lý inline trong kernel bằng eBPF, gần như không context switch |
| NAT                    | Thường xuyên dùng SNAT                         | Có thể tắt NAT, dùng routing gốc                                  |
| Khả năng mở rộng policy| Giới hạn khi rules lớn                         | eBPF maps có thể scale tới hàng ngàn rule mà vẫn ổn định          |
| Tốc độ truyền gói tin  | Trung bình                                     | Rất cao (do bypass iptables + bridge + routing table)             |

---

## 4. Tổng kết Flow

### Flannel (hoặc Calico)

```bash 
Pod -> veth -> cni0 bridge -> VXLAN (flannel.1) -> eth0 -> physical net
```
### Cilium

```bash 
Pod -> veth -> eBPF:from-container -> eBPF:to-overlay -> cilium-vxlan -> eth0
```




# Tóm tắt:

- Packet từ Pod A đến Pod B (khác node) được đóng gói bên trong một packet UDP/IP giữa hai Node vật lý.
- Node nguồn thực hiện đóng gói (encapsulation) VXLAN dựa trên bảng định tuyến Pod CIDR.
- Mạng vật lý chỉ định tuyến packet UDP/IP giữa các Node. Nó không quan tâm đến IP gốc của Pod.
- Node đích thực hiện giải đóng gói (decapsulation) VXLAN.
- Packet gốc được trả lại stack mạng của Node đích và sau đó được định tuyến cục bộ đến Pod đích thông qua bridge và veth pair.

Chế độ VXLAN cho phép tạo ra một mạng overlay đơn giản, không yêu cầu cấu hình định tuyến phức tạp trên mạng vật lý cho từng Pod CIDR. Mạng vật lý chỉ cần có khả năng định tuyến giữa các Node Kubernetes.



# Sources

[Kubernetes with CNI cilium deep dive](https://addozhang.medium.com/kubernetes-network-learning-with-cilium-and-ebpf-aafbf3163840)

[How pod communcate with pod in another node](https://www.redhat.com/en/blog/kubernetes-pods-communicate-nodes)

[kubernetes vxlan network with flannel](https://addozhang.medium.com/learning-kubernetes-vxlan-network-with-flannel-2d6a58c95300)