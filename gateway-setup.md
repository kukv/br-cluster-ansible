# Gateway setup

Gateway構築時にセットアップした内容のメモ
ansibleで自動化する際に参考にする

## システム設定

基本はrootユーザーで実施

### ホスト名を追加

```sh
echo "127.0.1.1 $(hostname)" >> /etc/hosts
```

### ネームサーバーの設定

```sh
mv /etc/resolv.conf /etc/resolv.conf.bk
cat <<'EOF' > /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
EOF
```

### cgroupの有効化

```sh
sed -i -e "1 s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1/g" /boot/firmware/cmdline.txt
```

### GPUメモリの割り当てサイズを変更

```sh
echo "gpu_mem=16" >> /boot/firmware/config.txt
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

### iptables-persistentをインストール時に手動で入力する項目を自動で入力するようにするための事前設定

```sh
echo iptables-persistent iptables-persistent/autosave_v4 boolean true | debconf-set-selections
echo iptables-persistent iptables-persistent/autosave_v6 boolean true | debconf-set-selections
```
### 必要なライブラリ群を一括インストール

```sh
apt install -y \
    git \
    dnsutils \
    curl \
    ca-certificates \
    gnupg \
    netfilter-persistent \
    iptables-persistent \
    nftables \
    lsof \
    net-tools \
    jq
```

## docker install

基本はrootユーザーで実施

### 公式DockerのGPG-keyを追加

```sh
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```

### Dockerリポジトリを追加

```sh
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Dockerインストール

```sh
apt update -y
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### dockerグループに運用ユーザーを追加

```sh
if [[ $(grep -c docker /etc/group) -eq 0 ]]; then
    groupadd docker
fi
gpasswd -a bradmin docker
```

### docker内で利用するDNSを追加

```sh
cat <<'EOF' > /etc/docker/daemon.json
{
    "dns": ["8.8.8.8", "8.8.4.4"],
    "iptables": false
}
EOF
```

### dockerを再起動

```sh
systemctl restart docker
```

## ルーター/ファイアウォールの設定

基本はrootユーザーで実施

### IPフォワード有効化

```sh
sysctl -w net.ipv4.ip_forward=1 >> /etc/sysctl.conf
```

### nftablesの設定

#### ディレクトリを作成

```sh
mkdir /etc/nftables.d
```

#### ルールセット作成

```sh
cat <<'EOF' > /etc/nftables.conf
#!/usr/sbin/nft -f

# 設定の初期化
flush ruleset

include "/etc/nftables.d/defines.nft"

table inet filter {
    chain global {
        # state management
        ct state established,related accept
        ct state invalid drop
    }
    include "/etc/nftables.d/sets.nft"
    include "/etc/nftables.d/filter-input.nft"
    include "/etc/nftables.d/filter-output.nft"
    include "/etc/nftables.d/filter-forward.nft"
}

# NAT変換用のテーブル定義
table ip nat {
    include "/etc/nftables.d/sets.nft"
    include "/etc/nftables.d/nat-prerouting.nft"
    include "/etc/nftables.d/nat-postrouting.nft"
}
EOF
```

#### 変数定義

```sh
cat <<'EOF' > /etc/nftables.d/defines.nft
# ブロードキャストとマルチキャストの定義(IPv4)
define badcast_addr = { 255.255.255.255, 224.0.0.1, 224.0.0.251 }

# ブロードキャストとマルチキャストの定義(IPv6)
define ip6_badcast_addr = { ff02::16 }

# インバウンドを許可するTCP通信
define in_tcp_accept = { ssh, https, http, 6443 }

# インバウンドを許可するUDP通信
define in_udp_accept = { snmp, domain, ntp, bootps }

# アウトバウンドを許可するTCP通信
define out_tcp_accept = { domain, bootps , ntp, http, https, ssh }

# アウトバウンドを許可するUDP通信
define out_udp_accept = { domain, bootps , ntp }

# LAN側のインターフェース定義
define lan_interface = eth0

# WAN側のインターフェース定義
define wan_interface = wlan0

# dockerのインターフェース定義
define docker_interface = { docker0 }

# LAN側のネットワーク定義
define lan_network = 172.22.10.0/24

