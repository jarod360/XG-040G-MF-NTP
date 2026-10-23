# XG-040G-MF / XG-040G-MD 桥接模式 NTP 自动校时

本教程从**光猫已获得超级管理员密码与 Telnet root 权限**开始，完整覆盖 OpenWrt/QWRT 路由器侧管理网段、NTP 服务器、防火墙（fw3/fw4）、光猫配置、持久化脚本、验证和排障。

> **2026-10-22，XG-040G-MD：** 原有实测记录已完成删除排障 TR069 WAN 后的重启校时验证。
>
> **2026-10-23，XG-040G-MF（原厂 Linux 5.4.55）：** 已确认 `cfgcli`、`/sbin/ntp.sh`、可写且持久化的 `/etc` 与 `/etc/rc.d/S*` 启动入口可用；收到 `192.168.1.2:123` 的 48 字节 NTP 应答，状态变为 `Synchronized`。修正脚本后，客户端跨多个检查周期保持同一 PID，主动停止后可自动恢复。**本次 MF 尚未重启验证自启，也未等待完整 24 小时周期**；不要把 MD 的重启结果当作 MF 的结果。
>
> 两者使用 `br0`、`-T 0`、`-i 86400`、`-h 192.168.1.2,`。其他固件版本仍需逐项检查。

**路由器管理链路和 NTP 已正常时，直接从第 2 节开始，只配置光猫。** 不必重复修改路由器、重建接口或重启防火墙。本方案使用数字 IP 校时，不依赖光猫解析公网 NTP 域名，也不会恢复光猫的通用公网 DNS 或默认路由。

## 0. 拓扑与已知参数

```text
运营商光纤
   │
XG-040G（宽带桥接；LAN 管理 IP 192.168.1.1/24；br0）
   │   同一物理上联 / 正确的管理 VLAN
OpenWrt / QWRT 路由器
   ├─ WAN: 原有 PPPoE/拨号业务（不改）
   ├─ 光猫管理网：192.168.1.2/24（示例实测为 QWRT eth1.1 侧）
   │     └─ NTP 服务 UDP/123
   └─ 路由器 LAN：192.168.0.0/24（实例中独立于光猫管理网）
```

光猫的 `192.168.1.1` 与路由器管理侧 `192.168.1.2` 需要**二层可达**。不要把 `192.168.1.2` 随便绑定在不连接光猫的路由器接口，也不要把 WAN PPPoE 配置改为静态 IP。路由器 LAN 若本来就是 `192.168.1.0/24`，先调整管理规划，避免网段冲突。

### 实测和通用示例的边界

实测证实光猫 `br0` 可以 ping 通路由器 `192.168.1.2`，光猫 `ntpclient` 收到 NTP 响应并改变为 `Synchronized`；已确认 QWRT 管理侧为 `eth1.1`、路由器 LAN 使用 `192.168.0.x`。**路由器侧实测补充（用户确认）：** QWRT 的 `192.168.1.2/24` 并非来自 UCI，而是用户自行添加的自定义规则，通过 `ip addr` 直接配置到 `eth1.1`；LAN 到光猫的管理访问则通过 `nft` forward 和 srcnat 规则实现。NTP 服务 `system.ntp.enable_server='1'` 已存在。以下 UCI 配置是给其他 OpenWrt 设备首次部署的**可选替代方案**，不要与既有自定义规则同时执行。

## 1. 路由器：让默认无法访问光猫的桥接 WAN 可管理

### 1.1 先确定接口及防火墙

登录路由器 SSH：

```sh
cat /etc/openwrt_release
uci -q get network.wan.device
uci -q get network.wan.ifname
uci show network | grep -E 'wan|device|ifname|proto|ipaddr'
ip -br addr
ip route
uci show firewall | grep -E 'zone|network|name|input'
command -v fw4
command -v fw3
```

