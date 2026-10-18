# FortiGate HA + Cumulus EVPN 实验手册

Oct 18, 2026 · @voight

## 实验目标与设计思路

这套拓扑同时验证两种网关模型：VPC 网段（VLAN 10）的网关是 leaf 上的分布式 anycast 网关；容器网段（VLAN 30）的网关在 FortiGate 上，交换机只做二层延伸。



- **Underlay**：四台 Cumulus 之间全部跑 eBGP unnumbered（RFC 5549，用 IPv6 链路本地地址承载 IPv4 路由）。互联链路不规划地址，只规划 loopback。
- **Overlay**：VXLAN-EVPN。两台 leaf 是 VTEP；两台 spine 组成 MLAG 对，兼做 border，对外用共享的 anycast VTEP 地址。
- **防火墙**：FG5 为主，FG6 为备，采用 FGCP 主备。每台 FG 的 port2、port3 组成 agg1，V 型交叉接到两台 spine；spine 侧给每台 FG 配一个 MLAG bond。
- **FW 对接**：在 VLAN 100 上用编号地址跑 eBGP（VRF-lite），FG 与两台 spine 各建一条邻居。FW 侧不用 unnumbered，因为对接点是 MLAG 上的 SVI，编号地址也更接近生产环境的常见做法。
- **容器网关**：VLAN 30 只做 L2VNI，所有交换机都不配 VLAN 30 的 SVI，网关 10.30.30.1 在 FG 的子接口上。

**VPC 访问容器的路径**（跨安全域，必须经过 FW）：VPC → leaf1 网关（VRF PROD）→ EVPN Type-5 路由 → spine → VLAN 100 → FG 策略检测 → FG 的 VLAN 30 子接口 → spine 桥接进 VNI 10030 → leaf → 容器。容器的回程以 FG 为网关，来回路径对称。

## 拓扑与接线

截图中左上的 cumulus 记为 spine1，右上的记为 spine2，cumulus1 为 leaf1，cumulus2 为 leaf2。

&#91;embedded content: 实验拓扑 · 2 台 FG、4 台 Cumulus、2 个终端\]

FG5 的两条线（实线）和 FG6 的两条线（虚线）分别落到两台 spine，在交换机侧各组成一个 MLAG bond；容器网段在交换机上只做二层，三层网关只在 FW 上。

| 本端 | 端口 | 对端 | 端口 | 用途 |
| --- | --- | --- | --- | --- |
| FG5 | port1 | FG6 | port1 | HA 心跳 |
| FG5 | port2 | spine1 | swp4 | agg1 成员 → bond\_fg5 |
| FG5 | port3 | spine2 | swp4 | agg1 成员 → bond\_fg5 |
| FG6 | port2 | spine1 | swp5 | agg1 成员 → bond\_fg6 |
| FG6 | port3 | spine2 | swp5 | agg1 成员 → bond\_fg6 |
| spine1 | swp3 | spine2 | swp3 | MLAG peerlink |
| spine1 | swp1 | leaf1 | swp1 | underlay，BGP unnumbered |
| spine1 | swp2 | leaf2 | swp1 | underlay，BGP unnumbered |
| spine2 | swp1 | leaf1 | swp2 | underlay，BGP unnumbered |
| spine2 | swp2 | leaf2 | swp2 | underlay，BGP unnumbered |
| leaf1 | swp4 | VPC | eth0 | access VLAN 10 |
| leaf1 | swp3 | 容器 | eth0 | access VLAN 30 |
| leaf2 | swp3 | 容器 | eth1 | access VLAN 30 |

leaf1 和 leaf2 之间没有 peerlink，所以容器的双归不能用 MLAG。第一阶段在容器内做 active-backup bond，交换机侧只需 access 口；EVPN 多归属（ESI）作为可选扩展。

## 地址与编号规划

互联链路没有地址，下表就是全部需要规划的编号。

| 设备 | ASN | Loopback / VTEP | 其他地址 |
| --- | --- | --- | --- |
| spine1 | 65001 | 10.255.0.1/32 | 共享 VTEP 10.255.0.100；vlan100 172.16.100.2/29 |
| spine2 | 65002 | 10.255.0.2/32 | 共享 VTEP 10.255.0.100；vlan100 172.16.100.3/29 |
| leaf1 | 65101 | 10.255.0.11/32 | vlan10 10.10.10.2/24，VRR 10.10.10.1 |
| leaf2 | 65102 | 10.255.0.12/32 | vlan10 10.10.10.3/24，VRR 10.10.10.1 |
| FG 集群 | 65500 | 无 | fw-in100 172.16.100.1/29；ctr30 10.30.30.1/24 |
| VPC | 无 | 无 | 10.10.10.10/24，网关 10.10.10.1 |
| 容器 | 无 | 无 | bond0 10.30.30.10/24，网关 10.30.30.1 |

