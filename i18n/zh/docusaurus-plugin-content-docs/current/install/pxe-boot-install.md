---
sidebar_position: 4
sidebar_label: PXE 引导安装
title: ""
keywords:
  - Harvester
  - harvester
  - Rancher
  - rancher
  - 安装 Harvester
  - 安装 Harvester
  - Harvester 安装
  - PXE 引导安装
Description: 从 `0.2.0` 开始，Harvester 可以自动安装。本文提供使用 pxe 引导进行自动安装的示例。
---

# PXE 引导安装

从 `0.2.0` 开始，Harvester 可以自动安装。本文提供使用 pxe 引导进行自动安装的示例。

我们建议使用 [iPXE](https://ipxe.org/) 进行网络启动。iPXE 比传统 pxe 引导程序功能更多，而且与现代网卡更匹配。如果你的网卡不支持 iPXE 固件，你可以先从 TFTP 服务器加载 iPXE 固件镜像。

如果需要获得 iPXE 脚本示例，请参见 [Harvester iPXE 示例](https://github.com/harvester/ipxe-examples)。

## 前提

:::info

节点至少需要有 **8 GB** 的内存，来让安装程序将整个 ISO 文件加载到 tmpfs。

:::

## 准备 HTTP 服务器

需要 HTTP 服务器来提供引导文件。
假设 NGINX HTTP 服务器的 IP 是 `10.100.0.10`，它提供 `/usr/share/nginx/html/` 目录，路径为 `http://10.100.0.10/`。

## 准备引导文件

- 从 [Harvester 发布页面](https://github.com/harvester/harvester/releases)下载所需文件：
   - ISO：`harvester-<version>-amd64.iso`
   - 内核：`harvester-<version>-vmlinuz-amd64`
   - initrd：`harvester-<version>-initrd-amd64`
   - rootfs squashfs 镜像：`harvester-<version>-rootfs-amd64.squashfs`

- 提供文件。

   将下载的文件复制或移动到适当的位置，以便通过 HTTP 服务器下载它们。例如：

   ```
   sudo mkdir -p /usr/share/nginx/html/harvester/
   sudo cp /path/to/harvester-<version>-amd64.iso /usr/share/nginx/html/harvester/
   sudo cp /path/to/harvester-<version>-vmlinuz-amd64 /usr/share/nginx/html/harvester/
   sudo cp /path/to/harvester-<version>-initrd-amd64 /usr/share/nginx/html/harvester/
   sudo cp /path/to/harvester-<version>-rootfs-amd64.squashfs /usr/share/nginx/html/harvester/
   ```

## 准备 iPXE 引导脚本

使用以下两种模式执行自动安装：

- `CREATE`：安装一个节点来构建一个初始的 Harvester 集群。
- `JOIN`：安装一个节点来加入现有的 Harvester 集群。


### CREATE 模式

:::caution

**安全风险**：下面的配置文件包含应保密的凭证。请不要公开配置文件。

:::

为 `CREATE` 模式创建一个名为 `config-create.yaml` 的 [Harvester 配置文件](./harvester-configuration.md)。按照需要修改值：

```YAML
# cat /usr/share/nginx/html/harvester/config-create.yaml
scheme_version: 1
token: token # 替换为所需的 token
os:
  hostname: node1 # 设置主机名如果 DHCP 服务器提供主机名，可以省略
  ssh_authorized_keys:
  - ssh-rsa ... # 替换为你的公钥
  password: p@ssword     # 替换为你的密码
  ntp_servers:
  - 0.suse.pool.ntp.org
  - 1.suse.pool.ntp.org
install:
  mode: create
  management_interface: 从 v1.1.0 开始可用
    interfaces:
      - name: ens5
    default_route: true
    method: dhcp
    bond_options:
      mode: balance-tlb
      miimon: 100
  device: /dev/sda # 目标磁盘
#  data_disk: /dev/sdb # 建议使用单独的磁盘来存储 VM 数据
  iso_url: http://10.100.0.10/harvester/harvester-<version>-amd64.iso
#  tty: ttyS1,115200n8   # 用于没有 VGA 控制台的主机

  vip: 10.100.0.99        # 访问 Harvester GUI 的 VIP。请确保 IP 可用
  vip_mode: static        # 或者 dhcp。详情请查看配置文件。
#  vip_hw_addr: 52:54:00:ec:0e:0b   # 如果 vip_mode 设为 static，则留空
```

对于需要使用 `CREATE` 模式安装的节点，以下是使用上述配置启动内核的 iPXE 脚本：

```
#!ipxe
kernel harvester-<version>-vmlinuz ip=dhcp net.ifnames=1 rd.cos.disable rd.noverifyssl console=tty1 root=live:http://10.100.0.10/harvester/rootfs.squashfs harvester.install.automatic=true harvester.install.config_url=http://10.100.0.10/harvester/config-create.yaml
initrd harvester-<version>-initrd
boot
```

以上假设 iPXE 脚本存储在 `/usr/share/nginx/html/harvester/ipxe-create` 中。

:::note

如果你有多个网络接口，你可以利用 dracut 的 `ip=` 参数来指定启动界面以及 dracut 支持的其他网络配置（例如，`ip=eth1:dhcp`）。
有关详细信息，请参阅 [`man dracut.cmdline`](https://man7.org/linux/man-pages/man7/dracut.cmdline.7.html)。

使用 `ip=` 参数仅指定启动界面，因为我们仅支持**一个 `ip=` 参数**。

:::

### JOIN 模式

:::caution

**安全风险**：下面的配置文件包含应保密的凭证。请不要公开配置文件。

:::

为 `JOIN` 模式创建一个名为 `config-join.yaml` 的 [Harvester 配置文件](./harvester-configuration.md)。按照需要修改值：

```YAML
# cat /usr/share/nginx/html/harvester/config-join.yaml
scheme_version: 1
server_url: https://10.100.0.99:443  # "CREATE" 中配置的 VIP
token: token
os:
  hostname: node2
  ssh_authorized_keys:
    - ssh-rsa ... # 替换为你的公钥
  password: p@ssword     # 替换为你的密码
  dns_nameservers:
  - 1.1.1.1
  - 8.8.8.8
install:
  mode: join
  management_interface: 从 v1.1.0 开始可用
    interfaces:
      - name: ens5
    default_route: true
    method: dhcp
    bond_options:
      mode: balance-tlb
      miimon: 100
  device: /dev/sda # 目标磁盘
#  data_disk: /dev/sdb # 建议使用单独的磁盘来存储 VM 数据
  iso_url: http://10.100.0.10/harvester/harvester-<version>-amd64.iso
#  tty: ttyS1,115200n8   # 用于没有 VGA 控制台的主机
```

注意 `mode` 是 `join` 并且需要提供 `server_url`。

对于需要使用 `JOIN` 模式安装的节点，以下是使用上述配置启动内核的 iPXE 脚本：

```
#!ipxe
kernel harvester-<version>-vmlinuz ip=dhcp net.ifnames=1 rd.cos.disable rd.noverifyssl console=tty1 root=live:http://10.100.0.10/harvester/rootfs.squashfs harvester.install.automatic=true harvester.install.config_url=http://10.100.0.10/harvester/config-join.yaml
initrd harvester-<version>-initrd
boot
```

以上假设 iPXE 脚本存储在 `/usr/share/nginx/html/harvester/ipxe-join` 中。


## DHCP 服务器配置

以下是如何配置 ISC DHCP 服务器以提供 iPXE 脚本的示例：

```sh
option architecture-type code 93 = unsigned integer 16;

subnet 10.100.0.0 netmask 255.255.255.0 {
	option routers 10.100.0.10;
        option domain-name-servers 192.168.2.1;
	range 10.100.0.100 10.100.0.253;
}

group {
  # 创建组
  if exists user-class and option user-class = "iPXE" {
    # ipxe 引导
    if option architecture-type = 00:07 {
      filename "http://10.100.0.10/harvester/ipxe-create-efi";
    } else {
      filename "http://10.100.0.10/harvester/ipxe-create";
    }
  } else {
    # pxe 引导
    if option architecture-type = 00:07 {
      # UEFI
      filename "ipxe.efi";
    } else {
      # Non-UEFI
      filename "undionly.kpxe";
    }
  }

  host node1 { hardware ethernet 52:54:00:6b:13:e2; }
}


group {
  # join group
  if exists user-class and option user-class = "iPXE" {
    # ipxe 引导
    if option architecture-type = 00:07 {
      filename "http://10.100.0.10/harvester/ipxe-join-efi";
    } else {
      filename "http://10.100.0.10/harvester/ipxe-join";
    }
  } else {
    # pxe 引导
    if option architecture-type = 00:07 {
      # UEFI
      filename "ipxe.efi";
    } else {
      # Non-UEFI
      filename "undionly.kpxe";
    }
  }

  host node2 { hardware ethernet 52:54:00:69:d5:92; }
}
```

配置文件声明了一个子网和两个组。第一组用于主机使用 `CREATE` 模式引导，另一组用于 `JOIN` 模式。默认情况下会选择 iPXE 路径。但是，如果有 PXE 客户端，它会根据客户端架构提供 iPXE 镜像。请先准备这些镜像以及 TFTP 服务器。

## Harvester 配置

有关 Harvester 配置的详情，请参见 [Harvester 配置](./harvester-configuration.md)。

默认情况下，第一个节点将是集群的管理节点。当有 3 个节点时，首先添加的另外 2 个节点会自动提升为管理节点，从而形成 HA 集群。

如果你想提升其它地区的管理节点，你可以在 [os.labels](./harvester-configuration.md#oslabels) 中添加节点标签 `topology.kubernetes.io/zone`。在这种情况下，至少需要三个不同的地区。

用户也可以通过内核参数提供配置。例如，如果你需要指定 `CREATE` 安装模式，你可以在启动时传入 `harvester.install.mode=create` 内核参数。通过内核参数传入的值比在配置文件中指定的值具有更高的优先级。

## UEFI HTTP 启动支持

UEFI 固件支持从 HTTP 服务器加载启动镜像。本节演示了如何使用 UEFI 的 HTTP 启动来加载 iPXE 程序并执行自动安装。

### 提供 iPXE 程序

从 http://boot.ipxe.org/ipxe.efi 下载 iPXE UEFI 程序，并确保 `ipxe.efi` 可以从 HTTP 服务器下载。例如：

```bash
cd /usr/share/nginx/html/harvester/
wget http://boot.ipxe.org/ipxe.efi
```

现在，你可以从 http://10.100.0.10/harvester/ipxe.efi 本地下载文件。

### DHCP 服务器配置

如果用户计划通过先获得一个动态 IP 来使用 UEFI HTTP 启动功能，DHCP 服务器在看到这样的请求时需要提供 iPXE 程序的 URL。下面是一个更新的 ISC DHCP 服务器组的示例：

```sh
group {
  # 创建组
  if exists user-class and option user-class = "iPXE" {
    # ipxe 引导
    if option architecture-type = 00:07 {
      filename "http://10.100.0.10/harvester/ipxe-create-efi";
    } else {
      filename "http://10.100.0.10/harvester/ipxe-create";
    }
  } elsif substring (option vendor-class-identifier, 0, 10) = "HTTPClient" {
    # UEFI HTTP 启动
    option vendor-class-identifier "HTTPClient";
    filename "http://10.100.0.10/harvester/ipxe.efi";
  } else {
    # pxe 引导
    if option architecture-type = 00:07 {
      # UEFI
      filename "ipxe.efi";
    } else {
      # Non-UEFI
      filename "undionly.kpxe";
    }
  }

  host node1 { hardware ethernet 52:54:00:6b:13:e2; }
}
```

`elsif substring` 语句是新的，它在看到 UEFI HTTP 启动 DHCP 请求时提供 `http://10.100.0.10/harvester/ipxe.efi`。在客户端获取 iPXE 程序并运行后，iPXE 程序将再次发送 DHCP 请求，并从 URL `http://10.100.0.10/harvester/ipxe-create-efi` 加载 iPXE 脚本。

### 用于 UEFI 启动的 iPXE 脚本

在内核参数中指定 UEFI 启动的 initrd 镜像是必须的。以下示例是为 `CREATE` 模式更新的 iPXE 脚本：

```
#!ipxe
kernel harvester-<version>-vmlinuz initrd=harvester-<version>-initrd ip=dhcp net.ifnames=1 rd.cos.disable rd.noverifyssl console=tty1 root=live:http://10.100.0.10/harvester/rootfs.squashfs harvester.install.automatic=true harvester.install.config_url=http://10.100.0.10/harvester/config-create.yaml
initrd harvester-<version>-initrd
boot
```

`initrd=harvester-<version>-initrd` 参数是必须的。

## 实用的内核参数

除了 Harvester 配置之外，你还可以指定用于其他场景的内核参数。
另请参见 [dracut.cmdline(7)](https://man7.org/linux/man-pages/man7/dracut.cmdline.7.html)。

### `ip=dhcp`

如果你有多个网络接口，你可以添加 `ip=dhcp` 参数，以便通过所有接口从 DHCP 服务器获取 IP。

### `rd.net.dhcp.retry=<cnt>`

如果你未能从 DHCP 服务器获取 IP，iPXE 引导将会失败。你可以添加参数 `rd.net.dhcp.retry=<cnt>`，从而重试 DHCP 请求 `<cnt>` 次。