**实测 QWRT 物理上联管理设备为 `eth1.1`，不能推广到所有 OpenWrt。** 若是 DSA 固件，设备可能是 `wan`；若为交换芯片 VLAN，可能是 `eth0.2`。必须使用真实连接光猫管理网的设备。默认拨号 PPPoE 通常对应 `pppoe-wan`，它不是用于加管理 IP 的底层设备。

### 1.1.1 QWRT 已实测的自定义规则（与首次部署 UCI 方案二选一）

用户确认，当前 QWRT 管理地址由以下**自定义规则**设置，不在 `/etc/config/network`。这是对既有环境的准确记录，不要求重新执行：

```sh
MODEMIP="192.168.1.1"
ROUTER_WAN_PORT_IP="192.168.1.2"
WAN_PORT="eth1.1"
# 清理其他 VLAN 子接口上可能重复的管理地址
for iface in $(ls /sys/class/net/ | grep -E '^eth1\.[0-9]+$'); do
    if [ "$iface" != "$WAN_PORT" ]; then
        ip addr del "$ROUTER_WAN_PORT_IP"/24 dev "$iface" 2>/dev/null
    fi
done
ip addr del "$ROUTER_WAN_PORT_IP"/24 dev "$WAN_PORT" 2>/dev/null
ip addr add "$ROUTER_WAN_PORT_IP"/24 dev "$WAN_PORT"
ip link set "$WAN_PORT" up
ip route flush cache
# LAN 客户端访问光猫 Web/管理服务（非 NTP 服务必需）
nft insert rule inet fw4 forward \
    iifname "br-lan" oifname "$WAN_PORT" ip daddr "$MODEMIP" counter accept
nft insert rule inet fw4 srcnat \
    ip daddr "$MODEMIP" oifname "$WAN_PORT" counter masquerade
```

**这是既有规则的归档，不是安装必做步骤。** 重复删除、添加地址会造成瞬时断连，重复执行 `nft insert rule` 会累积规则；不要在管理链路已正常时重跑。它依赖现有 QWRT 的 `inet fw4` 表和对应链，也不能直接推广到其他路由器。

NTP 只需要光猫到路由器本机的 UDP/123 可达。上面的 forward/srcnat 用于 LAN 客户端访问光猫，不是 NTP 必需配置。

### 1.2 首次部署：增加独立管理 IP（与既有自定义规则二选一）

以下以实际 QWRT 案例的 `eth1.1` 为例；如果不同，请只替换 `ONT_DEV`。如果已有 `192.168.1.2/24`（例如由上面的 QWRT 自定义规则添加），**跳过此步，不能重复创建接口**。

```sh
ONT_DEV='eth1.1'   # 务必先确认：这是真正能到达光猫 br0 的设备
ip -br addr show dev "$ONT_DEV"
uci set network.ont_mgmt='interface'
uci set network.ont_mgmt.device="$ONT_DEV"
uci set network.ont_mgmt.proto='static'
uci set network.ont_mgmt.ipaddr='192.168.1.2'
uci set network.ont_mgmt.netmask='255.255.255.0'
uci set network.ont_mgmt.defaultroute='0'
uci set network.ont_mgmt.peerdns='0'
uci commit network
/etc/init.d/network reload
```

某些旧式配置使用 `ifname` 而非 `device`。如果系统属于该类型，**不要同时重复设置**，改成：

```sh
uci -q delete network.ont_mgmt.device
uci set network.ont_mgmt.ifname="$ONT_DEV"
uci commit network
/etc/init.d/network reload
```

确认：

```sh
ip -4 addr show dev "$ONT_DEV"
ip route get 192.168.1.1 from 192.168.1.2
ping -c 3 -I 192.168.1.2 192.168.1.1
```

若 ping 不通，先排查物理口、VLAN、光猫 LAN 管理访问、接口是否真正可达；**不要用创建 TR069 WAN 的方法解决路由器到光猫的二层问题**。

### 1.3 fw3 / fw4 防火墙：放行光猫到路由器的 NTP（只开 UDP/123）