| VLAN | VNI | VRF | 网关位置 | 存在于 |
| --- | --- | --- | --- | --- |
| 10 | 10010 | PROD | leaf（anycast VRR） | leaf1、leaf2 |
| 30 | 10030 | 无（纯二层） | FortiGate ctr30 | 全部四台交换机 |
| 100 | 无（本地） | PROD | 无，FW 互联段 | spine1、spine2（经 peerlink 互通） |
| L3VNI | 4001 | PROD | 无 | 全部四台交换机 |

两台 spine 用不同 ASN，并通过 peerlink.4094 互建 eBGP，这样某台 spine 丢失全部下行时仍有路径。VRR 虚拟 MAC 统一为 00:00:5E:00:01:01，MLAG 系统 MAC 为 44:38:39:BE:EF:01。

## Spine（兼 border）配置

以 Cumulus Linux 5.x 的 NVUE 为准，下面是 spine1 的配置。spine2 只需改 hostname、loopback、ASN、MLAG 优先级、vlan100 地址和 VTEP 源地址，差异列在代码块之后。命令如果报错，用 Tab 补全核对当前版本的语法。

```bash
# 基础
nv set system hostname spine1
nv set interface lo ip address 10.255.0.1/32
nv set interface swp1-5 type swp

# MLAG：peerlink + 两个 FW bond
nv set interface peerlink bond member swp3
nv set interface peerlink type peerlink
nv set mlag mac-address 44:38:39:BE:EF:01
nv set mlag backup 10.255.0.2 vrf default
nv set mlag peer-ip linklocal
nv set mlag priority 1000
nv set interface bond_fg5 bond member swp4
nv set interface bond_fg5 bond mlag id 5
nv set interface bond_fg6 bond member swp5
nv set interface bond_fg6 bond mlag id 6
nv set interface bond_fg5,bond_fg6 bridge domain br_default vlan 30,100

# VLAN / VNI
nv set bridge domain br_default vlan 30 vni 10030
nv set bridge domain br_default vlan 100

# VRF PROD 与 FW 互联 SVI（不配 vlan30 的 SVI）
nv set vrf PROD evpn vni 4001
nv set interface vlan100 ip address 172.16.100.2/29
nv set interface vlan100 ip vrf PROD
nv set interface vlan100 link mtu 1500

# VXLAN / EVPN
nv set nve vxlan enable on
nv set nve vxlan source address 10.255.0.1
nv set nve vxlan mlag shared-address 10.255.0.100
nv set nve vxlan arp-nd-suppress on
nv set evpn enable on

# Underlay：BGP unnumbered
nv set router bgp autonomous-system 65001
nv set router bgp router-id 10.255.0.1
nv set router bgp graceful-restart mode helper-only
nv set vrf default router bgp neighbor swp1 remote-as external
nv set vrf default router bgp neighbor swp2 remote-as external
nv set vrf default router bgp neighbor peerlink.4094 remote-as external
nv set vrf default router bgp address-family ipv4-unicast network 10.255.0.1/32
nv set vrf default router bgp address-family ipv4-unicast network 10.255.0.100/32
nv set vrf default router bgp address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor swp1 address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor swp2 address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor peerlink.4094 address-family l2vpn-evpn enable on

# 发往 FW 的路由只放行 10.10.0.0/16 内 /24 及更短的前缀，过滤 EVPN 主机路由
nv set router policy prefix-list PL-TO-FW type ipv4
nv set router policy prefix-list PL-TO-FW rule 10 match 10.10.0.0/16 max-prefix-len 24
nv set router policy prefix-list PL-TO-FW rule 10 action permit
nv set router policy route-map RM-TO-FW rule 10 match ip-prefix-list PL-TO-FW
nv set router policy route-map RM-TO-FW rule 10 action permit

# VRF PROD 内与 FW 的 eBGP（VRF-lite），并把 FW 的路由导出为 Type-5
nv set vrf PROD router bgp autonomous-system 65001
nv set vrf PROD router bgp router-id 10.255.0.1
nv set vrf PROD router bgp neighbor 172.16.100.1 remote-as 65500
nv set vrf PROD router bgp neighbor 172.16.100.1 address-family ipv4-unicast policy outbound route-map RM-TO-FW
nv set vrf PROD router bgp address-family ipv4-unicast redistribute connected enable on
nv set vrf PROD router bgp address-family ipv4-unicast route-export to-evpn enable on

nv config apply
nv config save
```

