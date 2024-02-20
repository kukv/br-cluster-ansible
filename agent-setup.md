# Agent setup

Agentノード構築時にセットアップした内容のメモ
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

### snapパッケーの削除

```sh
snap remove lxd && snap remove core22 && snap remove snapd
apt purge -y snapd
apt autoremove -y
```

### NTPサーバーの変更

```sh
sed -i -e "s/#NTP=/NTP=ntp1.bright-room.work/g" /etc/systemd/timesyncd.conf
```

### タイムゾーンの変更

```sh
timedatectl set-timezone Asia/Tokyo
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

## K3s-agentのセットアップ

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
node-label:
  - 'node_type=worker'
kubelet-arg:
  - 'config=/etc/rancher/k3s/kubelet.config'
kube-proxy-arg:
  - 'metrics-bind-address=0.0.0.0'
EOF
```

### K3s-masterの起動

```sh
curl -sfL https://get.k3s.io | sh -s - agent --server https://172.22.10.1:6443
```

### workerロールを付与

```sh
kubectl label nodes <<node名>> kubernetes.io/role=worker
```

### K3s-masterのアンインストール

```sh
/usr/local/bin/k3s-agent-uninstall.sh
```
