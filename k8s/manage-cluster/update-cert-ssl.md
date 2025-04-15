# Update SSL Certificate Kubernetes

**Khi sản phẩm chạy trên production thì các app sẽ thường sử dụng cert đi mua từ bên ngoài và theo thời gian khi đến hạn chúng ta phải update manual**

### Follow các steps sau

![alt text](image-2.png)
1. Download cert mới về

- Như ở ảnh trên thì khi đến hạn update cert thì chúng ta sẽ được gửi các file như trên download về rồi giải nén ra sẽ có được các files như sau:

```Shell 

➜  star_gamota_com ll
total 40K
-rw-rw-r-- 1 datlaid datlaid 2.2K Apr 24  2024 ChainCA.crt
-rw-rw-r-- 1 datlaid datlaid 4.3K Apr 24  2024 Chain_RootCA_Bundle.crt
-rw-rw-r-- 1 datlaid datlaid 2.1K Apr 16  2024 RootCA.crt
-rw-rw-r-- 1 datlaid datlaid 2.2K Apr 14 10:58 star_gamota_com_cert.pem
➜  star_gamota_com pwd
/home/datlaid/Downloads/Telegram Desktop/star_gamota_com

```

2. Xác định cert cần update
- Trong trường hợp này thì cụm đang lấy ví dụ sử dụng cert có tên **star-gamota-com-tls** cho tất cả các services chạy trên production vì thế ta cần update nó
- Kiểm tra thông tin cert 


```bash
kd  secret star-gamota-com-tls -n prod-blogsg     

Name:         star-gamota-com-tls
Namespace:    prod-blogsg
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  6459 bytes
tls.key:  1703 bytes

kg secret star-gamota-com-tls -n prod-blogsg -oyaml > cert.yaml 
```
- Kiểm tra thông tin của secret thì chúng ta thấy có dạng như sau:

![alt text](image-4.png)

- Đây là 1 secret k8s dạng tls sau khi get raw data.
- Ta sẽ cần update **tls.crt** ở trong secret này
- Giữ nguyên **tls.key (private key)** 

> Sử dụng base64 decode để có thể get về dạng data decrypt 

3. Ghép cert 
- Cần phải ghép cert 

```bash
star_gamota_com_cert.pem          # ✔️ Cert chính (leaf cert)
ChainCA.crt                       # ✔️ Intermediate CA
Chain_RootCA_Bundle.crt           # ❓ Bundle (có thể = ChainCA + RootCA)
RootCA.crt                        # ❌ Không cần public, client có sẵn rồi
```

- Thường thì file **Chain_RootCA_Bundle.crt** đã ghép sẵn cert intermidiate vào trong rồi nên chỉ cần ghép 2 cert là **star_gamota_com_cert.pem** và cert bundle **Chain_RootCA_Bundle.crt** vào là được 


```bash 
( cat star_gamota_com_cert.pem; echo ""; cat Chain_RootCA_Bundle.crt ) > fullchain.pem
```

- Tiếp đến, lấy lại private.key đang dùng

```bash 

kubectl get secret star-gamota-com-tls -n prod-blogsg -o jsonpath="{.data.tls\.key}" | base64 -d > star_gamota_com.key

```

- Tạo lại tls secret

```shell 
kubectl create secret tls star-gamota-com-tls \
  --cert=fullchain.pem \
  --key=star_gamota_com.key \
  -n prod-blogsg \
  --dry-run=client -o yaml | kubectl apply -f -
```

- Kiểm tra 1 domain bất kỳ xem có đúng cert chưa

```bash 
openssl s_client -connect star.gamota.com:443 -showcerts

# Hoặc làm theo cách này


kg secret star-gamota-com-tls -n prod-blogsg -oyaml > cert.yaml 

# Sau đó decode base64 tls.crt nếu cert giống với các file cert ta dùng ban đầu thì ok
```

