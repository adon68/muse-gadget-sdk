<!--
Copyright (c) Meta Platforms, Inc. and affiliates.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Muse Gadgets

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset=".github/images/muse-gadgets-dark.png">
    <img src=".github/images/muse-gadgets-light.png" width="900" alt="Muse gadgets: a Waveshare round AMOLED, an M5Stack StickS3, Muse Home Link, a Raspberry Pi and a Seeed reTerminal e-ink display">
  </picture>
</p>

## 这是什么

本仓库是官方 [Muse Gadgets SDK](https://github.com/facebookincubator/muse-gadget-sdk) 的 fork，面向 Waveshare **ESP32-S3-Touch-AMOLED-1.75C** 等板型，在上游基础上增加受限网络场景下的连接能力。

| 目录 | 说明 |
|---|---|
| [**esp32**](esp32) | ESP32 设备 SDK / 固件（含本 fork 的 Relay 设置） |
| [**linux**](linux) | Linux / Raspberry Pi 等设备 SDK |

## 连接 Muse 网络 / Relay

设备需要能访问 Muse 云服务（例如 `hatch.metaaivm.com`、`api.muse.ai` 等域名）。在受限网络环境下，本 fork 在设置页 **MUSE** 中增加了 **Relay server IP**：

- 将板子的 DNS 指向你自建的中继；
- 中继侧用 DNS 把 Muse 相关域名解析回中继自身，并通过 **443 SNI / 透传** 转发到官方服务；
- **Relay 为空**时行为与官方一致；
- 保存 Relay IP 后会**立即应用 DNS**（无需刻意断网重连）。

示例填写：`RELAY_PUBLIC_IP`（请勿把真实 IP、token、邮箱写进仓库文档或提交到 git）。

### 中继 VPS 配置

以下以 **Ubuntu** 为例，给出通用、可复现的中继搭建步骤。全程用占位符 `RELAY_PUBLIC_IP`，不要把真实公网 IP / 凭证写进文档或 git。

#### 架构说明

1. **板子**：DNS 设为中继公网 IP（`RELAY_PUBLIC_IP`）。
2. **中继 dnsmasq**：仅将 `hatch.metaaivm.com`、`api.muse.ai` 的 A 记录指到中继自身；其它域名走正常上游解析。
3. **中继 443**：按 SNI 把上述域名 **TLS 透传**到官方上游。若 VPS 上 443 已被代理占用，应在**现有代理**（如 xray）上增加 SNI 分流，而不是再抢端口；也可用 nginx `stream` + `ssl_preread`（注意 `stream` 不能塞进 `http` 的 `conf.d`）。

```
板子 DNS → 中继:53 (dnsmasq)
板子 HTTPS → 中继:443 (SNI 透传) → 官方上游
```

#### 1. 安装并配置 dnsmasq

```bash
sudo apt update
sudo apt install -y dnsmasq
```

建议使用独立配置文件（勿把真实 IP 写死进仓库）：

```bash
# /etc/dnsmasq.d/muse-relay.conf
# 将 RELAY_PUBLIC_IP 换成你的中继公网 IP
address=/hatch.metaaivm.com/RELAY_PUBLIC_IP
address=/api.muse.ai/RELAY_PUBLIC_IP
# 其它域名走系统上游（或在此指定 server=）
# server=1.1.1.1
# server=8.8.8.8
```

```bash
sudo systemctl enable --now dnsmasq
sudo systemctl restart dnsmasq
```

若本机已有 `systemd-resolved` 占用 53，需先调整 resolved / 让 dnsmasq 监听公网网卡，再重启 dnsmasq。

#### 2. 443 SNI 透传（二选一）

**方案 A：已有 xray / 同类代理占 443**

不要再开第二个 443 监听。在现有入站前增加 **SNI 分流**：当 `server_name` 为 `hatch.metaaivm.com` 或 `api.muse.ai` 时，将流量转发到官方上游（域名直连或解析得到的上游地址）；其余流量保持原有代理逻辑。具体字段因代理软件而异，原则是 **SNI 匹配 → 纯 TCP/TLS 透传，不解密**。

**方案 B：nginx stream + ssl_preread（443 空闲时）**

`stream` 块必须写在 `http` 之外（例如 `/etc/nginx/nginx.conf` 顶层，或单独 `stream` 配置目录），**不能**放进 `conf.d` 里的 http 站点配置。

```nginx
# 示例：nginx stream（占位，勿提交真实 IP）
stream {
    map $ssl_preread_server_name $muse_backend {
        hatch.metaaivm.com  hatch.metaaivm.com:443;
        api.muse.ai         api.muse.ai:443;
        default             "";  # 非 Muse 域名可拒绝或转到其它服务
    }

    server {
        listen 443;
        ssl_preread on;
        proxy_pass $muse_backend;
        proxy_connect_timeout 5s;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

#### 3. 防火墙

放行 DNS 与 HTTPS（示例用 ufw；按你实际防火墙工具调整）：

```bash
sudo ufw allow 53/udp
sudo ufw allow 443/tcp
# 强烈建议按板子出口 IP 限源，避免 53 对全公网开放被滥用，例如：
# sudo ufw delete allow 53/udp
# sudo ufw allow from BOARD_EGRESS_IP to any port 53 proto udp
```

#### 4. 验证

在任意能访问中继的机器上（把 `RELAY_PUBLIC_IP` 换成实际中继 IP，仅用于本地验证，勿写回仓库）：

```bash
# DNS：应返回中继自身 IP
dig @RELAY_PUBLIC_IP hatch.metaaivm.com +short
dig @RELAY_PUBLIC_IP api.muse.ai +short

# TLS：应看到官方（Meta）相关证书，而不是自签证书
openssl s_client -connect RELAY_PUBLIC_IP:443 -servername hatch.metaaivm.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer
```

#### 5. 板子侧设置

1. 设备 **设置 → MUSE → Relay server IP**
2. 填入中继公网 IP（`RELAY_PUBLIC_IP`）
3. 保存 —— 本 fork 会**立即应用 DNS**（无需刻意断网重连）

#### 安全提醒

- **53/udp 对公网完全开放有被滥用（开放解析器）风险**，建议仅允许板子出口 IP（或你的家宽/办公出口）访问。
- 不要把真实公网 IP、SDK token、邮箱、具体主机名写进 README 或提交到 git。

## SDK Token

到 [gadgets.muse.ai](https://gadgets.muse.ai/settings/sdk-tokens) 申请 SDK token，写入本地 `sdkconfig`（常见为 `esp32/devices/sdkconfig.muse.token`），**勿提交到 git**。同时请阅读 [Gadget SDK Terms](https://gadgets.muse.ai/sdk-terms)。

## 编译与刷入

依赖 **ESP-IDF**（与上游一致，当前常用 **v6.x**）。板型别名 `s3` 对应 `waveshare-s3-175c`。

在 `esp32` 目录下执行：

```bash
# 编译
tools/muse/board.sh build s3

# 刷入（串口以本机为准，例如 /dev/cu.usbmodemXXX）
tools/muse/board.sh flash s3 /dev/cu.usbmodemXXX
```

建议不要随意 `erase-flash`，以免清掉 Wi‑Fi / 配对等 NVS 数据。

## 配对

在 Muse App 中打开 **开发者模式**，然后到 **Devices** 中配对（设备名通常带 `MuseGadget` 前缀）。

各子目录另有 `README.md` 入门说明，以及面向编程助手的 `AGENTS.md`（可配合 [Muse Code](https://developer.meta.com/ai/lp/muse-code/)）。

## 社区

可在官方 [Discord](https://discord.gg/3bhjCkZdd6) 与其他开发者交流。

## 许可证

本项目采用 **Apache License 2.0**（见 [`LICENSE`](LICENSE)），下列第三方文件保留其上游许可证：

| Path | Upstream | License |
|---|---|---|
| [`esp32/components/minimp3/include/minimp3.h`](esp32/components/minimp3) | [lieff/minimp3](https://github.com/lieff/minimp3) | CC0-1.0，见 [`LICENSE`](esp32/components/minimp3/LICENSE) |
| [`esp32/main/pixel_font.c`](esp32/main/pixel_font.c) | Adafruit GFX `glcdfont.c` | BSD-2-Clause（文件头注明） |

构建时拉取的依赖另有各自许可证：ESP-IDF 组件（进入 `esp32/managed_components/`），模拟器的 LVGL 与 SDL（见 [`esp32/simulator/THIRD_PARTY.md`](esp32/simulator/THIRD_PARTY.md)）。

Apache 2.0 **不覆盖** [Jollybot 头像资源](esp32/avatar)。
