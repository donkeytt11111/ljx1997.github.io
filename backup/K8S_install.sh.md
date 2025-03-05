```shell

#!/bin/bash

# 变量定义
IP_ADDRESS=$1
HOSTNAME=$2

# 检查参数是否为空
if [ -z "$IP_ADDRESS" ] || [ -z "$HOSTNAME" ]; then
    echo "使用错误: 使用方法为 bash test.sh <IP地址> <主机名>"
    exit 1
fi

# 设置主机名
set_hostname() {
    hostnamectl set-hostname $HOSTNAME
    current_hostname=$(hostname)
    if [[ "$current_hostname" == *"cncp"* ]]; then
        echo "Hostname set successfully and contains 'cncp'."
        echo "$IP_ADDRESS $HOSTNAME" | sudo tee -a /etc/hosts
    else
        echo "Hostname set successfully but does not contain 'cncp'. Exiting script."
        exit 1
    fi
}

# 初始化网络配置
initialize_network() {
    nmcli con mod enp3s0 connection.autoconnect yes
    systemctl enable NetworkManager

    cat <<EOF | sudo tee -a /etc/profile
#modprobe
modprobe ip6_tables
modprobe ip6table_mangle
modprobe ip6table_raw
modprobe ip6table_filter
EOF
    source /etc/profile
    echo "net.ipv6.conf.default.forwarding=1" >> /etc/sysctl.conf
    echo "net.ipv6.conf.all.forwarding=1" >> /etc/sysctl.conf
    sysctl -p
    sudo systemctl stop firewalld
    sudo systemctl disable firewalld
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
    sudo swapoff -a

    mkdir -p /opt/cni/bin
    mkdir -p /etc/cni/net.d
    tar -xzvf cni-plugins-linux-amd64-v1.6.2.tgz -C /opt/cni/bin
}



# 安装和配置 chrony
install_and_configure_chrony() {
    yum install -y chrony
    sed -i.bak '3,6d' /etc/chrony.conf
    sed -i '3c server ntp1.aliyun.com iburst' /etc/chrony.conf

    systemctl enable chronyd --now
    systemctl restart chronyd
    if systemctl is-active --quiet chronyd; then
        echo "chrony 服务正在运行"
    else
        echo "chrony 服务未运行"
        exit 1
    fi
    chronyc sources
}

# 安装和配置 IPVS
install_and_configure_ipvs() {
    yum install -y conntrack-tools socat iproute-tc
    yum -y install ipset ipvsadm ebtables socat ipset conntrack
    cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack
EOF

    chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules
}

# 安装和配置 containerd
install_and_configure_containerd() {
    # 解压 containerd
    tar xf cri-containerd-1.7.3-linux-amd64.tar.gz  -C /
    cp 05-cilium.conflist /etc/cni/net.d
    sudo cp -rf  helm nerdctl runc  cilium /usr/local/bin/
    mkdir -p /etc/containerd
    cp -rf config.toml /etc/containerd/
    cp -rf conf.d /etc/containerd/
    sudo chmod +x  /usr/local/bin/helm /usr/local/bin/nerdctl  /usr/local/bin/runc  /usr/local/bin/cilium
    sudo systemctl start containerd
    sudo systemctl enable containerd
    sudo systemctl restart containerd
}

# 创建并配置 containerd.service 文件
create_containerd_service_file() {
    cat > /etc/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStart=/usr/local/bin/containerd
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGQUIT
Delegate=yes
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStopSec=0
Restart=always
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOF

    # 重新加载 systemd 配置
    sudo systemctl daemon-reload
}

# 安装和配置 Kubernetes
install_and_configure_k8s() {
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

    modprobe br_netfilter && sysctl -p /etc/sysctl.d/k8s.conf


cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://kubernetes.io/docs/concepts/overview/components/#kubelet
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --kubeconfig=/etc/kubernetes/kubelet.conf \\
  --config=/var/lib/kubelet/config.yaml \\
  --v=2
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF


    systemctl daemon-reload
    tar -xzf kubernetes-server-linux-amd64.tar.gz
    sudo cp kubernetes/server/bin/kubeadm /usr/local/bin/
    sudo cp kubernetes/server/bin/kubelet /usr/local/bin/
    sudo cp kubernetes/server/bin/kubectl /usr/local/bin/

    systemctl enable kubeadm --now
    systemctl enable kubelet --now
    systemctl enable kubectl --now
    mkdir -p /var/lib/kubelet
    chown -R root:root /var/lib/kubelet
    chmod 755 /var/lib/kubelet
    kubeadm config print init-defaults --component-configs KubeletConfiguration > /var/lib/kubelet/config.yaml
    # 添加主机名和IP到 /etc/hosts
    systemctl enable kubelet && systemctl start kubelet
}

# 替换 kubeadm 配置文件内容
replace_kubeadm_config() {
    # 使用 sed 替换文件内容
    sed -i "s/172.16.9.120/$IP_ADDRESS/g" kubeadm.yaml
    sed -i "s/CNCP-MS-01/$HOSTNAME/g" kubeadm.yaml
    sed -i "s/cncp-ms-01/$HOSTNAME/g" cilium.yaml
    sed -i "s/172.16.9.120/$IP_ADDRESS/g" cluster.yaml
    echo "替换完成"
}

# 设置 Kubernetes
setup_kubernetes() {
    kubeadm init --config kubeadm.yaml --skip-phases=addon/kube-proxy --v=5
    mkdir -p /root/.kube
    cp -i /etc/kubernetes/admin.conf /root/.kube/config
    chown root:root /root/.kube/config
    kubectl create -f cilium.yaml
    kubectl create -f cluster.yaml
    kubectl create -f metallb-native.yaml
    kubectl create -f dashboard-deploy-singlestack.yaml
    kubectl create -f cni-installation-ds.yaml
    kubectl create -f openebs.yaml
    kubectl create -f openebs-sc.yaml
}

# 安装和配置 NFS
install_and_configure_nfs() {
    yum install -y nfs-utils
    systemctl enable --now nfs-server
    echo "/var/local-storage *(rw,sync,no_root_squash)" | sudo tee -a /etc/exports
    systemctl restart nfs-server
}

# 安装和配置桥接网络
install_and_configure_bridge() {
    yum install -y bridge-utils
    brctl addbr br-eno6
    ip link set br-eno6 up
    brctl addif br-eno6 eno6
    systemctl enable bridge-utils
}

# 设置主机名
set_hostname

# 初始化网络配置
initialize_network


# 安装和配置chrony
install_and_configure_chrony

# 安装和配置ipvs
install_and_configure_ipvs

# 安装和配置containerd
create_containerd_service_file

# 安装和配置k8s
install_and_configure_containerd

# 安装和配置k8s
install_and_configure_k8s

# 替换kubeadm配置
replace_kubeadm_config

# 安装和配置kubernetes
setup_kubernetes

# 安装和配置nfs
install_and_configure_nfs

# 安装和配置bridge
# install_and_configure_bridge
