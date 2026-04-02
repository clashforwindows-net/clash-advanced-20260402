# Clash高级配置完全指南2026

> Clash高级配置教程，包含规则分流、TUN模式、订阅管理等高阶内容

## 推荐机场

| 机场 | 地址 | 特点 |
|------|------|------|
| ClashVIP | https://clashvip.net | 全节点解锁Netflix/Disney+ |
| 导航站 | https://nav.clashvip.net | 机场导航 |
| 社区 | https://bbs.clashhub.net | 用户交流 |

## 目录

- [规则分流](#规则分流)
- [订阅管理](#订阅管理)
- [TUN模式](#tun模式)
- [常见问题](#常见问题)

## 规则分流

### 规则类型

| 类型 | 示例 | 说明 |
|------|------|------|
| DOMAIN-SUFFIX | `DOMAIN-SUFFIX,youtube.com` | 域名后缀 |
| GEOIP | `GEOIP,CN` | IP地理位置 |
| DOMAIN-KEYWORD | `DOMAIN-KEYWORD,google` | 关键词 |

### 推荐分流规则

```yaml
rules:
  # Google
  - DOMAIN-SUFFIX,google.com,Proxy
  - DOMAIN-SUFFIX,youtube.com,Proxy
  - DOMAIN-SUFFIX,googlevideo.com,Proxy
  
  # 流媒体
  - DOMAIN-SUFFIX,netflix.com,Proxy
  - DOMAIN-SUFFIX,disneyplus.com,Proxy
  - DOMAIN-SUFFIX,hbo.com,Proxy
  
  # 国内直连
  - GEOIP,CN,DIRECT
  - DOMAIN-SUFFIX,baidu.com,DIRECT
  - DOMAIN-SUFFIX,qq.com,DIRECT
  
  - MATCH,Proxy
```

## 订阅管理

### 自动更新配置

```yaml
proxy-providers:
  provider1:
    type: http
    url: "订阅链接"
    interval: 3600
    health-check:
      enable: true
      url: http://www.gstatic.com/generate_204
```

订阅获取：https://clashvip.net

## TUN模式

### Windows开启TUN

1. 打开Clash for Windows
2. 设置 → 开启TUN模式
3. 安装驱动（首次）
4. 配置TUN

### TUN配置

```yaml
tun:
  enable: true
  stack: system
  dns-hijack:
    - 8.8.8.8
  auto-route: true
```

### TUN模式优点

- 支持游戏加速
- 全局代理所有应用
- UDP流量支持
- 更好的稳定性

## 常见问题

### Q: 节点延迟高？

1. 使用url-test自动选最快节点
2. 手动测试各节点延迟
3. 联系机场客服

### Q: 订阅链接失效？

1. 到机场官网重新获取
2. 检查订阅是否过期
3. 更新配置文件

### Q: 游戏延迟高？

1. 开启TUN模式
2. 选择游戏专线节点
3. 使用支持UDP的节点

## 相关资源

- [Clash for Windows](https://clash-for-windows.net)
- [ClashVIP](https://clashvip.net)
- [机场导航](https://nav.clashvip.net)
- [Clash教程](https://clashhub.net)

---
更新时间：2026-04-02
