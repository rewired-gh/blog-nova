+++
author = 'Rewired'
date = '2023-12-04'
title = '为 UFW 配置 DNAT 端口双向转发'
tags = [
  '运维'
]
menu = "main"
+++

## 前情提要

某个妙妙服务需要将一些端口双向转发到另一个端口，一般来说可以使用 iptables 来完成路由配置，比如：

```
# IPv4
iptables -t nat -A PREROUTING -i eth0 -p udp --dport 20000:50000 -j DNAT --to-destination :443
# IPv6
ip6tables -t nat -A PREROUTING -i eth0 -p udp --dport 20000:50000 -j DNAT --to-destination :443
```

可惜的是，在配置了 UFW 的服务器上，这样的配置是不生效的（以前可能可用）。所以，需要按 UFW 的方法去配置，同时带来的好处是路由可以持久化。

## 操作步骤

### 1. 配置内核以允许端口转发

#### 方案一：使用 UFW 间接配置

在 `/etc/ufw/sysctl.conf` 文件中，将端口转发相关的配置项目去掉注释即可。这些项目通常列于这样一个说明的下方：

```
# Uncomment this to allow this host to route packets between interfaces
```

#### 方案二：使用 sysctl 配置

执行以下命令，这将会添加一个名为 `99-forward.conf` 的 sysctl 配置文件：

```
sudo tee /etc/sysctl.d/99-forward.conf >/dev/null <<EOT
net.ipv4.ip_forward = 1
net.ipv6.conf.default.forwarding = 1
net.ipv6.conf.all.forwarding = 1
EOT
```

然后执行 `sudo sysctl --system` 以重载配置。

### 2. 配置 UFW 路由

以这个妙妙服务的需求为例，需要在 `/etc/ufw/before.rules` 文件的末尾添加如下配置：

```
# port forwarding
*nat
:PREROUTING ACCEPT [0:0]
-A PREROUTING -i eth0 -p udp --dport 20000:50000 -j DNAT --to-destination :443
COMMIT
```

参数说明：
- `-i eth0`：针对 eth0 网卡
- `-p udp`：针对 UDP 协议
- `--dport 20000:50000`：入口端口针对 20000 到 50000
- `--to-destination :443`：出口端口针对 443

欲查看操作系统的网卡名称，可执行命令 `ip link`。

最后，执行命令 `systemctl restart ufw` 以重启 UFW。

## 参考资料

- <https://www.baeldung.com/linux/ufw-port-forward>
- <https://www.cyberciti.biz/faq/linux-list-network-interfaces-names-command>
- <https://www.ibm.com/docs/en/ztpf/2022?topic=collection-tuning-network-performance-your-linux-environment>