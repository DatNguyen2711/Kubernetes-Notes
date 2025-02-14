## Normal Cluster

## RKE
1. Thường cụm đầy disk là do có quá nhiều images layer cũ và các images ko sử dụng nữa chứ ko phải do logs (logs chiếm ít disk có thể chưa
đến 1GB)
2. Trong trường hợp node không có crictl:

```bash
echo "export CRI_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock" >> ~/.bashrc
echo "export PATH=$PATH:/var/lib/rancher/rke2/bin" >> ~/.bashrc
source  ~/.bashrc


# remove all unused images
/var/lib/rancher/rke2/bin/crictl --runtime-endpoint unix:///run/k3s/containerd/containerd.sock rmi --prune
crictl images --prune

```