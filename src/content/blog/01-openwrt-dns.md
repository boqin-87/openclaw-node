---
title: 'OpenWrt 下让 Google/YouTube 更稳定的 DNS 思路'
description: '从症状定位到 DNS 分流，减少“有时能上有时不能上”的情况。'
pubDate: '2026-03-15'
category: '网络'
tags: ['openwrt', 'dns', 'google', 'youtube']
---

很多“网页打不开但节点没挂”的问题，本质是 DNS 解析污染或回源不稳定。

## 排查顺序

1. 先看代理是否连通（延迟、握手）
2. 再看域名解析是否正确
3. 最后看分流规则是否命中预期组

## 实践建议

- 国内域名尽量走国内 DNS（如 223.5.5.5）
- 国外域名走 DoH（如 Cloudflare / Google）
- 在 OpenClash 开启 DNS 劫持，避免终端绕过

这套方式通常能显著提升移动端访问稳定性。