spine2 的差异：

| 项 | spine1 | spine2 |
| --- | --- | --- |
| hostname | spine1 | spine2 |
| loopback / router-id / VTEP 源地址 | 10.255.0.1 | 10.255.0.2 |
| ASN（default 和 PROD 两处） | 65001 | 65002 |
| mlag backup | 10.255.0.2 | 10.255.0.1 |
| mlag priority | 1000 | 2000 |
| vlan100 地址 | 172.16.100.2/29 | 172.16.100.3/29 |
| network 宣告 | 10.255.0.1/32、10.255.0.100/32 | 10.255.0.2/32、10.255.0.100/32 |

vlan100 的 MTU 设为 1500，与 FG 子接口保持一致。否则 spine 会按 9216 计算 BGP 的 TCP MSS，较大的 UPDATE 报文到 FG 时会被丢弃，表现为邻居建立后反复重置。

## Leaf 配置

下面是 leaf1 的配置。leaf2 没有接 VPC，所以去掉 swp4 那一行；同时把 hostname、loopback、ASN、vlan10 实地址和 VTEP 源地址换成 leaf2 的值。

```bash
nv set system hostname leaf1
nv set interface lo ip address 10.255.0.11/32
nv set interface swp1-4 type swp

# 接入口
nv set interface swp4 bridge domain br_default access 10   # VPC
nv set interface swp3 bridge domain br_default access 30   # 容器 eth0

# VLAN / VNI
nv set bridge domain br_default vlan 10 vni 10010
nv set bridge domain br_default vlan 30 vni 10030

# VRF PROD + 分布式 anycast 网关（只给 vlan10 配网关）
nv set vrf PROD evpn vni 4001
nv set interface vlan10 ip vrf PROD
nv set interface vlan10 ip address 10.10.10.2/24
nv set interface vlan10 ip vrr address 10.10.10.1/24
nv set interface vlan10 ip vrr mac-address 00:00:5E:00:01:01
nv set interface vlan10 ip vrr state up

# VXLAN / EVPN
nv set nve vxlan enable on
nv set nve vxlan source address 10.255.0.11
nv set nve vxlan arp-nd-suppress on
nv set evpn enable on

# Underlay：BGP unnumbered，向两台 spine 建邻居
nv set router bgp autonomous-system 65101
nv set router bgp router-id 10.255.0.11
nv set router bgp graceful-restart mode helper-only
nv set vrf default router bgp neighbor swp1 remote-as external
nv set vrf default router bgp neighbor swp2 remote-as external
nv set vrf default router bgp address-family ipv4-unicast network 10.255.0.11/32
nv set vrf default router bgp address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor swp1 address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor swp2 address-family l2vpn-evpn enable on

# VRF PROD：把 vlan10 网段以 Type-5 发布
nv set vrf PROD router bgp autonomous-system 65101
nv set vrf PROD router bgp router-id 10.255.0.11
nv set vrf PROD router bgp address-family ipv4-unicast redistribute connected enable on
nv set vrf PROD router bgp address-family ipv4-unicast route-export to-evpn enable on

nv config apply
nv config save
```

leaf2 完整配置（不接 VPC，只接容器 eth1；vlan10 仍然配置，保证 VPC 网段在任意 leaf 上都有网关）：

