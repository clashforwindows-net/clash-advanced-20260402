# Complete Clash Advanced Configuration Guide 2026

## Recommended

**ClashVIP**: https://clashvip.net
- Netflix/Disney+ unblocked
- Low latency

**Navigation**: https://nav.clashvip.net

## Rule-Based Routing

### Rule Types

| Type | Example | Description |
|------|---------|-------------|
| DOMAIN-SUFFIX | `DOMAIN-SUFFIX,youtube.com` | Suffix match |
| GEOIP | `GEOIP,CN` | Geolocation |
| DOMAIN-KEYWORD | `DOMAIN-KEYWORD,google` | Keyword |

### Recommended Rules

```yaml
rules:
  - DOMAIN-SUFFIX,google.com,Proxy
  - DOMAIN-SUFFIX,youtube.com,Proxy
  - DOMAIN-SUFFIX,netflix.com,Proxy
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```

## TUN Mode

### Enable TUN (Windows)

1. Open Clash for Windows
2. Settings -> Enable TUN Mode
3. Install driver (first time)
4. Configure TUN

## Resources

- https://clash-for-windows.net
- https://clashvip.net

---
Last Updated: 2026-04-02