两者的**标准 UCI 规则写法相同**，区别是底层实现和查看生效规则的命令：

- **fw3**（iptables）通常运行在较老的 OpenWrt；检查 `fw3 print`、`iptables -L INPUT -n -v`。
- **fw4**（nftables）通常运行在较新的 OpenWrt；检查 `fw4 print`、`nft list ruleset`。

先检查 `ont_mgmt` 被放在哪个 firewall zone。通常需要把它加入现有 `wan` zone，因为它接在上联端口；若你有专用 `ont` zone，请使用那个 zone。**下面假设专用管理接口在 `wan` zone**。

```sh
uci show firewall | grep -E '=zone|\.name=|\.network='
```

如现有 WAN zone 的 section 是 `firewall.@zone[1]`，不要凭索引假定；先找到 `option name 'wan'` 对应的实际 section。以下通过脚本查找：

```sh
WAN_ZONE=''
for zone in $(uci show firewall | sed -n 's/^\(firewall\.[^.=]*\)=zone$/\1/p'); do
    [ "$(uci -q get "$zone.name")" = 'wan' ] && WAN_ZONE="$zone"
done
echo "WAN_ZONE=$WAN_ZONE"
[ -n "$WAN_ZONE" ] || { echo '未找到 wan zone，停止并手动检查'; exit 1; }
uci add_list "$WAN_ZONE.network=ont_mgmt"
uci commit firewall
```

如果 `ont_mgmt` 已经存在于某个 zone，**不要重复加入另一个 zone**；已有 zone 是 LAN（input ACCEPT）时可能不需要额外规则，但仍应限制来源。不要修改 `wan` zone 的全局 `input REJECT` 策略。

在管理接口属于 `wan` zone 时，fw3/fw4 通用规则：

```sh
uci -q delete firewall.ont_ntp
uci set firewall.ont_ntp='rule'
uci set firewall.ont_ntp.name='Allow-ONT-NTP'
uci set firewall.ont_ntp.src='wan'
uci set firewall.ont_ntp.src_ip='192.168.1.1'
uci set firewall.ont_ntp.proto='udp'
uci set firewall.ont_ntp.dest_port='123'
uci set firewall.ont_ntp.target='ACCEPT'
uci set firewall.ont_ntp.family='ipv4'
uci commit firewall
/etc/init.d/firewall restart
```

这条是**到路由器本机**的 INPUT 规则，所以**不要**设置 `dest='lan'` 或 DNAT，也无需对光猫开放整个 WAN。若想仅限管理接口，可加：

```sh
uci set firewall.ont_ntp.device='eth1.1'
uci commit firewall
/etc/init.d/firewall restart
```

但不同 fw3/fw4 版本对 `device` 的处理并不一致，只有在输出中确认生效时才启用此可选项。若 UCI 防火墙中的 `src='wan'` 不包含 `ont_mgmt`，规则就不会匹配。

**fw3 验证**：

```sh
fw3 print 2>/dev/null | grep -A 5 -B 5 'Allow-ONT-NTP\|192.168.1.1'
iptables -L INPUT -n -v 2>/dev/null
```

**fw4 验证**：

```sh
fw4 print 2>/dev/null | grep -A 5 -B 5 'Allow-ONT-NTP\|192.168.1.1'
nft list ruleset 2>/dev/null | grep -A 5 -B 5 'Allow-ONT-NTP\|192.168.1.1'
```

若接口在独立 `ont` zone，把规则的 `src` 改成 `ont`；如果管理口已归入 WAN zone，无需另建 zone。

### 1.4 启用 NTP 服务（UDP/123）

先查看所用 NTP 实现：

```sh
uci show system | grep -i ntp
ps | grep -E '[s]ysntpd|[n]tpd|[c]hronyd'
netstat -lun 2>/dev/null | grep ':123 '
```

OpenWrt 常见的 `sysntpd` / busybox ntpd 通过 `system.ntp.enable_server` 开启服务。先备份并执行：

