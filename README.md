# 搬瓦工转发教程完全指南：iptables / socat / Nginx 怎么配？流量中转原理、步骤与套餐选择一篇搞定

朋友上个月问我，他有一台搬瓦工洛杉矶的 VPS，访问国内业务延迟高得离谱，有没有办法把流量通过国内的一台服务器中转一下。我跟他说，这不就是搬瓦工转发嘛，配起来也就二三十分钟的事。

结果他发过来十几条消息，说网上的教程要么太老要么太碎，看完还是不知道从哪下手。

所以干脆写一篇，从原理到操作，把搬瓦工转发这件事完整说一遍。

---

## 搬瓦工转发是什么，为什么要这么做

搬瓦工转发，说白了就是流量中转——原来是"你的电脑 ↔ 搬瓦工海外 VPS"这条链路，改成"你的电脑 ↔ 国内中转服务器 ↔ 搬瓦工海外 VPS"。

看起来多了一跳，实际上快了。

原因在于：国内到海外的直连链路，高峰期丢包率高、延迟抖动大；但国内服务器之间的网络是内网直连，延迟低，再从国内服务器出境走 CN2 GIA 这种优质线路，总延迟反而比直连小，带宽也更稳。

搬瓦工本身提供 CN2 GIA、香港、日本软银等高质量线路，已经是国内直连里比较顶级的方案了。加上国内中转之后，效果更明显，主要体现在：访问延迟降低、高峰期抖动减少、线路被封时有缓冲余地。

---

## 转发前的准备：你需要两台机器

搬瓦工转发的最小配置是两台服务器：

- **落地机**（搬瓦工 VPS）：跑代理服务或目标应用，位于海外
- **中转机**：国内服务器，负责把流量转过去

两台机器都要能 SSH 登录，系统建议用 Debian 或 Ubuntu，比 CentOS 少踩坑。

中转机的选择不用太纠结，国内任何云服务商的轻量应用服务器都行，主要看跟搬瓦工之间的延迟。不同运营商线路差异较大，建议选购前先 ping 一下搬瓦工目标机房的测试 IP，哪家延迟低就用哪家。

搬瓦工这边，转发对套餐配置没有特别高的要求，入门的 KVM 方案（1GB 内存、20GB SSD、1TB 月流量）对单人或小团队来说够用了，在意速度的话上 CN2 GIA-E 方案。