```bash
nv set system hostname leaf2
nv set interface lo ip address 10.255.0.12/32
nv set interface swp1-3 type swp

# 接入口
nv set interface swp3 bridge domain br_default access 30   # 容器 eth1

# VLAN / VNI
nv set bridge domain br_default vlan 10 vni 10010
nv set bridge domain br_default vlan 30 vni 10030

# VRF PROD + 分布式 anycast 网关（与 leaf1 相同的 VRR 地址和 MAC）
nv set vrf PROD evpn vni 4001
nv set interface vlan10 ip vrf PROD
nv set interface vlan10 ip address 10.10.10.3/24
nv set interface vlan10 ip vrr address 10.10.10.1/24
nv set interface vlan10 ip vrr mac-address 00:00:5E:00:01:01
nv set interface vlan10 ip vrr state up

# VXLAN / EVPN
nv set nve vxlan enable on
nv set nve vxlan source address 10.255.0.12
nv set nve vxlan arp-nd-suppress on
nv set evpn enable on

# Underlay：BGP unnumbered（swp1 → spine1 swp2，swp2 → spine2 swp2）
nv set router bgp autonomous-system 65102
nv set router bgp router-id 10.255.0.12
nv set router bgp graceful-restart mode helper-only
nv set vrf default router bgp neighbor swp1 remote-as external
nv set vrf default router bgp neighbor swp2 remote-as external
nv set vrf default router bgp address-family ipv4-unicast network 10.255.0.12/32
nv set vrf default router bgp address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor swp1 address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor swp2 address-family l2vpn-evpn enable on

# VRF PROD
nv set vrf PROD router bgp autonomous-system 65102
nv set vrf PROD router bgp router-id 10.255.0.12
nv set vrf PROD router bgp address-family ipv4-unicast redistribute connected enable on
nv set vrf PROD router bgp address-family ipv4-unicast route-export to-evpn enable on

nv config apply
nv config save
```

| 项 | leaf1 | leaf2 |
| --- | --- | --- |
| hostname | leaf1 | leaf2 |
| loopback / router-id / VTEP 源地址 | 10.255.0.11 | 10.255.0.12 |
| ASN（default 和 PROD 两处） | 65101 | 65102 |
| vlan10 实地址 | 10.10.10.2/24 | 10.10.10.3/24 |
| swp4 | VPC，access 10 | 不配置 |
| swp3 | 容器 eth0，access 30 | 容器 eth1，access 30 |
|  |  |  |

VLAN 30 在 leaf 上没有 SVI，ARP 抑制对它不生效，ARP 走头端复制泛洪。这是正常现象，在这个规模下不影响实验。

## FortiGate HA 配置

两台 FG 先各自单独配置 hostname 和 HA。HA 组建完成后，其余配置只在主机 FG5 上做，系统会自动同步到 FG6。两台的差异只有 hostname 和 priority。

```text
# ① 两台各自执行（FG6：hostname FG6，priority 100）
config system global
    set hostname "FG5"
end
config system ha
    set group-id 10
    set group-name "LAB-HA"
    set mode a-p
    set password LabHa2026
    set hbdev "port1" 50
    set session-pickup enable
    set session-pickup-connectionless enable
    set override disable
    set priority 200
end

# ② 集群组建后只在 FG5 上执行
config system interface
    edit "agg1"
        set vdom "root"
        set type aggregate
        set member "port2" "port3"
        set lacp-mode active
    next
    edit "fw-in100"
        set vdom "root"
        set interface "agg1"
        set vlanid 100
        set ip 172.16.100.1 255.255.255.248
        set allowaccess ping
    next
    edit "ctr30"
        set vdom "root"
        set interface "agg1"
        set vlanid 30
        set ip 10.30.30.1 255.255.255.0
        set allowaccess ping
    next
end
config system ha
    set monitor "agg1"
end

config router bgp
    set as 65500
    set router-id 172.16.100.1
    set ebgp-multipath enable
    set graceful-restart enable
    config neighbor
        edit "172.16.100.2"
            set remote-as 65001
        next
        edit "172.16.100.3"
            set remote-as 65002
        next
    end
    config network
        edit 1
            set prefix 10.30.30.0 255.255.255.0
        next
    end
end

config firewall address
    edit "NET-VPC"
        set subnet 10.10.10.0 255.255.255.0
    next
    edit "NET-CTR"
        set subnet 10.30.30.0 255.255.255.0
    next
end
config firewall policy
    edit 1
        set name "VPC-to-CTR"
        set srcintf "fw-in100"
        set dstintf "ctr30"
        set srcaddr "NET-VPC"
        set dstaddr "NET-CTR"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 2
        set name "CTR-to-VPC-ping"
        set srcintf "ctr30"
        set dstintf "fw-in100"
        set srcaddr "NET-CTR"
        set dstaddr "NET-VPC"
        set action accept
        set schedule "always"
        set service "PING"
        set logtraffic all
    next
end
```

- 策略默认不做 NAT，所以容器看到的源地址就是 VPC 的真实地址。
- 策略 2 只放行 ping，用来演示“容器主动访问 VPC 只允许 ICMP”。也可以先删掉它，观察回包是否仍然正常：回包属于策略 1 的会话，不需要反向策略。
- 3 个接口里 port1 用作心跳，port2、port3 用作数据口，没有剩余的管理口。管理通过 EVE 控制台进行，进入备机用 `execute ha manage 1 admin`。