```sh
cp /etc/config/system /etc/config/system.ntp.bak
uci set system.ntp.enable='1'
uci set system.ntp.enable_server='1'
uci commit system
/etc/init.d/sysntpd restart
```

如果 `system.ntp` 不是现有的命名 section（`uci get system.ntp` 报错），先通过 `uci show system` 查找真实 section，不要盲目覆盖现有设置。

再次检查监听：

```sh
ps | grep '[n]tpd'
netstat -lun 2>/dev/null | grep ':123 '
```

`netstat` 可能只显示 `0.0.0.0:123`；这仍可能覆盖 `192.168.1.2`，以光猫实际请求响应为最终判断。部分第三方 NTP 服务需要单独配置监听地址。

**测试路径必须是：光猫 `192.168.1.1` → 路由器 `192.168.1.2:123/udp`。** 路由器能 ping 光猫并不能证明 UDP/123 已放行。

## 2. 光猫：检查参数、验证 NTP、配置 24 小时校时

Telnet root 登录光猫。先核实本机：

```sh
mount | grep -E ' on / | on /etc '
ls -l /sbin/ntp.sh /sbin/ntpclient /usr/etc/init.d/rcS
ip -4 addr show br0
cfgcli -g InternetGatewayDevice.Time.Enable
cfgcli -g InternetGatewayDevice.Time.NTPServer1
cfgcli -g InternetGatewayDevice.Time.LocalTimeZone
cfgcli -g InternetGatewayDevice.Time.X_CT-COM_NTPInterval
cfgcli -g InternetGatewayDevice.Time.Status
```

本次实际配置：`NTPServer1=192.168.1.2`、时区 `+08:00`、间隔 `86400`、NTP 启用。可在光猫 Web 时间设置页填写 NTP 服务器为 `192.168.1.2`；若本机 `cfgcli` 支持下面字段，命令形式为：

```sh
cfgcli -s InternetGatewayDevice.Time.Enable TRUE
cfgcli -s InternetGatewayDevice.Time.NTPServer1 192.168.1.2
cfgcli -s InternetGatewayDevice.Time.LocalTimeZone '+08:00'
cfgcli -s InternetGatewayDevice.Time.X_CT-COM_NTPInterval 86400
```

上面 `cfgcli` 写操作依赖固件允许对应字段修改；设置完用 `cfgcli -g` 复核。其他 NTP 服务器可能显示厂商占位文本“**不配置**”，不建议未经核实批量覆盖字段。

**先手动校时验证 NTP 服务端可用**（临时测试，若系统已有 `ntpclient`，先确认后停止，避免两个客户端；验证后让看门狗接管）：

```sh
ping -c 2 -I br0 192.168.1.2
/sbin/ntp.sh '192.168.1.2,' '+08:00' 'br0' '0' '86400'
sleep 8
ps w | grep '[n]tpclient'
cfgcli -g InternetGatewayDevice.Time.Status
```

在本机 `-e br0 -T 0` 已收到 NTP 报文并校时。曾试过的 `-e pon_u_0_1` 无法与路由器二层通信，不应使用该接口。**不需要创建 TR069 WAN、不需要配置 ACS**。

## 3. 光猫：安装开机自动校时看门狗

本机根目录是只读 squashfs，但 `/etc` 持久可写；`rcS` 会执行 `/etc/rc.d/S*` 可执行脚本。**先确认固件确实支持该机制**：

```sh
grep -n '/etc/rc.d' /usr/etc/init.d/rcS
ls -ld /etc/rc.d
```

MF 实测安装前 `/etc/rc.d` 可以不存在，但 `rcS` 已有扫描该目录的逻辑；确认入口存在后用 `mkdir -p /etc/rc.d` 创建即可。不要把目录尚未创建误判为固件不支持自启。