👉 [查看搬瓦工当前所有套餐与价格](https://bit.ly/BanWaGon)

---

## 搬瓦工转发教程：三种主流方法

### 方法一：iptables 端口转发（推荐，最稳定）

iptables 是 Linux 内核自带的防火墙工具，直接在内核层做转发，CPU 占用几乎可以忽略不计，适合长期稳定运行。

**在中转机上操作：**

1. 关闭 firewalld（如果是 CentOS 系统）

bash
systemctl stop firewalld
systemctl disable firewalld


2. 安装 iptables

bash
# Debian / Ubuntu
apt install iptables iptables-persistent -y

# CentOS
yum install iptables-services -y
systemctl enable iptables
systemctl start iptables


3. 添加转发规则

把下面命令里的参数换成你自己的：`本机端口` 是中转机监听的端口，`落地机IP` 是搬瓦工 VPS 的 IP，`落地机端口` 是搬瓦工那边跑的服务端口，`本机内网IP` 是中转机的内网地址（用 `ip addr` 查）。

bash
# TCP 转发
iptables -t nat -A PREROUTING -p tcp --dport 本机端口 -j DNAT --to-destination 落地机IP:落地机端口
iptables -t nat -A POSTROUTING -p tcp -d 落地机IP --dport 落地机端口 -j SNAT --to-source 本机内网IP

# UDP 转发（需要的话同步加上）
iptables -t nat -A PREROUTING -p udp --dport 本机端口 -j DNAT --to-destination 落地机IP:落地机端口
iptables -t nat -A POSTROUTING -p udp -d 落地机IP --dport 落地机端口 -j SNAT --to-source 本机内网IP


4. 允许转发通过（防止 FORWARD 链丢包）

bash
iptables -I FORWARD -d 落地机IP -j ACCEPT
iptables -I FORWARD -s 落地机IP -j ACCEPT


5. 保存规则，重启后不丢失

bash
# Debian / Ubuntu
netfilter-persistent save

# CentOS
service iptables save


做完这几步，客户端把服务器地址换成中转机的 IP、端口填中转机监听的端口，就能用了。落地机搬瓦工那边的配置不用动。

---

### 方法二：socat（配置最简单，适合临时或少量端口）

socat 比 iptables 安装更简单，两行命令能起来，但在大流量场景下性能不如 iptables，适合测试或者短期需要。

1. 安装 socat

bash
apt install socat -y    # Debian / Ubuntu
yum install socat -y    # CentOS


2. 启动转发

bash
nohup socat TCP4-LISTEN:本机端口,reuseaddr,fork TCP4:落地机IP:落地机端口 &


加 `nohup` 和 `&` 是为了让进程在后台跑、不因为 SSH 断开而停止。

3. 要让 socat 开机自启，需要写一个 systemd 服务文件，网上有模板，这里不展开了。如果只是测试，手动跑就够。

socat 支持转发带域名的地址（iptables 不行），这是它的一个独特优势。如果落地机用的是域名而不是固定 IP，socat 是更合适的选择。

---

### 方法三：Nginx stream 模块（适合已有 Nginx 的环境）

如果中转机上已经装了 Nginx，可以直接用 stream 模块做 TCP/UDP 转发，不用额外装东西。

1. 确认 Nginx 版本带 stream 模块

bash
nginx -V 2>&1 | grep stream


2. 在 `/etc/nginx/nginx.conf` 里加入 stream 块（在 http 块之外）

nginx
stream {
    upstream backend {
        server 落地机IP:落地机端口;
    }
    server {
        listen 本机端口;
        proxy_pass backend;
    }
}


3. 测试配置并重载

bash
nginx -t && nginx -s reload


Nginx stream 方式的好处是可以顺手做多后端负载均衡，有多台搬瓦工 VPS 的时候比较好用。

---

## 三种方法横向对比

| 方式 | 性能 | 配置难度 | 适用场景 | TCP | UDP |
|---|---|---|---|---|---|
| iptables | ⭐⭐⭐⭐⭐ | 中等 | 长期稳定运行 | ✅ | ✅ |
| socat | ⭐⭐⭐ | 简单 | 临时测试、域名转发 | ✅ | ✅ |
| Nginx stream | ⭐⭐⭐⭐ | 中等 | 已有 Nginx 环境 | ✅ | ✅（需额外编译）|

实际用下来，日常场景选 iptables 就好，稳、快、省资源。socat 留着测试用。Nginx 留给有多个后端要做负载均衡的情况。

---

## 搬瓦工套餐选哪个做落地机

配合转发来用，搬瓦工 VPS 的选择逻辑其实很清晰——中转机已经帮你处理了国内那段延迟，搬瓦工这边主要看**出境线路质量**和**稳定性**。

搬瓦工目前在售套餐主要分这几档：

| 套餐系列 | 代表配置 | 月流量 | 带宽 | 价格 | 可选机房 | 适合场景 |
|---|---|---|---|---|---|---|
| 基础 KVM（入门） | 1GB 内存 / 20GB SSD / 2核 | 1TB | 1Gbps | $49.99/年 | 欧美多机房 | 学习、建站、轻量转发 |
| KVM 大配置 | 2GB 内存 / 40GB SSD / 3核 | 2TB | 1Gbps | $52.99/半年 | 欧美多机房 | 多人共用中转 |
| CN2 GIA-E（主力） | 1GB 内存 / 20GB SSD | 1TB | 1Gbps | $49.99/季 | 12个机房可切换 | 对速度有要求的转发 |
| CN2 GIA-E（大流量） | 2GB 内存 / 40GB SSD | 2TB | 1Gbps | $89.99/季 | 12个机房可切换 | 流量需求大的场景 |
| 香港 CN2 GIA | 2GB 内存 / 40GB SSD / 2核 | 500GB | 1Gbps | $89.99/月 | 香港机房 | 极低延迟、高端需求 |
| 大阪 CN2 GIA | 2GB 内存 / 40GB SSD / 2核 | 500GB | 1.5Gbps | $49.99/月 | 大阪机房 | 日本线路、软银直连 |
| 洛杉矶 SLA 企业版 | 1GB ECC / 20GB NVMe / 2核AMD | 1TB | 2.5Gbps | $65.89/季 | 洛杉矶 DC5 | 企业级稳定性需求 |

👉 [对比所有套餐，选最适合你的方案](https://bit.ly/BanWaGon)

说个实际用法：如果你是一个人用，走 $49.99/年 的入门 KVM 套餐接 iptables 中转，完全够——1TB 流量、1Gbps 带宽，跑个转发基本用不完。

如果两三个人一起用，或者流量需求大，CN2 GIA-E 季付方案是性价比最高的选择。算下来一个月不到 $17，还能在 12 个机房之间随便切，哪个机房被封了直接迁，不用重新买。

香港套餐延迟确实低到飞起，广东方向单向延迟可以压到个位数毫秒，但 $89.99/月 的价格不是所有人都需要，预算充足再考虑。

---

## 转发配好了，怎么验证是否生效

配完之后别急着关 SSH 窗口，先做个简单验证：

1. 在本地尝试连接中转机的端口，看是否能通
2. 用 telnet 或 nc 测试端口连通性：`nc -vz 中转机IP 本机端口`
3. 检查搬瓦工那边能否收到请求（看日志）

另外，搬瓦工的 KiwiVM 控制面板里有流量统计，转发生效后，搬瓦工 VPS 的入站流量应该会有变化，可以拿这个做参考。

---

## 常见问题

**Q：转发会影响搬瓦工的月流量计算吗？**

会。流量是从搬瓦工出口出去的，所以中转过来的流量会算进搬瓦工的月流量。选套餐时流量预算要留足，不够的话考虑上 2TB 以上的方案。

**Q：搬瓦工的流量用完了会怎样？**

不会产生额外扣费。月流量耗尽后，VPS 自动暂停到下个月重置，不会因为超量产生额外账单。下个月 1 号流量自动恢复。

**Q：用 iptables 转发，中转机重启后规则会不会丢失？**

会，所以必须保存规则。Debian/Ubuntu 用 `netfilter-persistent save`，CentOS 用 `service iptables save`，这样规则写入磁盘，重启后自动恢复。

**Q：搬瓦工 VPS 的 IP 被封了，转发还能用吗？**

国内访问被封但海外没问题的情况下，如果中转机 IP 没被封，还可以用——因为国内的连接只到中转机，搬瓦工 IP 封不封不影响这段。搬瓦工 CN2 GIA-E 套餐支持免费换机房，遇到这种情况迁过去就好了。

**Q：搬瓦工支持哪些支付方式？**

支持支付宝、PayPal、信用卡，国内用户用支付宝直接付就行，不需要找人代购。

---

## 写在最后

搬瓦工转发这个方案，门槛没有很多人想象的那么高。iptables 那几条命令，抄过去改一下 IP 和端口就能跑，不需要什么 Linux 高级技能。

真正需要花时间的是前期的线路测试——中转机选哪家、搬瓦工选哪个机房，测一下延迟和丢包，比跟着别人推荐无脑买靠谱得多。

搬瓦工这边，如果不知道选什么，先走 CN2 GIA-E 季付方案，12 个机房随便切，哪个快用哪个，迁移不要钱。

👉 [前往搬瓦工查看最新套餐与优惠](https://bit.ly/BanWaGon)
