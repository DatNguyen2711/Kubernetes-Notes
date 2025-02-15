# K8S networking Deepdive


1️⃣ Tổng quan về NAT, DNAT, SNAT và các khái niệm liên quan

🔹 NAT (Network Address Translation)

Là kỹ thuật thay đổi địa chỉ IP trong gói tin khi chúng đi qua một thiết bị mạng, giúp thay đổi địa chỉ nguồn hoặc đích của gói tin.

🔹 DNAT (Destination NAT)

Là loại NAT thay đổi địa chỉ đích của gói tin. Thường được dùng để chuyển tiếp lưu lượng đến các máy chủ nội bộ.

Ví dụ: Khi một yêu cầu đến Service trong Kubernetes, DNAT sẽ chuyển hướng yêu cầu đến một trong các Pod backend.

🔹 SNAT (Source NAT)

Là loại NAT thay đổi địa chỉ nguồn của gói tin. Thường được dùng khi gói tin rời khỏi mạng nội bộ.

Ví dụ: Khi một Pod gửi yêu cầu ra ngoài internet, SNAT thay đổi địa chỉ nguồn thành IP của node để gói tin có thể quay lại.

🔹 MASQUERADE

Là một dạng SNAT tự động thay đổi địa chỉ IP nguồn của gói tin khi rời khỏi mạng, thường được dùng trong các mạng động như Kubernetes.

2️⃣ Kiểu giao tiếp trong Kubernetes

🔹 Pod-to-Pod

Nếu các Pod nằm trong cùng một cụm (cluster), chúng giao tiếp trực tiếp qua địa chỉ IP.Pod to Service: 

🔹 Pod-to-Service

Khi tạo một Service trỏ vào các Pod (qua labels), Kubernetes tạo một Endpoint chứa IP của các Pod mà Service trỏ đến:

```bash
root@datnd-master-node:~# kg ep -n myapp
NAME                ENDPOINTS                         AGE
back-end-service    10.0.1.122:8080,10.0.2.209:8080   47h       # Đây là địa chỉ IP của các POD, tên của endpoints sẽ giống tên service
front-end-service   10.0.1.111:80,10.0.2.112:80       47h
sqlserver           10.0.1.218:1433                   47h
root@datnd-master-node:~# kgsvc -n myapp
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
back-end-service    ClusterIP   10.108.197.131   <none>        8080/TCP   47h
front-end-service   ClusterIP   10.107.126.93    <none>        80/TCP     47h
sqlserver           ClusterIP   10.102.152.25    <none>        1433/TCP   47h

```
2. Tiếp đó là Kube-proxy sẽ tạo ra các quy tắc (CHAIN) trong iptables:

"-t nat" có nghĩa là loại IPTable mà chúng ta muốn list ra đối với kube-Proxy nó sử dụng NAT table, ở chế độ iptables thì Kube-Proxy dựa
vào NAT table để dịch địa chỉ IP của services trong cụm
Chain: PREROUTING chain này mặc định có trong iptables, Kube-Proxy sẽ gán các rules của gói tin đến vào trong chain này

```bash

root@datnd-master-node:~# iptables -t nat -L PREROUTING #lệnh này để list ra các nat rule khi có gói tin đến
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
CILIUM_PRE_nat  all  --  anywhere             anywhere             /* cilium-feeder: CILIUM_PRE_nat */
KUBE-SERVICES  all  --  anywhere             anywhere             /* kubernetes service portals */

```

Tiếp đến list ra các NAT rule s trong CHAIN này


```bash
root@datnd-master-node:~# iptables -t nat -L 
Chain KUBE-SERVICES (2 references)
target     prot opt source               destination         
KUBE-SVC-WMHZRWPBLB277GS6  tcp  --  anywhere             10.99.83.161         /* argocd/argocd-repo-server:metrics cluster IP */
KUBE-SVC-KMECTPXUBDWOU35U  tcp  --  anywhere             10.105.54.93         /* cert-manager/cert-manager-webhook:metrics cluster IP */
KUBE-SVC-BH3MYSSE7ZYZQL6A  tcp  --  anywhere             10.107.126.93        /* myapp/front-end-service cluster IP */
KUBE-SVC-FZ3AB4F3GVRYL3EW  tcp  --  anywhere             10.101.253.49        /* monitoring/prometheus-grafana-stack-k-alertmanager:http-web cluster IP */
KUBE-SVC-DR7W6XG5DQMVRL5D  tcp  --  anywhere             10.99.88.254         /* monitoring/prometheus-grafana-stack-k-prometheus:http-web cluster IP */
KUBE-SVC-RGMS66KYPCAJJEVV  tcp  --  anywhere             10.108.197.131       /* myapp/back-end-service cluster IP */
KUBE-SVC-CG5I4G2RS3ZVWGLK  tcp  --  anywhere             10.107.249.55        /* ingress-nginx/ingress-nginx-controller:http cluster IP */
KUBE-SVC-OQBJ37UZJAVIUAOI  tcp  --  anywhere             10.102.152.25        /* myapp/sqlserver cluster IP */
KUBE-SVC-ZUD4L6KQKCHD52W4  tcp  --  anywhere             10.105.54.93         /* cert-manager/cert-manager-webhook:https cluster IP */
KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */
KUBE-SVC-GZ25SP4UFGF7SAVL  tcp  --  anywhere             10.96.20.11          /* metallb-system/metallb-webhook-service cluster IP */
KUBE-SVC-4ZTUC2SWXFPJXSLF  tcp  --  anywhere             10.107.179.120       /* monitoring/prometheus-grafana-stack-k-operator:https cluster IP */
KUBE-SVC-E3IBCFULSWKQCT47  tcp  --  anywhere             10.97.79.64          /* cert-manager/cert-manager:tcp-prometheus-servicemonitor cluster IP */
KUBE-SVC-MWRHWWJJ5GR3VDOR  tcp  --  anywhere             10.101.149.4         /* argocd/argocd-applicationset-controller:webhook cluster IP */
KUBE-SVC-KZKWQ6OUXCZC3R2R  tcp  --  anywhere             10.110.226.244       /* argocd/argocd-notifications-controller-metrics:metrics cluster IP */

and more.....

KUBE-NODEPORTS  all  --  anywhere             anywhere             /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

Đây chính là các nat rule được tạo ra trong chain  KUBE-SERVICES, nơi định nghĩa các gói tin khi được chuyern đến địa chỉ 
IP của services sẽ được iptables chuyển tới các pod do service này trỏ vào, giả sử kiểm tra tiếp 1 CHAIN

```bash

