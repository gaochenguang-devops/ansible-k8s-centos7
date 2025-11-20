# Kubernetes 集群销毁指南

本文档说明如何使用删除剧本销毁 Kubernetes 集群。

## 删除剧本文件

- `destroy.yml` - 简化版删除剧本，一步到位清理所有节点
- `destroy-detailed.yml` - 详细版删除剧本，分步骤清理 worker 和 master 节点

## 使用方法

### 1. 清理集群（保留 CNI 配置，可重新部署）

```bash
# 快速清理
ansible-playbook destroy.yml

# 分步清理
ansible-playbook destroy-detailed.yml
```

**特点：**
- ✅ 删除 Kubernetes 组件和配置
- ✅ 保留 CNI 配置文件（`/etc/cni/net.d`）
- ✅ 支持重新部署集群
- ✅ 节点可以快速重新加入集群

### 2. 完全销毁集群（包括 CNI 配置）

```bash
# 添加 cleanup_cni_config 变量
ansible-playbook destroy.yml -e "cleanup_cni_config=true"

# 或分步执行
ansible-playbook destroy-detailed.yml -e "cleanup_cni_config=true"
```

**特点：**
- ✅ 删除所有 Kubernetes 相关配置
- ✅ 删除 CNI 配置文件
- ✅ 节点回到初始状态
- ✅ 适合完全卸载或更换 CNI 插件

### 3. 仅清理特定节点

```bash
# 仅清理 worker 节点
ansible-playbook destroy.yml -l workers

# 仅清理 master 节点
ansible-playbook destroy.yml -l masters

# 仅清理特定节点
ansible-playbook destroy.yml -l k8s-master-1
```

## 销毁步骤说明

### Drain（排空）
- 将 worker 节点上的所有 Pod 迁移到其他节点
- 忽略 DaemonSet 和空目录数据

### Cleanup（清理）
- 停止 kubelet 服务
- 删除 Kubernetes 相关目录：
  - `/etc/kubernetes`
  - `/var/lib/kubelet`
  - `/var/lib/etcd`
  - `/etc/cni/net.d`
  - `/root/.kube`
- 清理 iptables 规则
- 删除网络接口（flannel.1, cni0, tunl0）
- 执行 kubeadm reset

### Final Cleanup（最终清理）
- 删除 nginx 代理配置
- 验证清理状态

## 注意事项

1. **备份重要数据**：销毁前请备份集群中的重要数据
2. **谨慎操作**：销毁操作不可逆，请确认无误后再执行
3. **网络隔离**：销毁后节点将与集群断开连接
4. **持久化存储**：如有 PVC，请先备份数据
5. **DNS 更新**：销毁后需要更新相关 DNS 记录
6. **Calico 清理**：使用 Calico 时，销毁脚本会自动清理 Calico 特定配置（tunl0 接口、iptables 规则、/var/lib/calico 目录）

## 销毁后恢复

销毁后如需重新部署集群，可以直接运行部署剧本：

```bash
ansible-playbook site.yml
```

## 故障排查

### 销毁失败

如果销毁过程中出现错误，可以：

1. 检查节点连接：
```bash
ansible all -m ping
```

2. 查看具体错误：
```bash
ansible-playbook destroy-detailed.yml -vvv
```

3. 手动清理：
```bash
# 在目标节点上手动执行
kubeadm reset -f
rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd /etc/cni/net.d
```

### 节点无法连接

如果节点无法通过 Ansible 连接，可以 SSH 到节点后手动执行清理命令。

## 相关命令

```bash
# 验证集群状态
kubectl get nodes
kubectl get pods -A

# 查看 kubelet 日志
journalctl -u kubelet -f

# 检查网络接口
ip link show

# 检查 iptables 规则
iptables -L -n

# 检查 Calico 状态
kubectl get pods -n calico-system
kubectl get ippool
kubectl get installation
```