FortiOS 默认 `cfg-save automatic`，每次 `end` 后自动写入启动配置，不需要手动保存；HA 集群中主机的配置会同步到备机，可用 `diagnose sys ha checksum cluster` 确认两台一致。

## 终端配置

VPC：

```text
ip 10.10.10.10/24 10.10.10.1
save
```

容器节点（Linux，需要 root 权限）：

```bash
ip link add bond0 type bond mode active-backup miimon 100
ip addr flush dev eth0; ip addr flush dev eth1
ip link set eth0 down; ip link set eth0 master bond0
ip link set eth1 down; ip link set eth1 master bond0
ip link set bond0 up
ip addr add 10.30.30.10/24 dev bond0
ip route replace default via 10.30.30.1
cat /proc/net/bonding/bond0   # 确认当前活动口
```

如果镜像不支持 bond，先只用 eth0 配置 10.30.30.10/24，等主路径调通后再做双归。

VPC 的 `save` 会把地址写入节点的启动配置。容器内的 bond 和地址在重启后会丢失，用下面的启动脚本恢复。脚本可以重复执行：它会等待 EVE 把 eth0、eth1 接入，bond 不可用时退回到只用 eth0。

**第一步：在容器里写入脚本**

```bash
cat > /usr/local/bin/ctr-net.sh <<'EOF'
#!/bin/sh
# 网络初始化：两块网卡组成 active-backup bond，可重复执行
IF1=eth0            # 接 leaf1 swp3，网卡名不同就改这里（如 ens3）
IF2=eth1            # 接 leaf2 swp3（如 ens4）
IP=10.30.30.10/24
GW=10.30.30.1

# 等待网卡出现，最多 30 秒
for i in $(seq 1 30); do
  [ -e /sys/class/net/$IF1 ] && [ -e /sys/class/net/$IF2 ] && break
  sleep 1
done

modprobe bonding 2>/dev/null
if ip link show bond0 >/dev/null 2>&1 || \
   ip link add bond0 type bond mode active-backup miimon 100 2>/dev/null; then
  for IF in $IF1 $IF2; do
    [ -e /sys/class/net/$IF ] || continue
    if [ ! -e /sys/class/net/$IF/master ]; then
      ip addr flush dev $IF
      ip link set $IF down
      ip link set $IF master bond0
    fi
  done
  ip link set bond0 up
  DEV=bond0
else
  echo "bond 不可用，退回只用 $IF1" >&2
  ip link set $IF1 up
  DEV=$IF1
fi

ip addr replace $IP dev $DEV
ip route replace default via $GW dev $DEV
echo "网络就绪：$DEV $IP，网关 $GW"
EOF
chmod +x /usr/local/bin/ctr-net.sh
/usr/local/bin/ctr-net.sh
```

**第二步：开机自动运行**（在节点自身的 Linux 里配置，不依赖 docker）

先执行 `cat /proc/1/comm` 判断节点的 init 方式：输出 `systemd` 用方式 A，其他输出用方式 B。

方式 A，systemd 服务：

```bash
cat > /etc/systemd/system/ctr-net.service <<'EOF'
[Unit]
Description=Lab bond network
After=network-pre.target
Wants=network-pre.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/ctr-net.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now ctr-net.service
systemctl status ctr-net.service   # 确认 active (exited)
```

方式 B，没有 systemd 时，按顺序选第一个可用的：

```bash
# 1. cron 的 @reboot
(crontab -l 2>/dev/null; echo "@reboot /usr/local/bin/ctr-net.sh") | crontab -

# 2. 没有 cron 时用 rc.local
[ -f /etc/rc.local ] || printf '#!/bin/sh\nexit 0\n' > /etc/rc.local
sed -i '/^exit 0/i /usr/local/bin/ctr-net.sh' /etc/rc.local
chmod +x /etc/rc.local
```

- 如果系统里的 NetworkManager、netplan 或 ifupdown 在用 DHCP 管理这两块网卡，要先把 DHCP 关掉，否则它会和脚本抢配置。
- 重启节点后执行 `cat /proc/net/bonding/bond0` 和 `ip route`，确认 bond 已建立、默认路由指向 10.30.30.1。

## 验证与故障演练

按自下而上的顺序逐层验证，每一层通过后再进行下一层。

**建成验证**