仓库附有 [S99ntpfix](S99ntpfix)。首次安装先创建目录，再把脚本传到 `/etc/rc.d/S99ntpfix`（例如通过 Telnet 粘贴内容，或在能使用 scp 的环境复制）。**更新已有安装时先停止旧看门狗，并把旧文件备份到 `/etc/ntpfix-backups/` 等目录，不要把可执行备份留在 `/etc/rc.d/S*`，否则开机可能同时运行两份。** 传输完成后设置权限并检查语法：

```sh
mkdir -p /etc/rc.d
chmod 755 /etc/rc.d/S99ntpfix
sh -n /etc/rc.d/S99ntpfix && /etc/rc.d/S99ntpfix start
sync
```

默认参数在脚本顶部：

```sh
SERVER_IP='192.168.1.2'
INTERFACE='br0'
INTERVAL=86400
TZ_OFFSET='+08:00'
CHECK_SECONDS=10
BOOT_DELAY=20
```

- **`INTERVAL=86400` 是 ntpclient 校时间隔**；10 秒只是看门狗检查进程和网络就绪情况，**不是每 10 秒校时**。
- 看门狗仅在 NTP 未运行、实例多于一个或启动参数错误时才重启。
- 不直接替换厂商只读 `/sbin/ntp.sh`、`ntpclient` 或 `cfgmgr`。
- 光猫厂商 `tr` 并非标准 `tr`，使用 `ps w` 检查完整进程参数。普通 `ps` 在 MF 上会把服务器地址截断为 `192.`，不能用它判断参数是否正确。
- MF 实测 `ping -c 1 -W 2 -I br0 192.168.1.2` 返回 `0`，支持 `-W`，即使 `ping --help` 没列出该选项。以实际执行结果为准。
- `/tmp/ntpfix.log` 是临时日志；首次同步前系统可能显示 `1970-01-01 08:00:xx`，不代表脚本失败。

**停止/查询**：

```sh
/etc/rc.d/S99ntpfix status
/etc/rc.d/S99ntpfix stop
/etc/rc.d/S99ntpfix start
```

`stop` 只停止看门狗，保留已运行的 NTP。此前版本曾使用 `LOCKDIR`，最终脚本改为 PID 文件；安装前如有旧进程，请先检查 `ps w | grep '[S]99ntpfix'` 并停掉旧监控，避免两个循环同时管理。

### 3.1 已修正的两处误重启问题

1. **进程参数截断：** 原脚本使用普通 `ps`，MF 默认输出宽度导致 `-h 192.168.1.2,` 显示不全。改为 `ps w`，进程检查和 status 输出均使用完整命令行。
2. **匹配模式重复空格：** 原 `case` 把 `" -e br0 "` 和 `" -T 0 "` 连在一起，前一段尾部与后一段开头各占一个空格，无法匹配实际仅有一个分隔空格的相邻参数。修正模式让下一段从 `-T`、`-i`、`-h` 开始，避免重复消耗空格。

两处问题都会把健康客户端误判为参数错误，造成大约每 14 秒出现一次 `Restarting NTP on br0`。实测旧模式返回不匹配，修正模式匹配同一条正确命令；修复后进程稳定，恢复测试才新增一次重启记录。**10 秒是健康检查间隔，不能因为旧脚本频繁重启就认为 86400 秒的校时间隔没有生效。**

## 4. 完整验证

### 4.1 查看当前运行状态

```sh
cat /tmp/ntpfix.pid
ps w | grep '[S]99ntpfix'
ps w | grep '[n]tpclient'
cfgcli -g InternetGatewayDevice.Time.Status
tail -n 20 /tmp/ntpfix.log
```

正确参数应类似：

```text
/sbin/ntpclient -d -e br0 -T 0 -i 86400 -tsl -h 192.168.1.2,
InternetGatewayDevice.Time.Status=Synchronized
```

### 4.2 独立恢复测试

请在可接受时间短暂中断时操作；**这会重新触发一次 NTP 校时，但不改变后续 86400 秒间隔**。

```sh
killall ntpclient
sleep 45
ps w | grep '[n]tpclient'
tail -n 10 /tmp/ntpfix.log
cfgcli -g InternetGatewayDevice.Time.Status
```

