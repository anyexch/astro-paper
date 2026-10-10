---
author: anyexch
pubDatetime: 2026-10-10T12:35:00+08:00
title: 我把 cloudflared 装进 OpenWrt：用 WARP 从外网安全访问家庭内网
featured: false
draft: false
tags:
  - Cloudflare Tunnel
  - OpenWrt
  - WARP
  - self-hosting
  - project-retrospective
description: 复盘一次家庭私网远程访问实践：让 VMware 中常驻的 OpenWrt 运行 cloudflared，通过 Cloudflare Tunnel、Zero Trust 和 WARP 从移动网络访问内网，同时梳理它与 Workers、端口映射及远程开机的区别。
---

我希望在外面也能安全访问家里的 NAS、路由器和其他内网服务，但又不想把管理端口直接暴露到公网，也不想依赖一台 Windows 电脑长期开机做代理。

最后采用的方案，是让 VMware 中常驻的 OpenWrt 运行 `cloudflared`，主动连接 Cloudflare；手机或外部电脑则使用 Cloudflare One Agent／WARP，通过 Zero Trust 身份认证后获得家庭私网路由。整个过程不需要家庭宽带具备公网 IP，也不需要在路由器上开放公网入站端口。

这篇文章记录已经完成并真实验证的私网访问主链路，以及它还可以怎样扩展到 Wake-on-LAN 远程开机。

## 最终架构

数据路径可以简化成：

```text
外部手机或电脑
    ↓ Cloudflare One Agent / WARP
Cloudflare Zero Trust
    ↓ Private Network Route
Cloudflare Tunnel
    ↓ cloudflared 主动建立的出站连接
OpenWrt 虚拟机
    ↓
家庭局域网中的 NAS、Web、SSH 和其他服务
```

OpenWrt 是家中常在线的连接器。客户端只有在需要回到家庭内网时才连接 WARP，并且必须先通过指定账户的身份验证。

## OpenWrt 上真正安装的是什么

OpenWrt 上安装的是 Cloudflare 官方的 `cloudflared`，而不是 WARP 客户端，也不是自建的 Cloudflare Workers 程序。

我为它做了几项基础配置：

- 使用与 OpenWrt 架构匹配的官方二进制，并在安装前核对版本和摘要；
- 把 Tunnel Token 保存在只允许管理员读取的独立文件中；
- 使用 OpenWrt 的 `procd` 注册为系统服务；
- 设置开机自启、异常退出后自动重启；
- 先检查 DNS、Cloudflare API 以及 Tunnel 所需网络连通性，再启动长期连接。

这样即使日常电脑关机，OpenWrt 仍能保持到 Cloudflare 的出站 Tunnel。

## Cloudflare 端配置了什么

Cloudflare 端完成了三层配置：

1. 创建一个远程管理的 Cloudflare Tunnel，并让 OpenWrt 上的 `cloudflared` 连接它；
2. 把家庭局域网网段作为 Private Network Route 交给该 Tunnel；
3. 配置 WARP 设备注册和身份策略，只允许指定账户加入组织。

这里没有使用 Workers 来转发家庭内网流量。Workers 内网穿透是另一条曾经记录过的开发设想，但没有进入本次实施，也不能把它写成已经部署完成。

## 最容易踩坑的地方

### 私网地址可能被 Split Tunnel 排除

WARP 客户端通常会默认排除一部分私网地址。如果家庭网段仍落在宽泛排除规则中，客户端会直接在当前网络寻找目标地址，而不会把流量交给 Cloudflare。

本次做法是只调整目标家庭网段需要经过 WARP 的范围，避免粗暴改变其他本地网络的访问行为。

### 注册成功不等于数据通道正常

设备能够登录组织，只代表身份注册完成。WARP 数据通道仍可能与电脑上的其他 VPN、TUN 或代理软件争抢默认路由。

Windows 测试机就遇到了现有代理与 WARP 的路由冲突，因此没有强行停止正在使用的代理服务。最终验收改在手机移动网络上完成，这也更接近真正的远程使用场景。

### Zero Trust 不能替代源服务认证

WARP 和 Zero Trust 控制的是“谁能进入这条私网路径”，并不会自动替代 NAS、SSH、OpenWrt 或其他后台自己的账号、密钥和多因素认证。

即使外部设备已经通过 Cloudflare 验证，内网服务本身仍应保留原有访问控制。

## 已经完成的验证

本次已经完成的结果包括：

- OpenWrt 上的 `cloudflared` 正常运行并设置为开机自启；
- Cloudflare Tunnel 保持健康连接；
- 家庭私网路由已经关联到 Tunnel；
- WARP 设备注册只允许指定身份加入；
- 手机关闭家庭 Wi-Fi、切换移动数据后，可以经 WARP 访问家庭内网中的 OpenWrt 管理页面；
- 整个链路没有开放家庭公网入站端口。

这证明“外部设备 → WARP → Cloudflare → Tunnel → OpenWrt → 家庭局域网”的主链路已经成立。

## 怎样扩展到远程开机

私网通道建立后，可以继续利用 OpenWrt 或另一台常在线设备发送 Wake-on-LAN 魔术包，从外部唤醒已经关机的本地主机。WARP 负责让我安全进入家庭网络，真正执行唤醒的是内网中的 Wake-on-LAN 工具，两者不是同一个功能。

要让这条链路可靠工作，目标主机还需要满足这些条件：

- 主板、网卡和操作系统已经启用 Wake-on-LAN；
- 关机后网卡仍保持待机供电；
- 常在线设备能够向目标二层网络发送魔术包；
- 已记录目标网卡信息，并限制谁可以触发唤醒；
- 分别验证正常关机、断电恢复和不同网络环境下的唤醒结果。

现有项目记录已经证明远程私网访问可用，但还没有保存一次完整的远程开机端到端验收。因此远程开机目前应视为下一步扩展能力，而不是已经完成的项目结论。

## 这套方案适合什么场景

它适合个人 NAS、家庭实验室、内网 Web 工具、SSH 和只对本人开放的管理服务。相比传统端口映射，它减少了公网暴露面，也不要求家庭宽带拥有公网 IP。

它仍然有清晰的边界：家庭断电、OpenWrt 虚拟机停止、运营商网络异常或 Cloudflare 路径不可用时，远程访问仍会中断。当前只有一个常驻连接器，也还需要继续验证重启恢复和长期运行稳定性。

对我来说，这个项目最有价值的地方不是“又装了一个网络工具”，而是把远程访问拆成了身份、路由、隧道、内网服务和设备唤醒几个边界明确的部分。以后无论接入 NAS、开发机还是其他自托管服务，都可以沿用同一套判断方法。