# フォワードを許可するTCP通信
define forward_tcp_accept = { http, https, ssh }

# フォワードを許可するUDP通信
define forward_udp_accept = { domain, ntp }
EOF
```

#### 型付けされた変数セット

```sh
cat <<'EOF' > /etc/nftables.d/sets.nft
set blackhole {
    type ipv4_addr;
    elements = $badcast_addr
}

set forward_tcp_accept {
    type inet_service; flags interval;
    elements = $forward_tcp_accept
}

set forward_udp_accept {
    type inet_service; flags interval;
    elements = $forward_udp_accept
}

set in_tcp_accept {
    type inet_service; flags interval;
    elements = $in_tcp_accept
}

set in_udp_accept {
    type inet_service; flags interval;
    elements = $in_udp_accept
}

set ip6blackhole {
    type ipv6_addr;
    elements = $ip6_badcast_addr
}

set out_tcp_accept {
    type inet_service; flags interval;
    elements = $out_tcp_accept
}

set out_udp_accept {
    type inet_service; flags interval;
    elements = $out_udp_accept
}
EOF
```

#### インバウンド側のトラフィックフィルタリングルール

```sh
cat <<'EOF' > /etc/nftables.d/filter-input.nft
chain input {
    # policy
    type filter hook input priority 0; policy drop;

    # global
    jump global

    # localhost
    iif lo accept

    # icmp
    meta l4proto {icmp,icmpv6} accept

    # input udp accepted
    udp dport @in_udp_accept ct state new accept

    # input tcp accepted
    tcp dport @in_tcp_accept ct state new accept
}
EOF
```

#### アウトバウンド側のトラフィックフィルタリングルール

```sh
cat <<'EOF' > /etc/nftables.d/filter-output.nft
chain output {
    # policy: Allow any output traffic
    type filter hook output priority 0;
}
EOF
```

#### フォワーディングのトラフィックフィルタリングルール

```sh
cat <<'EOF' > /etc/nftables.d/filter-forward.nft
chain forward {
    # policy
    type filter hook forward priority 0; policy drop;

    # global
    jump global

    # lan to wan tcp
    iifname $lan_interface ip saddr $lan_network oifname $wan_interface tcp dport @forward_tcp_accept ct state new accept

    # lan to wan udp
    iifname $lan_interface ip saddr $lan_network oifname $wan_interface udp dport @forward_udp_accept ct state new accept

    # ssh from wan
    iifname $wan_interface oifname $lan_interface ip daddr $lan_network tcp dport ssh ct state new accept

    # http from wan
    iifname $wan_interface oifname $lan_interface ip daddr $lan_network tcp dport {http, https} ct state new accept

    # port-forwarding from wan
    iifname $wan_interface oifname $lan_interface ip daddr 172.22.10.50 tcp dport 8080 ct state new accept

    # docker
    ct state { established, related } accept;
    ct state invalid drop;

    iifname $docker_interface accept;     
  }
EOF
```

#### NATプレルーティングルール

```sh
cat <<'EOF' > /etc/nftables.d/nat-prerouting.nft
chain prerouting {
    # policy
    type nat hook prerouting priority 0;
  }
EOF
```

#### NATポストルーティングルール

```sh
cat <<'EOF' > /etc/nftables.d/nat-postrouting.nft
chain postrouting {
    # policy
    type nat hook postrouting priority 100;

    # masquerade lan to wan
    ip saddr $lan_network oifname $wan_interface masquerade
    oifname != $docker_interface masquerade;
}
EOF
```

#### 設定を読み込み

```sh
nft -f /etc/nftables.conf
netfilter-persistent save
```

## ゲートウェイのセットアップ

基本はbradminユーザーで実施

### 資材のダウンロード

```sh
mkdir /works

git clone https://github.com/kukv/br-cluster-gateway.git /works/br-cluster-gateway
chown -R bradmin:bradmin /works/br-cluster-gateway
```

### systemd-resolveの無効化

```sh
systemctl stop systemd-resolved
systemctl disable systemd-resolved
rm /etc/resolv.conf
```

### コンテナを立ち上げ

```sh
docker compose up -d
```
