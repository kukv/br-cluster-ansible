# Master setup

Master(Primary)ノード構築時にセットアップした内容のメモ
ansibleで自動化する際に参考にする

## システム設定

基本はrootユーザーで実施

### ホスト名を追加

```sh
echo "127.0.1.1 $(hostname)" >> /etc/hosts
```

### cgroupの有効化

```sh
sed -i -e "1 s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1/g" /boot/firmware/cmdline.txt
```

### GPUメモリの割り当てサイズを変更

```sh
echo "gpu_mem=16" >> /boot/firmware/config.txt
```

## スワップの無効化

```sh
swapoff --all
```

### apt upgradeを行うたびに、Pending kernel upgrade と表示されてしまうのでその対処

```sh
apt purge -y needrestart
```

### 再起動

```sh
reboot
```

## パッケージ関連

基本はrootユーザーで実施

### パッケージの最新化

```sh
apt update && apt -y upgrade
```

## K3s-masterのセットアップ

基本はbradminユーザーで実施

### 下準備

```sh
sudo mkdir -p /etc/rancher/k3s
sudo chown bradmin:bradmin -R /etc/rancher
```

### クラスタトークン作成

```sh
echo JTyUx4e9XVAHdsereuVW7fwXZ > /etc/rancher/k3s/cluster-token
```

### kubelet-configの設定

```sh
cat <<EOF > /etc/rancher/k3s/kubelet.config
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
shutdownGracePeriod: 30s
shutdownGracePeriodCriticalPods: 10s
EOF
```

### k3sの設定

```sh
cat <<EOF > /etc/rancher/k3s/config.yaml
token-file: /etc/rancher/k3s/cluster-token
disable:
  - local-storage
  - servicelb
  - traefik
etcd-expose-metrics: true
kube-controller-manager-arg:
  - bind-address=0.0.0.0
  - terminated-pod-gc-threshold=10
kube-proxy-arg:
  - metrics-bind-address=0.0.0.0
kube-scheduler-arg:
  - bind-address=0.0.0.0
kubelet-arg:
  - config=/etc/rancher/k3s/kubelet.config
node-taint:
  - node-role.kubernetes.io/master=true:NoSchedule
tls-san:
  - 172.22.10.1
  - 192.168.2.50
write-kubeconfig-mode: 644
EOF
```

### K3s-masterの起動

Primaryの場合

```sh
curl -sfL https://get.k3s.io | sh -s - server --cluster-init
```

Secondaryの場合

```sh
curl -sfL https://get.k3s.io | sh -s - server --server https://172.22.10.60:6443
```

### K3s-masterのアンインストール

```sh
/usr/local/bin/k3s-uninstall.sh
```
