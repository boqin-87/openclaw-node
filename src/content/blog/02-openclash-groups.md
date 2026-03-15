---
title: 'OpenClash 代理组设计：自动选择 + 手动兜底'
description: '一套可维护的代理组结构，兼顾速度、可控性和容错。'
pubDate: '2026-03-15'
category: '网络'
tags: ['openclash', 'proxy', 'rule']
---

推荐保留三层：

- **自动选择**：日常默认
- **手动切换**：排障时快速改线
- **业务组**：YouTube、AI、Telegram 分开

这样在某个业务异常时，不会影响全部流量。
