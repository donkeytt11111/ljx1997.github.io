```shell
#!/bin/bash


if [ -z "$1" ] || [ -z "$2" ] || [ -z "$3" ] || [ -z "$4" ] || [ -z "$5" ] || [ -z "$6" ] || [ -z "$7" ] || [ -z "$8" ]; then
    echo "请提供八个IP地址作为参数，例如：bash ddi.sh 地址池：IPV4_1 IPV4_2 IPV6_1 IPV6_2 dnsIPV4_1 dnsIPV6_1 dhcpIPV4_1 dhcpIPV6_1"
    echo "使用方式：bash ddi_install.sh 172.16.9.240 172.16.9.244 2408:862E:406:109::1000 2408:862E:406:109::2000 172.16.9.241 2408:862E:406:109::1000 172.16.9.242 2408:862E:406:109::1001"
    exit 1
fi


LOG_FILE="./deployment.log"

# 日志
log() {
  local message="$(date +'%Y-%m-%d %H:%M:%S') - $1"
  echo "$message" 
  echo "$message" >> "$LOG_FILE" 
}

kubectl taint nodes cncp-ms-01 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes cncp-ms-02 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes cncp-ms-03 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes cncp-ns-01 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes cncp-ns-02 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes cncp-ns-03 node-role.kubernetes.io/control-plane:NoSchedule-

check_pods_running() {
  while true; do
    pod_status=$(kubectl get pods --all-namespaces --no-headers -o custom-columns=STATUS:.status.phase)
    
    if echo "$pod_status" | grep -q -vE "Running|Completed|Succeeded"; then
      log "并非所有Pod都处于Running、Completed或Succeeded状态，等待5秒..."
      sleep 5
    else
      log "所有Pod都处于Running、Completed或Succeeded状态。"
      sleep 5
      break
    fi
  done
}

# check_pods_running() {
#   while true; do
#     pod_status=$(kubectl get pods --all-namespaces --no-headers -o custom-columns=STATUS:.status.phase)
    
#     if echo "$pod_status" | grep -q -v "Running"; then
#       log "并非所有Pod都处于Running状态，等待5秒..."
#       sleep 5
#     else
#       log "所有Pod都处于Running状态。"
#       sleep 5
#       break
#     fi
#   done
# }

auto_helm() {
  helm install signoz ./signoz -n platform

  check_pod_status() {
    kubectl get pods --all-namespaces | grep -v "Running\|Completed" | grep -q platform
    return $?
  }

  # 检查Pod状态
  checks=0
  while [ $checks -lt 5 ]; do
    sleep 30
    check_pod_status
    if [ $? -ne 0 ]; then
      log "所有Pods都处于Running或Completed状态。"
      break
    fi
    checks=$((checks + 1))
    log "检查 $checks: 部分Pods不处于Running或Completed状态。30秒后重试..."
  done

  if [ $checks -eq 5 ]; then
    log "经过5次检查，部分Pods仍不处于Running或Completed状态。正在卸载Signoz..."
    helm uninstall signoz -n platform

    # 检查Pod状态，如果都是running状态则重新安装Signoz
    while true; do
      sleep 30
      check_pod_status
      if [ $? -eq 0 ]; then
        log "所有Pods都处于Running或Completed状态。正在重新安装Signoz..."
        helm install signoz ./signoz -n platform
        break
      fi
      log "部分Pods不处于Running或Completed状态。30秒后重试..."
    done
  fi
}

update_clickhouse() {
  SERVICE_DEFINITION=$(kubectl get svc signoz-clickhouse -n platform -o yaml)

  if [ -z "$SERVICE_DEFINITION" ]; then
    log "无法获取Service定义: signoz-clickhouse"
    return 1
  fi
  MODIFIED_DEFINITION=$(echo "$SERVICE_DEFINITION" | sed -e 's/internalTrafficPolicy: Cluster/internalTrafficPolicy: Cluster\n  ipFamilies:\n  - IPv4\n  ipFamilyPolicy: SingleStack/g')
  MODIFIED_DEFINITION=$(echo "$MODIFIED_DEFINITION" | sed -e 's/ports:/ports:\n  - name: http\n    nodePort: 30123\n    port: 8123\n    protocol: TCP\n    targetPort: 8123\n  - name: tcp\n    nodePort: 31570\n    port: 9000\n    protocol: TCP\n    targetPort: 9000/g')
  MODIFIED_DEFINITION=$(echo "$MODIFIED_DEFINITION" | sed -e 's/type: ClusterIP/type: NodePort/g')

  echo "$MODIFIED_DEFINITION" | kubectl apply -f -

  log "Service已更新: signoz-clickhouse"
}

update_frontend() {
  SERVICE_DEFINITION=$(kubectl get svc signoz-frontend -n platform -o yaml)

  if [ -z "$SERVICE_DEFINITION" ]; then
    log "无法获取Service定义: signoz-frontend"
    return 1
  fi
  MODIFIED_DEFINITION=$(echo "$SERVICE_DEFINITION" | awk '
  BEGIN {
    in_ports_section = 0
  }
  {
    # 查找 internalTrafficPolicy 并插入 ipFamilies 和 ipFamilyPolicy
    if ($0 ~ /^  internalTrafficPolicy:/) {
      print
      print "  ipFamilies:"
      print "  - IPv4"
      print "  ipFamilyPolicy: SingleStack"
      next
    }

    # 查找 ports 并插入新的端口定义
    if ($0 ~ /^  ports:/) {
      in_ports_section = 1
      print
      print "  - name: http"
      print "    nodePort: 32452"
      print "    port: 3301"
      print "    protocol: TCP"
      print "    targetPort: http"
      next
    }

    # 如果在 ports 部分，跳过原始端口定义
    if (in_ports_section && $0 ~ /^  - /) {
      next
    }

    # 替换 type: ClusterIP 为 type: NodePort
    if ($0 ~ /^  type: ClusterIP/) {
      print "  type: NodePort"
      next
    }

    print
  }
  ')
  echo "$MODIFIED_DEFINITION" | kubectl apply -f -
  log "Service已更新: signoz-frontend"
}

update_otel_collector() {
  SERVICE_DEFINITION=$(kubectl get svc signoz-otel-collector -n platform -o yaml)

  if [ -z "$SERVICE_DEFINITION" ]; then
    log "无法获取Service定义: signoz-otel-collector"
    return 1
  fi

  MODIFIED_DEFINITION=$(echo "$SERVICE_DEFINITION" | sed -e 's/internalTrafficPolicy: Cluster/internalTrafficPolicy: Cluster\n  ipFamilies:\n  - IPv4\n  ipFamilyPolicy: SingleStack/g')
  MODIFIED_DEFINITION=$(echo "$MODIFIED_DEFINITION" | sed -e 's/type: ClusterIP/type: NodePort/g')
  MODIFIED_DEFINITION=$(echo "$MODIFIED_DEFINITION" | sed -e 's/ports:/ports:\n  - name: otlp\n    nodePort: 32317\n    port: 4317\n    protocol: TCP\n    targetPort: otlp\n  - name: otlp-http\n    port: 4318\n    protocol: TCP\n    targetPort: otlp-http/g')

  echo "$MODIFIED_DEFINITION" | kubectl apply -f -
  log "Service已更新: signoz-otel-collector"
}

update_otel_agent() {
  # 获取 DaemonSet 的 YAML 定义
  DAEMONSET_DEFINITION=$(kubectl get ds infra-k8s-infra-otel-agent -n platform -o yaml)

  if [ -z "$DAEMONSET_DEFINITION" ]; then
    echo "无法获取DaemonSet定义: infra-k8s-infra-otel-agent"
    return 1
  fi

  HOST_ALIASES=""
  while true; do
    read -p "请输入IP地址 (输入'q'退出): " ip
    [ "$ip" == "q" ] && break
    read -p "请输入主机名: " hostname

    HOST_ALIASES+="      - ip: $ip\n        hostnames:\n          - $hostname\n"
  done

  # 插入 hostAliases 到 tolerations 下的 - operator: Exists 之后
  MODIFIED_DEFINITION=$(echo "$DAEMONSET_DEFINITION" | sed -e "/- operator: Exists/a \      hostAliases:\n$HOST_ALIASES")

  # 应用修改后的 YAML 定义
  echo "$MODIFIED_DEFINITION" | kubectl apply -f -
  echo "DaemonSet已更新: infra-k8s-infra-otel-agent"
}

replace_all_ips() {
    local IPV4_1=$1
    local IPV4_2=$2
    local IPV6_1=$3
    local IPV6_2=$4
    local dnsIPV4_1=$5
    local dnsIPV6_1=$6
    local dhcpIPV4_1=$7
    local dhcpIPV6_1=$8

    # 替换ipaddresspool.yaml中的IPv4地址
    sed -i "s/172.16.9.240/$IPV4_1/g" ipaddresspool.yaml
    sed -i "s/172.16.9.244/$IPV4_2/g" ipaddresspool.yaml

    # 替换ipaddresspool.yaml中的IPv6地址
    sed -i "s/2408:862E:406:109::1000/$IPV6_1/g" ipaddresspool.yaml
    sed -i "s/2408:862E:406:109::2000/$IPV6_2/g" ipaddresspool.yaml

    # 替换07_dns.yaml中的IPv4地址
    sed -i "s/172.16.9.240/$dnsIPV4_1/g" 07_dns.yaml

    # 替换07_dns.yaml中的IPv6地址
    sed -i "s/2408:862E:406:109::1000/$dnsIPV6_1/g" 07_dns.yaml

    # 替换08_dhcp_ddi_new.yml中的IPv4地址
    sed -i "s/172.16.9.241/$dhcpIPV4_1/g" 08_dhcp_ddi_new.yml

    # 替换08_dhcp_ddi_new.yml中的IPv6地址
    sed -i "s/2408:862E:406:109::1001/$dhcpIPV6_1/g" 08_dhcp_ddi_new.yml


}







# 开始部署pgsql
log "开始部署pgsql..."
kubectl apply -f 01_kubegres-deploy.yaml
check_pods_running


# 开始部署signoz
log "开始部署signoz..."
kubectl apply -f 02_openebs-sc2.yaml
check_pods_running
kubectl apply -f 03_platform.yaml
check_pods_running


# 部署helm-signoz
kubectl apply -f 01_kubegres-deploy.yaml
log "开始部署helm-signoz..."
auto_helm


# 更新signoz参数
log "开始更新signoz参数..."
update_clickhouse
update_frontend
update_otel_collector

#infra
echo "开始部署infra"
helm install infra ./k8s-infra -n platform
check_pods_running

#修改infra文件
log "开始更新infra参数..."
update_otel_agent
echo "infra参数修改成功"

#部署前后端
kubectl apply -f 05_ddi-netbox-deployment.yaml
check_pods_running
kubectl apply -f 06_ddi-deployment-https.yaml
check_pods_running


#地址池分配
# 调用函数并传递参数
replace_all_ips $1 $2 $3 $4 $5 $6 $7 $8

kubectl apply -f ipaddresspool.yaml
check_pods_running

kubectl apply -f 07_dns.yaml
check_pods_running

kubectl apply -f 08_dhcp_ddi_new.yml
check_pods_running

kubectl apply -f k8s-dashboard-http-deploy.yaml
check_pods_running



log "部署完成。"
