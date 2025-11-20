# Kubernetes HA Cluster Deployment with Ansible

生产级别的 Kubernetes 高可用集群部署 Playbook，基于 CentOS 7.9 和 containerd。

## 架构特点

- **高可用设计**：3 个 master 节点 + N 个 worker 节点
- **API Server 代理**：每个节点上运行 nginx 静态 Pod，反向代理 master 节点的 API Server
- **容器运行时**：containerd
- **网络插件**：Flannel 或 Calico CNI
- **系统要求**：CentOS 7.9

## 项目结构

```
.
├── inventory/              # 清单文件
│   └── hosts.ini          # 主机配置
├── group_vars/            # 组变量
│   ├── all.yml            # 全局变量
│   ├── masters.yml        # master 组变量
│   └── workers.yml        # worker 组变量
├── roles/                 # Ansible 角色
│   ├── system-init/       # 系统初始化
│   ├── containerd/        # containerd 安装
│   ├── kubernetes-install/# Kubernetes 组件安装
│   ├── master-init/       # master 节点初始化
│   ├── cni-plugin/        # CNI 插件安装
│   ├── worker-join/       # worker 节点加入
│   ├── nginx-proxy/       # nginx 代理配置
│   ├── cleanup/           # 集群清理
│   └── cert-renewal/      # 证书更新
├── site.yml               # 主 playbook
├── add-worker.yml         # 添加 worker 节点剧本
├── renew-cert.yml         # 证书更新剧本
├── destroy.yml            # 快速销毁剧本
├── destroy-detailed.yml   # 详细销毁剧本
├── ansible.cfg            # Ansible 配置
├── README.md              # 本文件
├── DESTROY.md             # 销毁指南
└── update-kubeadm-cert.sh # 证书更新脚本
```

## 使用说明

### 1. 环境准备

编辑 `inventory/hosts.ini`，配置你的主机信息：

```ini
[masters]
master1 ansible_host=192.168.200.10 ansible_user=root
master2 ansible_host=192.168.200.11 ansible_user=root
master3 ansible_host=192.168.200.12 ansible_user=root

[workers]
worker1 ansible_host=192.168.200.20 ansible_user=root

```

### 2. 配置变量

根据需要编辑 `group_vars/all.yml`：

```yaml
k8s_version: "1.28.0"
containerd_version: "1.6.33"
cni_plugin: "calico"  # 可选: flannel 或 calico
kubernetes_pod_subnet: "10.244.0.0/16"
kubernetes_service_subnet: "10.96.0.0/12"
```

**重要**：确保 `kubernetes_pod_subnet` 与 CNI 插件配置一致（特别是 Calico）。

### 3. 执行部署

```bash
# 检查连接
ansible all -m ping

# 执行完整部署
ansible-playbook site.yml

# 执行特定步骤
ansible-playbook site.yml --tags "system-init"
ansible-playbook site.yml --tags "containerd"
ansible-playbook site.yml --tags "master-init"
ansible-playbook site.yml --tags "cni-plugin"
ansible-playbook site.yml --tags "worker-join"
ansible-playbook site.yml --tags "nginx-proxy"

# 执行初始化步骤
ansible-playbook site.yml --tags "init"
```

### 5. 添加新的 Worker 节点

编辑 `inventory/hosts.ini`，在 `[new_workers]` 组中添加新节点：

```ini
[new_workers]
worker2 ansible_host=192.168.200.21 ansible_user=root
worker3 ansible_host=192.168.200.22 ansible_user=root
```

然后执行添加 worker 节点剧本：

```bash
ansible-playbook add-worker.yml
```

### 6. 延长 Kubernetes 证书有效期

证书默认有效期为 1 年。使用以下命令延长到 100 年：

```bash
ansible-playbook renew-cert.yml
```

此命令会：
- 自动上传证书更新脚本到所有 master 节点
- 延长所有证书有效期至 10 年（36500 天）
- 更新 etcd、apiserver、controller-manager、scheduler 等所有组件的证书
- 自动重启相关服务

### 4. 验证集群

```bash
# 在 master 节点上执行
kubectl get nodes
kubectl get pods -A
```

## 高可用架构说明

### nginx 反向代理

- 每个节点（master 和 worker）都运行一个 nginx 静态 Pod
- nginx 监听本地 6443 端口，反向代理所有 master 节点的 API Server
- 应用通过 localhost:6443 访问 API Server，实现高可用

### 静态 Pod 配置

nginx 静态 Pod 清单位于 `/etc/kubernetes/manifests/nginx-proxy.yaml`，kubelet 会自动启动和监控。

## 故障排查

### 查看 nginx 日志

```bash
docker logs $(docker ps | grep nginx-proxy | awk '{print $1}')
```

### 检查 kubelet 状态

```bash
systemctl status kubelet
journalctl -u kubelet -f
```

### 验证 API Server 连接

```bash
curl -k https://localhost:6443/api/v1
```

## CNI 插件说明

### Flannel

- 简单易用，适合小规模集群
- 使用 VXLAN 或 host-gw 作为后端
- 配置文件：自动从上游下载

### Calico

- 功能强大，支持网络策略
- 使用 BGP 或 VXLAN 作为后端
- **重要**：Calico IPPool CIDR 必须与 `kubernetes_pod_subnet` 一致
- 配置文件：使用 Jinja2 模板自动生成，确保与 pod 网段同步

## 集群销毁

详见 [DESTROY.md](DESTROY.md) 文档。

快速销毁命令：

```bash
# 保留 CNI 配置，可重新部署
ansible-playbook destroy.yml

# 完全销毁，包括 CNI 配置
ansible-playbook destroy.yml -e "cleanup_cni_config=true"
```

## 注意事项

1. 确保所有节点网络互通
2. 关闭防火墙或配置相应规则
3. 确保 SSH 密钥配置正确
4. 部署前备份重要数据
5. 生产环境建议使用外部 etcd 集群
6. 使用 Calico 时，确保 pod 网段配置正确
7. 证书更新前建议备份 `/etc/kubernetes` 目录

## 参考文档

- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [kubeadm 部署指南](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [containerd 文档](https://containerd.io/)
- [Calico 文档](https://docs.tigera.io/calico/latest/)
- [Flannel 文档](https://github.com/coreos/flannel)
