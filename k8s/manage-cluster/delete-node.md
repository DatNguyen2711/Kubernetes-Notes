1. Cordon node and move pods form that node

```bash
kubectl cordon node worker-node-4
kubectl drain datnd-worker-node-4 --ignore-daemonsets --delete-local-data --force

kubectl delete node datnd-worker-node-4

```
2. Delete config cni, cri,... form node

```bash
sudo kubeadm reset --force

sudo rm -rf /etc/cni/net.d
sudo rm -rf /etc/kubernetes
sudo rm -rf /var/lib/etcd
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/cni
sudo rm -rf /var/lib/dockershim
sudo rm -rf /var/lib/containerd
sudo rm -rf /var/lib/crio
sudo rm -rf /etc/systemd/system/kubelet.service.d
sudo rm -rf /etc/systemd/system/crio.service.d



sudo apt remove -y kubeadm kubectl kubelet kubernetes-cni cri-o cri-tools
sudo apt autoremove -y


sudo systemctl daemon-reload
sudo systemctl stop kubelet
sudo systemctl disable kubelet
sudo systemctl stop crio
sudo systemctl disable crio

sudo reboot


```