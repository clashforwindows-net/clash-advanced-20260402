# Clash Meta内核深度剖析与Mihomo生态实战

> 从Clash Premium到Clash.Meta/Mihomo内核的架构演进，涵盖规则引擎原理、DNS解析管线、TUN协议栈实现、sing-box横向对比、性能火焰图分析，面向需要理解内核机制的高级用户和开发者。

## 目录

- [第一章 Clash内核演进史](#第一章-clash内核演进史)
- [第二章 Meta内核架构解析](#第二章-meta内核架构解析)
- [第三章 规则引擎工作原理](#第三章-规则引擎工作原理)
- [第四章 DNS解析管线](#第四章-dns解析管线)
- [第五章 TUN协议栈实现](#第五章-tun协议栈实现)
- [第六章 代理协议适配层](#第六章-代理协议适配层)
- [第七章 sing-box vs Mihomo横向对比](#第七章-sing-box-vs-mihomo横向对比)
- [第八章 性能分析与火焰图](#第八章-性能分析与火焰图)
- [第九章 内核编译与自定义构建](#第九章-内核编译与自定义构建)
- [第十章 生产级部署案例](#第十章-生产级部署案例)
- [常见问题](#常见问题)
- [相关资源](#相关资源)

---

## 第一章 Clash内核演进史

### 1.1 时间线

| 时间 | 事件 | 意义 |
|------|------|------|
| 2016 | Dreamacro创建Clash | Go语言编写的代理内核诞生 |
| 2018 | Clash Premium发布 | 加入TUN模式、增强DNS |
| 2020 | Clash闭源Premium | 社区无法继续贡献核心功能 |
| 2021 | Clash.Metafork | 基于最后开源版本，社区接手开发 |
| 2022 | 新增VLESS/Reality支持 | 跟进Xray生态协议 |
| 2023 | 更名Mihomo | 因Clash商标问题更名 |
| 2024 | Mihomo成熟期 | 完善API、Ruleset、脚本支持 |
| 2026 | Mihomo + sing-box双雄格局 | 两个主流Go代理内核并存 |

### 1.2 分支对比

| 特性 | Clash Premium(停更) | Clash.Meta/Mihomo | Clash(原版开源) |
|------|---------------------|-------------------|-----------------|
| TUN模式 | ✅ 闭源实现 | ✅ 开源实现 | ❌ |
| VLESS/Reality | ❌ | ✅ | ❌ |
| Hysteria2 | ❌ | ✅ | ❌ |
| 规则集(RuleSet) | ❌ | ✅ | ❌ |
| 脚本过滤 | ❌ | ✅ 简版 | ❌ |
| DNS over QUIC | ❌ | ✅ | ❌ |
| MIT License | 部分 | ✅ | ✅ |

推荐使用 Mihomo 内核的客户端：[Clash for Windows](https://clash-for-windows.net)。

---

## 第二章 Meta内核架构解析

### 2.1 整体架构

```
┌──────────────────────────────────────────────────┐
│                   Mihomo 内核                     │
│                                                   │
│  ┌─────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ Inbound │  │  Router  │  │   Outbound   │    │
│  │ Handler │→→│  Engine  │→→│   Dialer     │    │
│  └─────────┘  └──────────┘  └──────────────┘    │
│       ↑            ↑              ↓              │
│  ┌─────────┐  ┌──────────┐  ┌──────────────┐    │
│  │  TUN    │  │   DNS    │  │   Proxy      │    │
│  │ Device  │  │ Resolver │  │   Chain      │    │
│  └─────────┘  └──────────┘  └──────────────┘    │
│       ↑            ↑              ↓              │
│  ┌─────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ HTTP/   │  │  Ruleset │  │   Provider    │    │
│  │ SOCKS/  │  │  Engine  │  │   Manager     │    │
│  │ Mixed   │  │          │  │               │    │
│  └─────────┘  └──────────┘  └──────────────┘    │
│                                                   │
│  ┌─────────────────────────────────────────┐     │
│  │            RESTful API (9090)           │     │
│  └─────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
```

### 2.2 数据流路径

```
应用发出流量
    │
    ├──→ HTTP/SOCKS/Mixed Inbound (监听 7890)
    │    或
    ├──→ TUN Inbound (虚拟网卡，捕获所有流量)
    │
    ↓
DNS解析阶段
    │
    ├──→ 查询域名类型规则
    ├──→ 如需要，通过代理或直连DNS解析
    ├──→ fake-ip模式：返回虚拟IP，后续匹配
    │
    ↓
规则匹配阶段
    │
    ├──→ 依次匹配 rules 列表
    ├──│  DOMAIN-SUFFIX/DOMAIN-KEYWORD → 字符串匹配
    ├──│  GEOIP → 查IP地理数据库
    ├──│  PROCESS-NAME → 进程名匹配
    ├──│  RULE-SET → 加载外部规则集
    ├──│  SCRIPT → 执行JS脚本
    │
    ↓
出站阶段
    │
    ├──→ DIRECT → 直接连接
    ├──→ REJECT → 拒绝连接
    ├──→ Proxy节点 → 通过代理协议连接
    ├──│  Shadowsocks → AEAD加密
    ├──│  VMess/VLESS → XTLS/Reality
    ├──│  Trojan → TLS伪装
    ├──│  Hysteria2 → QUIC传输
    ├──│  WireGuard → 内核级VPN
    │
    ↓
目标服务器
```

### 2.3 核心配置结构

```yaml
# mihomo-full-config.yaml — 完整的Mihomo配置结构

# === 基础设置 ===
mixed-port: 7890          # HTTP/SOCKS混合端口
allow-lan: false          # 是否允许局域网
bind-address: "*"         # 绑定地址
mode: rule                # rule/global/direct
log-level: info           # silent/error/warning/info/debug
ipv6: false               # IPv6支持

# === TUN模式 ===
tun:
  enable: true
  stack: system           # system/gvisor/mixed
  dns-hijack:
    - any:53              # 劫持所有DNS请求
  auto-route: true        # 自动设置路由
  auto-detect-interface: true

# === DNS解析 ===
dns:
  enable: true
  listen: 0.0.0.0:1053
  ipv6: false
  
  # fake-ip 模式
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - "*.lan"
    - "*.local"
    - "*.localhost"
    - "localhost.ptlogin2.qq.com"
    - "+.msftconnecttest.com"
    - "+.msftncsi.com"
  
  # 默认DNS（用于解析域名规则）
  default-nameserver:
    - 223.5.5.5
    - 119.29.29.29
  
  # 主DNS（通过代理或直连）
  nameserver:
    - https://dns.alidns.com/dns-query  # DoH
    - https://doh.pub/dns-query
  
  # 代理域名DNS（通过代理解析，防DNS泄漏）
  proxy-server-nameserver:
    - https://dns.google/dns-query
  
  # 分流DNS
  nameserver-policy:
    "geosite:cn":
      - https://dns.alidns.com/dns-query
      - https://doh.pub/dns-query
    "geosite:geolocation-!cn":
      - https://dns.google/dns-query
      - https://1.1.1.1/dns-query
  
  # fallback DNS
  fallback:
    - https://dns.google/dns-query
    - tls://1.1.1.1:853
  fallback-filter:
    geoip: true
    geoip-code: CN

# === 代理节点 ===
proxies:
  - name: "HK-SS"
    type: ss
    server: example.com
    port: 443
    cipher: aes-256-gcm
    password: "your-password"
    udp: true
    
  - name: "JP-VLESS"
    type: vless
    server: example.com
    port: 443
    uuid: "your-uuid"
    network: ws
    tls: true
    servername: example.com
    ws-opts:
      path: "/path"
      headers:
        Host: example.com
    
  - name: "US-Hysteria2"
    type: hysteria2
    server: example.com
    port: 443
    password: "your-password"
    sni: example.com

# === 代理组 ===
proxy-groups:
  - name: "🚀 节点选择"
    type: select
    proxies:
      - "♻️ 自动选择"
      - "HK-SS"
      - "JP-VLESS"
      - "US-Hysteria2"
  
  - name: "♻️ 自动选择"
    type: url-test
    url: http://www.gstatic.com/generate_204
    interval: 300
    tolerance: 50
    proxies:
      - "HK-SS"
      - "JP-VLESS"
      - "US-Hysteria2"
  
  - name: "🎬 流媒体"
    type: select
    proxies:
      - "🚀 节点选择"
      - "JP-VLESS"  # 日本解锁Netflix
      - "US-Hysteria2"  # 美国解锁Disney+

# === 规则 ===
rules:
  # 规则集
  - RULE-SET,https://raw.githubusercontent.com/.../netflix.yaml,🎬 流媒体
  - RULE-SET,https://raw.githubusercontent.com/.../telegram.yaml,🚀 节点选择
  
  # 基础规则
  - DOMAIN-SUFFIX,google.com,🚀 节点选择
  - DOMAIN-KEYWORD,youtube,🚀 节点选择
  - GEOIP,CN,DIRECT
  - MATCH,🚀 节点选择
```

订阅推荐：[ClashVIP](https://clashvip.net) — 提供高质量多协议节点订阅。

---

## 第三章 规则引擎工作原理

### 3.1 规则匹配算法

Mihomo的规则匹配使用**短路评估（Short-circuit evaluation）**：

```go
// 简化的规则匹配逻辑（伪代码）
func matchRules(metadata *Metadata, rules []Rule) (proxy string) {
    for _, rule := range rules {
        if rule.Match(metadata) {
            return rule.Target()  // 命中即返回，不继续匹配
        }
    }
    return "MATCH"  // 默认规则
}
```

**性能含义**：规则顺序很重要。高频匹配的规则应放在前面。

### 3.2 各类规则的匹配复杂度

| 规则类型 | 匹配算法 | 时间复杂度 | 备注 |
|---------|---------|-----------|------|
| DOMAIN | Hash查找 | O(1) | 最快 |
| DOMAIN-SUFFIX | Trie树 | O(k), k=域名层级 | 快 |
| DOMAIN-KEYWORD | 子串搜索 | O(n×m) | 较慢，避免高频使用 |
| IP-CIDR | Radix树 | O(log n) | 快 |
| GEOIP | MaxMind DB查找 | O(1) | 需加载GeoIP数据库 |
| PROCESS-NAME | 系统调用 | O(1) | 需要TUN模式 |
| RULE-SET | 取决于子规则 | 变化 | 外部加载，有缓存 |
| SCRIPT | JS引擎执行 | 不可预测 | 最慢，谨慎使用 |

### 3.3 规则优化建议

```yaml
# 优化前的规则（慢）
rules:
  - DOMAIN-KEYWORD,google,Proxy       # 关键词匹配，慢
  - DOMAIN-KEYWORD,youtube,Proxy      # 关键词匹配，慢
  - DOMAIN-SUFFIX,google.com,Proxy    # 后缀匹配，快
  - DOMAIN-SUFFIX,youtube.com,Proxy   # 后缀匹配，快
  - GEOIP,CN,DIRECT                   # IP匹配，中等

# 优化后的规则（快）—— 把高频和快速规则放前面
rules:
  # 1. 先匹配精确域名（O(1)或O(k)）
  - DOMAIN-SUFFIX,google.com,Proxy
  - DOMAIN-SUFFIX,youtube.com,Proxy
  - DOMAIN-SUFFIX,googlevideo.com,Proxy
  - DOMAIN-SUFFIX,ggpht.com,Proxy
  
  # 2. 再匹配IP段（O(log n)）
  - IP-CIDR,104.16.0.0/12,Proxy,no-resolve
  
  # 3. 最后用关键词兜底（O(n×m)，但此时大部分流量已匹配）
  - DOMAIN-KEYWORD,google,Proxy
  
  # 4. GEOIP兜底
  - GEOIP,CN,DIRECT
  
  # 5. MATCH最终兜底
  - MATCH,Proxy
```

### 3.4 RuleSet使用

```yaml
# 使用远程规则集（自动缓存）
rule-providers:
  netflix:
    type: http
    behavior: classical
    url: "https://raw.githubusercontent.com/.../Netflix.yaml"
    interval: 86400  # 每天更新
    path: ./ruleset/netflix.yaml
  
  telegram:
    type: http
    behavior: classical  
    url: "https://raw.githubusercontent.com/.../Telegram.yaml"
    interval: 86400
    path: ./ruleset/telegram.yaml
  
  cn-domain:
    type: http
    behavior: domain
    url: "https://raw.githubusercontent.com/.../cn-domain.yaml"
    interval: 86400
    path: ./ruleset/cn-domain.yaml

rules:
  - RULE-SET,netflix,🎬 流媒体
  - RULE-SET,telegram,🚀 节点选择
  - RULE-SET,cn-domain,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,🚀 节点选择
```

---

## 第四章 DNS解析管线

### 4.1 DNS解析流程图

```
应用请求 google.com
    │
    ↓
[DNS Hijack] (TUN模式劫持所有53端口)
    │
    ↓
[fake-ip检查] ──→ 在fake-ip范围内？
    │                    │
    │                   是→ 查fake-ip映射表 → 返回真实域名给规则引擎
    │                    │
    │                   否→ 继续解析
    ↓
[nameserver-policy分流]
    │
    ├── 域名匹配 geosite:cn？
    │   ├── 是 → 使用国内DNS (alidns/doh.pub)
    │   └── 否 → 使用国外DNS (google/cloudflare)
    │
    ↓
[fallback检查]
    │
    ├── 国内DNS返回的IP，检查GEOIP
    │   ├── IP在CN → 接受结果
    │   └── IP不在CN（DNS污染）→ 使用fallback DNS
    │
    ↓
[返回结果给应用]
    │
    ├── fake-ip模式：返回198.18.x.x虚拟IP
    └── redir-host模式：返回真实IP
```

### 4.2 fake-ip vs redir-host

| 特性 | fake-ip | redir-host |
|------|---------|------------|
| DNS解析速度 | 即时返回虚拟IP | 需真实解析 |
| 规则匹配 | 基于域名（准确） | 基于IP（可能误判） |
| 兼容性 | 个别应用不兼容 | 通用兼容 |
| DNS泄漏 | 不会泄漏 | 可能泄漏 |
| 推荐度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

### 4.3 防DNS泄漏配置

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  
  # 关键：代理服务器域名用安全DNS解析
  proxy-server-nameserver:
    - https://dns.google/dns-query
  
  # 关键：国内域名用国内DNS
  nameserver-policy:
    "geosite:cn,private":
      - https://dns.alidns.com/dns-query
      - 223.5.5.5
  
  # 关键：国外域名用国外DNS
  nameserver:
    - https://dns.google/dns-query
    - https://1.1.1.1/dns-query
  
  # Fallback防污染
  fallback:
    - https://dns.google/dns-query
    - tls://8.8.8.8:853
  fallback-filter:
    geoip: true
    geoip-code: CN
    ipcidr:
      - 240.0.0.0/4  # 被污染的IP段
```

---

## 第五章 TUN协议栈实现

### 5.1 三种TUN Stack对比

| Stack | 实现 | 性能 | 兼容性 | 适用场景 |
|-------|------|------|--------|---------|
| system | 操作系统网络栈 | 中 | 最好 | 通用场景 |
| gvisor | 用户态网络栈 | 低 | 好 | 需要隔离的场景 |
| mixed | 混合模式 | 高 | 好 | 推荐生产使用 |

### 5.2 TUN模式工作原理

```
┌──────────────────────────────────┐
│           应用进程               │
│        (浏览器/游戏等)           │
└──────────┬───────────────────────┘
           │ 系统调用
           ↓
┌──────────────────────────────────┐
│      操作系统网络栈              │
│   (TCP/IP协议栈)                │
└──────────┬───────────────────────┘
           │ 路由表指向tun设备
           ↓
┌──────────────────────────────────┐
│      TUN虚拟网卡                 │
│   (Mihomo创建的/dev/net/tun)     │
└──────────┬───────────────────────┘
           │ 读取IP包
           ↓
┌──────────────────────────────────┐
│    Mihomo TUN Handler           │
│  ┌─────────────┐                │
│  │ IP包解析    │ → 提取目标IP/端口│
│  │ TCP重组     │ → 重组TCP流     │
│  │ DNS劫持     │ → 拦截53端口    │
│  └─────────────┘                │
└──────────┬───────────────────────┘
           │ 转换为Metadata
           ↓
┌──────────────────────────────────┐
│      规则引擎 + DNS解析          │
│   (决定走代理还是直连)           │
└──────────┬───────────────────────┘
           │
           ↓
┌──────────────────────────────────┐
│      Outbound Dialer             │
│   (Shadowsocks/VMess/Trojan)     │
└──────────┬───────────────────────┘
           │ 加密隧道
           ↓
       目标服务器
```

### 5.3 TUN配置最佳实践

```yaml
tun:
  enable: true
  stack: mixed              # 推荐mixed模式
  dns-hijack:
    - any:53                # 劫持所有DNS请求
  auto-route: true          # 自动设置路由规则
  auto-detect-interface: true  # 自动检测物理网卡
  mtu: 1500                 # 标准MTU
  
  # 高级选项
  # strict-route: true     # 严格路由（防止流量绕过TUN）
  # rx-buffer: 8192        # 接收缓冲区大小
```

---

## 第六章 代理协议适配层

### 6.1 协议支持矩阵

| 协议 | Mihomo支持 | 加密方式 | 传输层 | 特点 |
|------|-----------|---------|--------|------|
| Shadowsocks | ✅ | AEAD-2022 | TCP/UDP | 轻量高效 |
| ShadowsocksR | ✅ | 混淆+协议 | TCP | 已过时，不推荐 |
| VMess | ✅ | AES-128-GCM | TCP/WS/gRPC | V2Ray生态 |
| VLESS | ✅ | 无加密(依赖TLS) | TCP/WS/gRPC/HTTP2 | Xray生态 |
| VLESS+Reality | ✅ | Reality握手 | TCP | 最强抗检测 |
| Trojan | ✅ | TLS伪装 | TCP/WS | 伪装HTTPS |
| Hysteria2 | ✅ | QUIC加密 | UDP(QUIC) | 高速抗丢包 |
| WireGuard | ✅ | ChaCha20 | UDP | 内核级VPN |
| TUIC | ✅ | QUIC加密 | UDP(QUIC) | 低延迟 |
| SSH | ✅ | SSH加密 | TCP | 跳板机场景 |

### 6.2 协议选择决策

```
需要抗审查(防检测)？
├── 是 → VLESS+Reality (最强伪装)
│       或 Trojan (TLS伪装HTTPS)
│
├── 否，需要高速？
│   ├── UDP环境好 → Hysteria2 (QUIC高速)
│   ├── UDP环境差 → Shadowsocks AEAD (TCP稳定)
│   └── 需要低延迟 → TUIC (QUIC低延迟)
│
└── 否，需要通用兼容？
    └── VMess (最广泛支持)
        或 Shadowsocks (最轻量)
```

优质多协议节点订阅：[ClashVIP](https://clashvip.net) — 支持Shadowsocks/VMess/VLESS/Trojan/Hysteria2等全协议。

---

## 第七章 sing-box vs Mihomo横向对比

### 7.1 架构对比

| 维度 | Mihomo (Clash.Meta) | sing-box |
|------|---------------------|----------|
| 语言 | Go | Go |
| 配置格式 | YAML | JSON |
| 规则引擎 | 顺序匹配+RuleSet | 顺序匹配+规则集 |
| DNS | fake-ip+redir-host | fake-ip+redir-host |
| TUN | system/gvisor/mixed | system/gvisor |
| API | RESTful (9090) | RESTful (9090) |
| 脚本支持 | JavaScript | 无 |
| 配置热重载 | ✅ | ✅ |
| 性能 | 中上 | 高（更优化的内存管理） |
| 社区活跃度 | 高 | 高 |
| 生态客户端 | Clash Verge/Clash for Windows | sing-box GUI/NekoBox |

### 7.2 性能基准测试

```bash
#!/bin/bash
# benchmark-mihomo-vs-singbox.sh — 简单性能对比
# 需要：mihomo + sing-box + iperf3 + curl

echo "=== Mihomo vs sing-box Performance Benchmark ==="

# 测试1：TCP吞吐量
echo "[1] TCP Throughput (iperf3 through proxy)"
echo "  Mihomo:"
iperf3 -c 127.0.0.1 -p 7890 -t 10 -P 4 2>/dev/null | grep "SUM.*bits" | tail -1
echo "  sing-box:"
iperf3 -c 127.0.0.1 -p 2080 -t 10 -P 4 2>/dev/null | grep "SUM.*bits" | tail -1

# 测试2：HTTP延迟
echo "[2] HTTP Latency (100 requests)"
echo "  Mihomo:"
for i in $(seq 1 100); do
    curl -o /dev/null -s -w "%{time_total}\n" --proxy http://127.0.0.1:7890 https://www.google.com
done | awk '{sum+=$1;count++}END{printf "  avg: %.0fms\n", sum/count*1000}'

echo "  sing-box:"
for i in $(seq 1 100); do
    curl -o /dev/null -s -w "%{time_total}\n" --proxy http://127.0.0.1:2080 https://www.google.com
done | awk '{sum+=$1;count++}END{printf "  avg: %.0fms\n", sum/count*1000}'

# 测试3：内存占用
echo "[3] Memory Usage"
echo "  Mihomo: $(ps aux | grep mihomo | grep -v grep | awk '{print $6/1024 "MB"}')"
echo "  sing-box: $(ps aux | grep sing-box | grep -v grep | awk '{print $6/1024 "MB"}')"
```

### 7.3 选择建议

| 用户类型 | 推荐 | 原因 |
|---------|------|------|
| Clash老用户 | Mihomo | 配置兼容，迁移成本低 |
| 新用户 | sing-box | 性能更优，JSON配置更规范 |
| 需要脚本过滤 | Mihomo | 支持JavaScript规则 |
| 追求极致性能 | sing-box | 内存管理更优 |
| 需要GUI客户端 | Mihomo | Clash生态客户端更丰富 |

---

## 第八章 性能分析与火焰图

### 8.1 开启pprof

```yaml
# mihomo配置中开启pprof
profile:
  store-selected: true
  store-fake-ip: true
  # 开启pprof（默认端口9090同API端口）
  # 访问 http://localhost:9090/debug/pprof/
```

### 8.2 生成火焰图

```bash
# 安装pprof工具
go install github.com/google/pprof@latest

# 采集CPU profile（30秒）
pprof -http=:8080 http://localhost:9090/debug/pprof/profile?seconds=30

# 采集内存 profile
pprof -http=:8080 http://localhost:9090/debug/pprof/heap

# 采集goroutine profile
pprof -http=:8080 http://localhost:9090/debug/pprof/goroutine
```

### 8.3 性能优化检查清单

- [ ] 规则顺序优化（高频规则在前）
- [ ] 避免大量DOMAIN-KEYWORD规则
- [ ] 使用RuleSet替代超长inline规则
- [ ] 开启fake-ip模式（减少DNS解析延迟）
- [ ] TUN使用mixed stack
- [ ] 关闭不必要的日志（log-level设为warning）
- [ ] proxy-provider health-check间隔不要太短（建议300s）
- [ ] 避免过多proxy-group嵌套

---

## 第九章 内核编译与自定义构建

### 9.1 从源码编译

```bash
#!/bin/bash
# build-mihomo.sh — 编译Mihomo内核

# 1. 安装Go
wget -qO- https://go.dev/dl/go1.22.0.linux-amd64.tar.gz | tar -C /usr/local -xzf -
export PATH=$PATH:/usr/local/go/bin

# 2. 克隆源码
git clone https://github.com/MetaCubeX/mihomo.git
cd mihomo

# 3. 编译（带GEOIP数据库）
mkdir -p bin

# 标准版本
CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" \
    -tags "with_gvisor" \
    -o bin/mihomo \
    ./main.go

# 便携版本（内嵌GEOIP数据库，文件更大但无需额外下载）
CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" \
    -tags "with_gvisor,with_embed" \
    -o bin/mihomo-portable \
    ./main.go

# 4. 验证
./bin/mihomo -v
```

### 9.2 自定义修改示例

```go
// 添加自定义规则类型示例
// 在 rules/ 目录下创建 custom_rule.go

package rules

import (
    "strings"
    "github.com/metacubex/mihomo/component/metadata"
)

// CustomTimeRule: 根据时间匹配规则
type CustomTimeRule struct {
    startHour int
    endHour   int
    target    string
}

func (r *CustomTimeRule) Match(metadata *metadata.Metadata) bool {
    hour := time.Now().Hour()
    return hour >= r.startHour && hour < r.endHour
}

func (r *CustomTimeRule) Target() string {
    return r.target
}

// 注册规则
func init() {
    // 需要在parser中注册解析逻辑
}
```

---

## 第十章 生产级部署案例

### 10.1 企业网关部署

```yaml
# enterprise-gateway.yaml — 企业级网关配置

# 基础设置
mixed-port: 7890
allow-lan: true
bind-address: 0.0.0.0
mode: rule
log-level: warning
ipv6: false

# TUN模式
tun:
  enable: true
  stack: mixed
  dns-hijack:
    - any:53
  auto-route: true

# DNS
dns:
  enable: true
  enhanced-mode: fake-ip
  nameserver:
    - https://dns.google/dns-query
  nameserver-policy:
    "geosite:cn":
      - https://dns.alidns.com/dns-query

# 代理节点（从provider加载）
proxy-providers:
  main:
    type: http
    url: "订阅链接"
    interval: 3600
    health-check:
      enable: true
      url: http://www.gstatic.com/generate_204
      interval: 300

# 代理组
proxy-groups:
  - name: "主出口"
    type: url-test
    use:
      - main
    url: http://www.gstatic.com/generate_204
    interval: 300
    tolerance: 50

  - name: "备份出口"
    type: fallback
    use:
      - main
    url: http://www.gstatic.com/generate_204
    interval: 60

# 规则
rule-providers:
  cn:
    type: http
    behavior: domain
    url: "https://raw.githubusercontent.com/.../cn.yaml"
    interval: 86400

rules:
  - RULE-SET,cn,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,主出口
```

### 10.2 多WAN负载均衡

```yaml
# 多代理链路负载均衡
proxy-groups:
  - name: "负载均衡-轮询"
    type: load-balance
    strategy: round-robin
    proxies: [HK-01, JP-01, US-01]
    url: http://www.gstatic.com/generate_204
    interval: 300
  
  - name: "负载均衡-一致性哈希"
    type: load-balance
    strategy: consistent-hashing
    proxies: [HK-01, JP-01, US-01]
    url: http://www.gstatic.com/generate_204
    interval: 300
    # 一致性哈希保证同一目标IP走同一出口
    # 避免session分布在不同节点导致登录失效
```

---

## 常见问题

### Q: Mihomo和Clash.Meta是什么关系？

A: Clash.Meta因Clash商标问题更名为Mihomo。两者是同一个项目，Mihomo是当前名称，Clash.Meta是历史名称。

### Q: fake-ip模式下某些应用异常怎么办？

A: 将问题域名加入fake-ip-filter列表。常见需要排除的域名包括：局域网域名、本地解析域名、某些登录认证域名。参考配置中的fake-ip-filter示例。

### Q: TUN模式和HTTP代理模式有什么区别？

A: TUN模式在网卡层面捕获所有流量（包括不遵守系统代理设置的应用），HTTP代理模式只捕获遵守系统代理的流量。需要全局代理（如游戏）时用TUN，只需要浏览器代理时用HTTP。

### Q: 如何选择Mihomo客户端？

A: 推荐使用 [Clash for Windows](https://clash-for-windows.net)。其他选择包括Clash Verge Rev（跨平台）、FlClash（移动端）等。

### Q: sing-box和Mihomo该选哪个？

A: 如果你已经在用Clash生态，选Mihomo（迁移成本低）。如果是新部署，两个都值得考虑。性能差距在10%以内，不是决定性因素。

---

## 相关资源

- [Clash for Windows](https://clash-for-windows.net) — 推荐客户端
- [ClashVIP](https://clashvip.net) — 优质机场订阅（全协议支持）
- [机场导航](https://nav.clashvip.net) — 机场对比选择
- [Clash教程](https://clashhub.net) — Clash使用教程合集
- [用户社区](https://bbs.clashhub.net) — 技术讨论社区
- [VPSVIP](https://vpsvip.net) — VPS自建节点推荐

## 免责声明

1. 本仓库提供技术分析，不构成使用建议
2. 请遵守当地法律法规使用代理工具
3. 内核编译和自定义修改需自行承担风险
4. 性能数据为参考值，实际表现取决于网络环境和硬件配置

## 许可证

MIT License

---
更新时间：2026-09-18 | 主题：Clash Meta内核深度剖析与Mihomo生态实战
