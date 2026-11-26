# telegram机器人 vps 怎么选：1G 内存够不够、机房放哪、webhook 的 443 从哪来（附 SixtyNet 套餐价格与端口限制）

跑一个 Telegram bot，1 GB 内存的 KVM 就够用。2026 年 9 月俄语区几家服务商的入门档在 99 到 290 ₽/月之间，配置是约 1 vCPU、1 GB 内存、NVMe。SixtyNet 的 VPS 从 2 核 2 GB 起，年付 $200，这个配置对单个 bot 明显过剩。它值得掏钱的理由在别处：美国住宅 IP、中国方向优化的 AS9929 线路、5 TB 起步的流量。代价写在官网产品页上——住宅 IP 那几档不开 80 和 443 端口，webhook 部署方式直接出局。

下面按你下单前真正要做的几个判断来排：SixtyNet 现在有哪些档、多少钱；要不要自己管服务器；内存买多大；机房和端口怎么定；买完之后怎么让 bot 自己活过来。

> **大多数人的选择**：单个 bot、只处理命令和回调，LAX-STARTER 的年付 $200 已经超出需求三倍以上，但它是这家最便宜的入口。要用 webhook，就必须避开住宅 IP 档。只有当你还需要美国住宅 IP 做别的业务、bot 顺带挂在同一台机器上，IRVINE-PRO-STARTER 的年付 $350 才有意义。
> 👉 [查看 SixtyNet VPS 各档当前价格与库存](https://sixtynet.net/aff.php?aff=39)

## SixtyNet 的 VPS 全部档位与价格

官网 VPS 页面当前展示的五档有明确套餐名和年付价，整理如下。价格是页面上的年付促销价，原价同时列出。

| 套餐 | CPU / 内存 / 磁盘 | 流量与网络 | 年付价格 | 入口 |
| --- | --- | --- | --- | --- |
| LAX-STARTER | 2 核 / 2 GB / 40 GB NVMe | 5 TB，2G 口，含 DDoS 防护 | **$200**（原价 $240） | [查看 LAX-STARTER](https://sixtynet.net/aff.php?aff=39) |
| LAX-MICRO | 4 核 / 4 GB / 80 GB NVMe | 10 TB，4G 口，含 DDoS 防护 | **$400**（原价 $480） | [查看 LAX-MICRO](https://sixtynet.net/aff.php?aff=39) |
| LAX-MEDIUM | 8 核 / 8 GB / 160 GB NVMe | 20 TB，10G 口，含 DDoS 防护 | **$800**（原价 $960） | [查看 LAX-MEDIUM](https://sixtynet.net/aff.php?aff=39) |
| LAX-LARGE | 8 核 / 16 GB / 320 GB NVMe | 40 TB，10G 口，含 DDoS 防护 | **$1,600**（原价 $1,920） | [查看 LAX-LARGE](https://sixtynet.net/aff.php?aff=39) |
| IRVINE-PRO-STARTER | 2 核 / 2 GB / 40 GB NVMe | 5 TB，2G 口，AS9929 中国优化，住宅 IP | **$350**（原价 $420） | [查看 IRVINE-PRO-STARTER](https://sixtynet.net/aff.php?aff=39) |

五档的年付折扣都是 17%，等于把计费周期锁在一年。官网同一页面还列了住宅 IP 的 4 核、8 核、16 核档，以及 AS9929 系列里 4 核 / 4 GB / 80 GB 的那一档，抓取时这几档的价格没有展开在页面上。下单当天以官网显示为准，不要按同系列的配置比例去推算。

**逐档判断**

LAX 四档带的是 DDoS 防护，页面没有标注住宅 IP，也没有那条端口限制说明。对 Telegram bot 来说，LAX-STARTER 是唯一年付价低于 $250 的选项，2 核 2 GB 跑一个 aiogram 或 Telegraf 写的 bot，CPU 和内存都用不到一半。

LAX-MICRO 往上，多出来的核心和内存是给同机跑数据库、队列、后台任务或者多个 bot 用的。单个 bot 买 LAX-MEDIUM 的 8 核 8 GB，是纯粹为流量和带宽付费——除非你还要在同一台机器上跑别的服务，否则这 $800 花得没有落点。

IRVINE-PRO-STARTER 是这一页里唯一同时带 AS9929 和住宅 IP 的入门档，年付比 LAX-STARTER 贵 $150。它多出来的价值全在 IP 类型和回程线路上，不在 CPU 和内存。页面还额外写了两条限制：为安全起见拒绝所有外部探测和扫描，只有加入白名单或做了端口转发的设备能访问；退款只退到网站余额，不原路返回。

## 先想清楚：为什么家里的电脑跑不了

bot 的可用性等于进程的可用性。进程一停，用户发消息就是没有回应。家用电脑有三条路会让进程停掉：系统休眠、意外重启、宽带拿到的是运营商 NAT 后面的地址，外部根本连不进来。前两条靠改电源设置能缓解，第三条改不了。

部署方式有两种，对服务器的要求完全不同。

**long polling**：你的程序主动向 `api.telegram.org` 发起 HTTPS 请求，问有没有新消息，然后等在那里。整个链路只需要出站 443，不需要公网 IP，不需要开放任何入站端口，不需要证书。

**webhook**：Telegram 主动往你的域名 POST 消息。需要公网可达的 443 端口，需要一个有效证书，还需要一个能对外提供 HTTPS 的进程（nginx、Caddy 或者框架自带的服务器）。延迟更低，代价是运维面变大。

这一条区别决定了后面的所有选择。SixtyNet 的住宅 IP 产品不开 80 和 443，意味着那几档只能跑 long polling。

**买完先做的事**

文件传上去、依赖装好，先用 `nohup python3 bot.py &` 试跑一遍，确认 token 和网络都没问题。跑通之后立刻交给 systemd，否则 SSH 一断、服务器一重启，bot 就没了。

ini
[Unit]
Description=Telegram Bot
After=network.target

[Service]
ExecStart=/usr/bin/python3 /root/bot.py
Restart=always

[Install]
WantedBy=multi-user.target


存到 `/etc/systemd/system/bot.service`，然后：

bash
sudo systemctl enable bot
sudo systemctl start bot
sudo systemctl status bot


`Restart=always` 是这段配置里唯一不能省的一行。进程因为异常退出、内存耗尽、依赖更新炸掉，systemd 会把它拉起来。`enable` 保证开机自启。没有这两条，你等于花钱买了一台需要手动重启的服务器。

## 内存到底买多大

俄语区一份 2026 年 9 月的横评给出的分档口径比较严：

| 场景 | 内存 | 说明 |
| --- | --- | --- |
| 只处理命令、走 webhook，用 aiogram 或 Telegraf | 1 GB | 够用，且有余量 |
| bot 旁边还要跑 SQLite、PostgreSQL 或 Redis | 接近 2 GB | 数据库进程会吃掉内存 |
| 处理图片、文件、消息队列 | 2–4 GB，且要 NVMe | 磁盘 I/O 会成为瓶颈 |

同一份资料明确把 512 MB 及以下排除在参考范围之外。另一份 2026 年 6 月的榜单口径更宽松：简单的 Python 或 Node.js bot，1 vCPU 加 512 MB 到 1 GB 内存就够，前提是 443 端口开放、带宽不低于 100 Mbit/s、服务商不在高峰期压缩 CPU。两套说法并列看，取严的那套更安全。

SixtyNet 最低档是 2 GB，比两套口径的上限都高。单个 bot 用不满，但你也没有更小的可选。这也是为什么前面说，选这家不该是为了跑 bot 本身。

👉 [按年付价对比 SixtyNet 的 2 GB 与 4 GB 档](https://sixtynet.net/aff.php?aff=39)

## 443 端口与机房：两个必须先确认的点

**端口。** SixtyNet 在住宅 IP 产品下面写得很直接：出于 IP 风控原因，住宅 IP 产品不开放 80 和 443 端口。落到部署上就是一句话：这几档跑不了 webhook，只能 long polling。IRVINE-PRO-STARTER 那一档还叠加了拒绝外部探测和扫描的策略，只有白名单或端口转发的设备能连进来。出站连接不受这条限制影响，所以 long polling 照常工作。

要跑 webhook 就走 LAX 系列。官网在这四档下面没有写端口限制，但也没有明确写 443 开放。下单前开工单问一句，比事后发现要换机器省事。

**机房与连通性。** 有一篇 2026 年 4 月的文章主张，bot 要连上 Telegram API 必须放在俄罗斯境外的 VPS。这篇文章出自一家卖境外机房的厂商账号，属于利益相关方的说法。同期另一份横评把俄罗斯机房当成正常选项，也有服务商在自己的落地页上写明荷兰和马德里 IP 可以直连 Telegram API。实际情况是部分地区、部分 IP 段会遇到连通问题，不是普遍规律。

判断方法是在目标机器上直接测：

bash
curl -sS -m 10 -o /dev/null -w "%{http_code}\n" https://api.telegram.org


拿到 HTTP 状态码，说明出站通。超时或者连接被重置，换机房，别在配置上耗时间。

还有一个容易被忽略的变量是 IP 的"干净程度"。有服务商提到 Telegram API 可能封禁被滥用过的 IP 段，机房 IP 池被反复使用，撞上黑名单的概率不低。住宅 IP 天然不在这类 IP 段里，这是 SixtyNet 那几档的实际价值所在，代价就是端口限制。两件事是绑在一起的，不能只取其一。

## 同类需求下，别人的价格是多少

以下是 2026 年 9 月俄语区市场快照，口径统一为 KVM、约 1 vCPU、1 GB 内存、SSD 或 NVMe，不含 OpenVZ 和带附加条件的免费额度。价格会变，只作量级参照。

| 服务商 | 约 1 GB 档月价 | 磁盘 | 备注 |
| --- | --- | --- | --- |
| DockerVPS | 99 ₽ | NVMe | 页面未标机房城市，不能默认是俄境内 |
| VDSina | 150 ₽ | NVMe | 俄罗斯 / 荷兰 / 德国 |
| Beget | 210 ₽ | NVMe | 俄罗斯 / 哈萨克斯坦 / 荷兰，另有 1 个月免费测试 |
| FirstVDS | 249 ₽ | SSD | 莫斯科 / 荷兰，最低档不是 NVMe |
| Timeweb Cloud | 290 ₽ | NVMe | 多地区，支持按小时计费 |
| Aeza | 378 ₽ | NVMe | 九个国家，按小时计费 |
| RUVDS | 390 ₽ | SSD | 表内 VPS-1 档；139 ₽ 那档是 HDD，带库的 bot 别买 |
| Fornex | 520 ₽ | NVMe | 荷兰 / 德国 / 摩尔多瓦 / 美国，也有俄罗斯 |
| REG.RU | 590 ₽ | SSD | 支持按小时 |
| Selectel | 1,140 ₽ | NVMe | 俄罗斯 / 爱沙尼亚 |

不想碰 nginx 和证书的话，PaaS 侧有另一条路。Bothost 的 Starter 免费且不限时长，但只有 1 个 bot、256 MB 内存、0.5 GB 磁盘，没有环境变量，重启后磁盘清零；Basic 99 ₽/月给到 5 个 bot、1 GB 内存、5 GB 磁盘和环境变量；Pro 250 ₽/月给到 20 个 bot、2 GB 内存。三档都没有 root，也没法像在 VPS 上那样自己用 Docker Compose 拼一套 PostgreSQL 加 Redis。

把这些数字和 SixtyNet 放在一起：LAX-STARTER 年付 $200，摊到每月约 $16.7，比表里最贵的一档还高一个量级。你为住宅 IP、2G 起步的带宽、5 TB 流量和 DDoS 防护付溢价。只跑 bot，这笔钱花得不值；把这些资源用起来，价格才成立。

## 三个容易踩的坑

**Docker 里的数据库没挂 volume。** 容器一重建，PostgreSQL 的数据就没了。`docker-compose.yml` 里给数据库目录配一个命名 volume 或者绑定宿主机目录，这是唯一能防住这件事的操作。

**拿免费档当生产环境。** Bothost 的免费 Starter 重启清盘、没有环境变量，意味着 token 只能硬写进代码，用户数据活不过一次重启。它适合验证流程，不适合接真实用户。

**以为便宜的那档一直便宜。** 最便宜的 VPS 只便宜到第一次出事：证书过期、进程崩溃、磁盘写满，没人替你重启。省下的钱会在第一个出问题的晚上花回去，而且是以用户投诉的形式。

## 按你的情况对号入座

单个 bot，只处理命令和回调，接受 long polling → 1 GB 就够，SixtyNet 这边是 LAX-STARTER，年付 $200。

bot 带数据库、带队列、要跑后台任务 → 2 GB 起步，同机还要跑 PostgreSQL 和 Redis 的话直接看 LAX-MICRO。

一定要用 webhook → 先确认 LAX 系列的 443 可用，确认不了就用 PaaS，Bothost Basic 的 99 ₽/月比买错一台机器便宜。

需要美国住宅 IP 做别的业务，bot 顺带挂上去 → IRVINE-PRO-STARTER，年付 $350，接受只跑 long polling、退款只退余额。

下单前确认两件事。第一，年付 $200 是 17% 折扣后的促销价，原价 $240，优惠绑定一整年，付款前在页面上核对当日价格和计费周期。第二，机房位置和你需要的端口，工单问清楚比看页面描述可靠。

👉 [到 SixtyNet 核对当日价格并开通 VPS](https://sixtynet.net/aff.php?aff=39)
