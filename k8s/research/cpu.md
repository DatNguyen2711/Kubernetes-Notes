# CPU trong K8S

1. Đơn vị **millicore** trong k8s, giả sử ta có 1 pod như sau:


```yaml
backEnd:
  replicaCount: 1
  name: back-end
  image:
    repository: datnd2711/pharmacy-be
    tag: prod-0.104
    pullPolicy: IfNotPresent
  imagePullSecrets:
    name: my-dockerhub-secret
  service:
    name: back-end-service
    type: ClusterIP
    port: 8080
    targetPort: "http-backend" 
  resources:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
```

- Khi nghe tới đơn vị này nhiều người sẽ lầm tưởng rằng là k8s sẽ chia tài nguyên cho pod là 1/10 CPU (100m) ví dụ, cấp cho một process 100 millicore (tức 1/10 CPU) nhưng thực ra không phải như vậy

- Kubernetes không trực tiếp chia CPU thành millicore, mà nó chỉ chuyển cấu hình này xuống container runtime (như Docker, containerd...). Chính Linux Kernel mới là nơi xử lý thực sự, thông qua **CFS Bandwidth Control** trong **cgroups v1/v2**.

💡 Cốt lõi của cơ chế:

- Linux dùng CFS (Completely Fair Scheduler) và cgroups để giới hạn bao nhiêu thời gian CPU một process (container) được phép sử dụng trong một khoảng thời gian nhất định.

**Hai thông số quan trọng**:

`quota`: bao nhiêu thời gian (microsecond) được phép dùng CPU.

`period`: chu kỳ tính toán tổng thời gian (thường là 100ms, 250ms...).

📌 Ví dụ minh họa:
- Nếu quota = 250ms, period = 250ms ⇒ dùng được 100% CPU (toàn quyền chạy trong suốt chu kỳ).

- Nếu quota = 125ms, period = 250ms ⇒ dùng được 50% CPU.

- Nếu quota = 500ms, period = 250ms ⇒ dùng được 200% CPU (2 core).

➡ Vậy “100 millicore” đơn giản là 10% của 1 core ⇒ tức quota=25ms, period=250ms.

👉 Tóm lại:
“Millicore” thực chất chỉ là biểu diễn tỉ lệ thời gian sử dụng CPU của process trong 1 chu kỳ, được Linux Kernel giới hạn thông qua cgroups + CFS, chứ không phải chia thật CPU thành từng lát 1/1000. Millicore là time-based chứ không hardware-based!

### Check quota period of pod

1. Linux mặc định dùng `period = 100ms` = **100000 microseconds**.

- Ví dụ, 1 pod có thông số như sau:

```yaml
  resources:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
```

- Ta sẽ check pod

```bash 
datnd@datnd-master-node:~$ k exec -it back-end-5f5548f44-fk75s -n myapp -- bash
root@back-end-5f5548f44-fk75s:/app# cat /sys/fs/cgroup/cpu
cpu.idle               cpu.max.burst          cpu.stat               cpu.uclamp.min         cpu.weight.nice        cpuset.cpus.effective  cpuset.mems
cpu.max                cpu.pressure           cpu.uclamp.max         cpu.weight             cpuset.cpus            cpuset.cpus.partition  cpuset.mems.effective

root@back-end-5f5548f44-fk75s:/app# cat /sys/fs/cgroup/cpu.max
50000 100000

→ quota = 50ms, period = 100ms
```

📌 Tổng kết:

`quota`	            Container được chạy bao lâu mỗi chu kỳ
`period`	        Độ dài mỗi chu kỳ, thường là 100ms
`quota / period`	Tính ra được container được bao nhiêu core


# Sources:
[CPU resource limit](https://stackoverflow.com/questions/71944390/how-are-cpu-resource-units-millicore-millicpu-calculated-under-the-hood)