预期：出现新 PID，日志出现 `Restarting NTP on br0`，配置仍为 `-i 86400`。单看 `Synchronized` 可能是上次状态，最好结合 NTP 调试输出、路由器 UDP/123 计数或抓包验证新响应。

### 4.3 重启验证

在可维护窗口重启光猫，约 2 分钟后检查：

```sh
cat /tmp/ntpfix.pid
ps w | grep '[S]99ntpfix'
ps w | grep '[n]tpclient'
cfgcli -g InternetGatewayDevice.Time.Status
cat /tmp/ntpfix.log
ping -c 2 -I br0 192.168.1.2
```

**2026-10-22 MD 历史实测：** 开机日志 `1970-01-01 08:00:17 Monitor started`，`1970-01-01 08:01:40 Restarting NTP on br0`；最终自动校时成功，临时自建 TR069 WAN 已被删除。

**2026-10-23 MF：** 已安装持久化脚本并验证启动入口、运行稳定性与进程退出后的自动恢复，但没有重启光猫验证；上面的重启步骤仍待在维护窗口执行。

## 5. 排障速查

| 表现 | 原因与检查 |
| --- | --- |
| 路由器无法访问桥接光猫 | `ont_mgmt` 绑错设备、上联 VLAN 不通、与 LAN 子网冲突；查 `ip route get 192.168.1.1` 和 `ip -4 addr` |
| 光猫 ping 不通 `192.168.1.2` | 检查光猫 `br0`、路由器设备/VLAN、路由器 firewall 对 ICMP 的策略 |
| Ping 通但 NTP 不同步 | 检查路由器 `sysntpd`、UDP/123 INPUT 规则、来源 192.168.1.1、实际监听 |
| `ntpclient` 绑定 `pon_u_0_1` | 原厂选错 routed WAN；看门狗统一使用 `br0`；不需要新建 TR069 WAN |
| 光猫重启后暂时没有 NTP | 等 LAN 初始化和第一次看门狗检查；查 `/tmp/ntpfix.log` |
| `Synchronized` 但无 NTP 进程 | 状态可能未刷新，查看进程和新 NTP 应答 |
| 日志显示 1970 年 | 校时前系统初始时钟；同步成功后恢复正确日期 |
| 看门狗约每 14 秒重启客户端 | 先升级修正后的脚本，检查 `ps w` 与第 3.1 节的空格匹配问题；再检查上游网络、参数和重复看门狗 |
| `tr` 输出异常 | 某些厂商固件重定义 `tr`；本脚本不依赖它 |

## 6. 回滚

1. `/etc/rc.d/S99ntpfix stop`，删除 `/etc/rc.d/S99ntpfix` 即可取消本方案的开机看门狗。
2. 如在路由器新增了 `ont_mgmt` 与 `firewall.ont_ntp`，可针对**确实由本教程新增**的配置删除（注意此举会断开光猫管理网络）：
   ```sh
   uci -q delete firewall.ont_ntp
   uci commit firewall
   /etc/init.d/firewall restart
   uci -q delete network.ont_mgmt
   uci commit network
   /etc/init.d/network reload
   ```
3. 如果为 `wan` zone 添加过 `ont_mgmt` network 成员，还需仅移除该成员：先从 `uci show firewall` 找到正确 zone，使用 `uci del_list "$WAN_ZONE.network=ont_mgmt"`，确认后提交防火墙配置。
4. 若路由器原本就通过 `192.168.1.2` 服务其他设备，不要因本教程随意关闭 NTP 服务或删除原本存在的地址。

## 安全与适用范围

本教程仅针对自己管理的光猫和路由器。公网侧不应暴露 Telnet/root 管理入口。更改防火墙或网络配置前先备份 `/etc/config/network`、`/etc/config/firewall`、`/etc/config/system`，避免断开现有上网业务。XG-040G-MF 和 MD 固件差异请逐项核实。