- [x] Underlay 邻居：`sudo vtysh -c "show bgp summary"`，确认 swp1、swp2（spine 上还有 peerlink.4094）都是 Established，并且邻居显示为接口名
- [ ] RFC 5549 下一跳：leaf1 上执行 `ip route show 10.255.0.12`，预期下一跳为 `169.254.0.1 dev swpX onlink`，并且是两条 ECMP 路由
- [x] MLAG：在 spine 上执行 `nv show mlag` 和 `clagctl`，确认 peer 为 alive，bond\_fg5、bond\_fg6 两侧都是 dual
- [x] FG LACP：执行 `diagnose netlink aggregate name agg1`，确认 port2、port3 都处于 selected 状态
- [x] HA：执行 `get system ha status`，确认 FG5 为 Primary、FG6 为 Secondary，配置 checksum 一致
- [x] FW BGP：FG 上执行 `get router info bgp summary`，spine 上执行 `sudo vtysh -c "show bgp vrf PROD summary"`，两条邻居都应 Established；FG 只学到 10.10.10.0/24，没有 /32 主机路由
- [x] EVPN：执行 `show evpn vni` 和 `show bgp l2vpn evpn route type prefix`，leaf 上应能看到 10.30.30.0/24，下一跳为 10.255.0.100
- [ ] 没有旁路：确认四台交换机都没有 vlan30 的 SVI（`ip -br addr | grep vlan30` 无输出）
- [ ] 业务连通：VPC 执行 `ping 10.30.30.10` 成功；FG 上 `diagnose sys session filter dst 10.30.30.10` 后执行 `diagnose sys session list`，能看到会话
- [ ] 路径：VPC 执行 `trace 10.30.30.10`，路径中应出现 172.16.100.x

**故障演练**（演练期间 VPC 保持持续 ping 容器）

- [ ] 关闭 spine1：FG 不发生主备切换，agg1 只剩一条成员，丢包应在数秒以内
- [ ] 主备切换：FG5 上执行 `execute ha failover set 1`，观察丢包量和 BGP 邻居是否经 GR 保持；完成后执行 `execute ha failover unset 1`
- [ ] 会话保持：先从 VPC 打开一条到容器的 TCP 长连接（例如容器里起 `nc -l 8080`），再切换 HA，检查连接是否存活；然后关闭 session-pickup 重做一遍，对比结果
- [ ] 容器双归：在 leaf1 上执行 `nv set interface swp3 link state down`，确认 bond 切换到 eth1；在 leaf 上用 `show evpn mac vni 10030` 观察 MAC 迁移和序列号增加
- [ ] Leaf 上联故障：关闭 leaf1 的 swp1，确认流量全部经 spine2 转发，且 BGP unnumbered 恢复后无需任何配置
- [ ] 策略软复位：在 FG 上给 spine 邻居加一条入向 route-map（例如修改 local-pref），观察路由不变；执行 `execute router clear bgp ip 172.16.100.2 soft in` 后，确认新策略生效

## 实验注意事项

最可能卡住实验的是 FortiGate-VM 的许可，而不是网络配置。

- **永久试用许可的限制**：只支持低加密；最多 1 个 CPU、2 GB 内存；接口、防火墙策略、路由各最多 3 个。本实验正好用了 port1 到 port3 三个口和 2 条策略。官方没有说明 agg 和 VLAN 子接口是否计入接口数，如果创建 agg1 或子接口被拒绝，只能退而用 port2 单口做 trunk，V 型冗余的演练就无法进行。
- **两台 FG 需要两份许可**：社区反馈每个 FortiCare 账号只能有一台永久试用 VM，所以 FG5 和 FG6 需要用两个账号分别激活，或者向厂商申请完整的评估许可。两台的许可类型要一致，否则 HA 可能无法组建。
- **路由数量**：spine 发往 FW 的 route-map 已经过滤掉主机路由，FG 只会学到 1 条路由，可以避免触及路由数上限。
- **建网顺序**：underlay BGP → MLAG → EVPN → FG 单机 LACP → 组建 HA → FW BGP → 策略。先让 FG5 单机跑通，再加入 FG6，排错范围更小。
- **NVUE 保存**：`nv config apply` 使配置生效后，再执行一次 `nv config save` 写入启动配置，避免重启后丢失；需要回滚时用 `nv config history` 查看历史版本。

**Sources**

- [Permanent trial mode for FortiGate-VM（FortiOS 7.4）](https://docs.fortinet.com/document/fortigate/7.4.6/administration-guide/441460)
- [Community：Failed to download license（每账号一台试用 VM）](https://community.fortinet.com/t5/Support-Forum/Failed-to-download-license/m-p/246410)