root@datnd-master-node:~# iptables -t nat -L KUBE-SERVICES | grep 10.108.197.131
KUBE-SVC-RGMS66KYPCAJJEVV  tcp  --  anywhere             10.108.197.131       /* myapp/back-end-service cluster IP */

root@datnd-master-node:~# iptables -t nat -L KUBE-SVC-RGMS66KYPCAJJEVV
Chain KUBE-SVC-RGMS66KYPCAJJEVV (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  tcp  -- !192.168.10.0/24      10.108.197.131       /* myapp/back-end-service cluster IP */
KUBE-SEP-ZPYHQGISNP3GWMDD  all  --  anywhere             anywhere             /* myapp/back-end-service -> 10.0.1.122:8080 */ statistic mode random probability 0.50000000000
KUBE-SEP-AXVPFJ6VU2VXWKMK  all  --  anywhere             anywhere             /* myapp/back-end-service -> 10.0.2.209:8080 */


```

KUBE-MARK-MASQ: Quy tắc này sẽ đánh dấu các gói tin để có thể thực hiện MASQUERADE (đổi địa chỉ nguồn khi gói tin rời khỏi node) cho các gói tin không đến từ mạng nội bộ (không phải từ 192.168.10.0/24). ở đây toàn ClusterIP nên ko có gói nào cần thực hiện MASQUERADE

KUBE-SEP-ZPYHQGISNP3GWMDD và KUBE-SEP-AXVPFJ6VU2VXWKMK: Đây là các Service Endpoints (điểm cuối của dịch vụ), là các pod thực sự nhận lưu lượng. Khi gói tin đến 10.108.197.131, nó sẽ được chuyển tiếp tới một trong các pod:


- 10.0.1.122:8080 
- 10.0.2.209:8080

probability 0.5: Điều này có nghĩa là kube-proxy sẽ chọn ngẫu nhiên giữa hai pod này với xác suất 50%.

Giải thích tiếp về 1 rule cụ thể (KUBE-SEP-AXVPFJ6VU2VXWKMK)

```bash
root@datnd-master-node:~# iptables -t nat -L KUBE-SEP-AXVPFJ6VU2VXWKMK
Chain KUBE-SEP-AXVPFJ6VU2VXWKMK (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.0.2.209           anywhere             /* myapp/back-end-service */
DNAT       tcp  --  anywhere             anywhere             /* myapp/back-end-service */ tcp to:10.0.2.209:8080

```

KUBE-MARK-MASQ: Đánh dấu các gói tin để thực hiện MASQUERADE khi gói tin rời khỏi pod và quay lại vào hệ thống.

DNAT: Đây là Destination NAT, có tác dụng thay đổi địa chỉ đích của gói tin đến 10.0.2.209:8080, tức là dịch vụ back-end-service sẽ chuyển tiếp lưu lượng đến pod có địa chỉ 10.0.2.209 và cổng 8080.

> Như đã nói ở trên DNAT cho phép chuyển đổi dịa chỉ IP đích của gói tin, (ở đây là gói tin đến services rồi nó chuyển qua IP của POD)






### Tóm tắt quy trình khi giao tiếp qua Service:
Pod gửi yêu cầu đến Service: Khi một pod muốn gửi yêu cầu đến service back-end-service, nó sẽ sử dụng địa chỉ IP của service (10.108.197.131).
iptables xử lý DNAT: iptables sẽ áp dụng DNAT để chuyển tiếp yêu cầu đến một trong các pod backend (ví dụ: 10.0.1.122:8080 hoặc 10.0.2.209:8080).
Pod nhận và xử lý yêu cầu: Pod đích nhận yêu cầu và xử lý.
Trả kết quả: Kết quả được trả lại cho pod gửi yêu cầu qua quy trình tương tự.


> Lưu ý:
Kube-Proxy tự động tạo các quy tắc iptables để thực hiện việc chuyển tiếp lưu lượng giữa các Pod và dịch vụ.
Kube-Proxy cũng xử lý cân bằng tải, đảm bảo rằng yêu cầu đến dịch vụ được phân phối đồng đều giữa các pod backend.


> Facts:

A Service’s ClusterIP is a virtual IP (VIP), meaning it doesn’t have a physical network entity but is used for handling packet routing.

#### Sources

[Kube-Proxy-vs-CNI](https://medium.com/@rifewang/kubernetes-how-kube-proxy-and-cni-work-together-1255d273f291)


[Kube-Proxy-how-its-working](https://medium.com/@amroessameldin/kube-proxy-what-is-it-and-how-it-works-6def85d9bc8f)