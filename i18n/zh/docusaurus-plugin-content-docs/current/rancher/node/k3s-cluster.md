---
sidebar_position: 4
sidebar_label: 创建 K3s Kubernetes 集群
title: ""
---

# 创建 K3s Kubernetes 集群

在 Rancher `2.6.3` 或以上的版本，你可以使用内置的 Harvester 主机驱动创建基于 Harvester 集群之上的 K3s Kubernetes 集群。

![k3s-cluster](/img/v1.1/rancher/rke2-k3s-node-driver.png)

:::note

- Harvester K3s 主机驱动处于**技术预览**阶段。
- Harvester 主机驱动需要 [VLAN 网络](../../networking/harvester-network.md#vlan-网络)。
- Harvester 主机驱动仅支持云服务镜像（Cloud Image）。

:::

### 创建你的云凭证

1. 单击 **☰ > Cluster Management**。
2. 单击 **Cloud Credentials**。
3. 单击 **Create**。
4. 单击 **Harvester**。
5. 输入你的云凭证名称。
6. 选择 **Imported Harvester** 或 **External Harvester**。
7. 单击 **Create**。

![create-harvester-cloud-credentials](/img/v1.1/rancher/create-cloud-credentials.png)

### 创建 K3s Kubernetes 集群

你可以通过 K3s 主机驱动从 **Cluster Management** 页面创建 K3s Kubernetes 集群。

1. 点击 **Clusters** 菜单。
2. 点击 **Create** 按钮。
3. 切换到 **RKE2/K3s**。
4. 选择 Harvester 主机驱动。
5. 选择 **Cloud Credential**。
6. 输入 **Cluster Name**（必须）。
7. 选择 **Namespace**（必须）。
8. 选择 **Image**（必须）。
9. 选择 **Network Name**（必须）。
10. 输入 **SSH User**（必须）。
11. 单击 **Create**。

![create-k3s-harvester-cluster](/img/v1.1/rancher/create-k3s-harvester-cluster.png)

#### 添加节点亲和性

_从 v1.0.3 + Rancher v2.6.7 起可用_

Harvester 主机驱动现在支持通过节点亲和性规则将一组主机调度到特定节点，这能提供高可用性并提高资源的利用率。

你可以在集群创建期间将节点亲和性添加到主机池中：

1. 单击 `Show Advanced` 按钮并单击 `Add Node Selector`：
   ![affinity-add-node-selector](/img/v1.1/rancher/affinity-rke2-add-node-selector.png)
2. 如果你希望调度程序仅在满足规则时调度主机，请将优先级设置为 `Required`。
3. 点击 `Add Rule` 指定节点亲和规则，例如，对于 [topology spread constraints](./node-driver.md#拓扑分布约束) 用例，你可以添加 `region` 和 `zone` 标签，如下：
   ```yaml
   key: topology.kubernetes.io/region
   operator: in list
   values: us-east-1
   ---
   key: topology.kubernetes.io/zone
   operator: in list
   values: us-east-1a
   ```
   ![affinity-add-rules](/img/v1.1/rancher/affinity-rke2-add-rules.png)
4. 点击 `Create` 保存节点模板。集群安装完成后，你可以查看其主机节点是否按照亲和性规则进行调度。


### 在离线环境中使用 Harvester K3s 主机驱动

K3s 配置依赖 `qemu-guest-agent` 包来获取虚拟机的 IP。

但是，你可能无法在离线环境中安装软件包。

你可以使用以下选项解决安装限制：

选项 1：使用安装了所需软件包的 VM 镜像。

选项 2：配置 **Show Advanced > User Data**，使 VM 能够通过 HTTP(S) 代理安装所需的包。

Harvester 节点模板中的`用户数据`示例：
```
#cloud-config
apt:
  http_proxy: http://192.168.0.1:3128
  https_proxy: http://192.168.0.1:3128
```
