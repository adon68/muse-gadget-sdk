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

## 连接 Muse 网络

设备需要能访问 Muse 云服务（例如 `hatch.metaaivm.com` 等域名）。在受限网络环境下，本 fork 在设置页 **MUSE** 中增加了 **Relay server IP**：

- 将板子的 DNS 指向你自建的中继；
- 中继侧用 DNS 把 Muse 相关域名解析回中继自身，并通过 **443 SNI / 透传** 转发到官方服务；
- **Relay 为空**时行为与官方一致；
- 保存 Relay IP 后会**立即应用 DNS**（无需刻意断网重连）。

示例填写：`你的中继公网IP`（请勿把真实 IP 写进仓库文档或提交到 git）。

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
