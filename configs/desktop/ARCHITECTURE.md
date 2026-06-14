# Архитектура конфига

## Общая схема прохождения трафика

```mermaid
flowchart TD
    APP([Приложение / ОС]) --> TUN["tun-in\n172.19.0.1/30\nmixed · strict_route"]
    APP --> MIX["mixed-in\n127.0.0.1:2080\nHTTP / SOCKS5"]

    TUN --> R{Маршрутизатор}
    MIX --> R

    R -->|"protocol: dns"| DNSOUT[hijack-dns]
    R -->|"bittorrent"| DIR[direct]
    R -->|"icmp / port 123 NTP"| DIR
    R -->|"192.168.123.0/24"| WGS["wg-sample\nWireGuard endpoint"]
    R -->|"ip_is_private"| DIR
    R -->|"youtube · reddit"| HY2["hy2-sample2\nHysteria2 port hopping"]
    R -->|"cursor · claude · anthropic\nClaude Helper"| VLS["vls-sample\nVLESS·REALITY·gRPC"]
    R -->|"Telegram"| SEL
    R -->|".ru · geoip-ru\ngeosite-ru"| DIR
    R -->|"всё остальное"| SEL{{"proxy selector\n[default: hy2-sample]"}}

    SEL --> HY1["hy2-sample\nHysteria2"]
    SEL --> HY2
    SEL --> VLS
    SEL --> DIR

    style WGS fill:#e8f4f8,stroke:#5b9bd5
    style SEL fill:#fff3cd,stroke:#ffc107
    style DIR fill:#d4edda,stroke:#28a745
    style DNSOUT fill:#f8d7da,stroke:#dc3545
```

> WireGuard-туннели (`wg-*`) **не входят в selector** — они используются только для доступа к корпоративным сетям по конкретным IP-подсетям и работают всегда, независимо от выбранного прокси.

---

## DNS

```mermaid
flowchart LR
    Q([DNS-запрос]) --> DR{Правила DNS}

    DR -->|"geosite-ru\ngeosite-category-ru\n.ru · .рф · .su · ..."| DD["dns-direct\nYandex 77.88.8.8\nчерез direct"]
    DR -->|"всё остальное"| DP["dns-remote\nGoogle 8.8.8.8 DoH\nчерез proxy"]

    style DD fill:#d4edda,stroke:#28a745
    style DP fill:#cce5ff,stroke:#004085
```

---

## Секции конфига

### Inbounds

| Тег | Тип | Адрес | Назначение |
|-----|-----|-------|-----------|
| `tun-in` | TUN | 172.19.0.1/30 | Перехват всего трафика ОС через виртуальный сетевой интерфейс |
| `mixed-in` | Mixed | 127.0.0.1:2080 | HTTP/SOCKS5 для приложений с ручной настройкой прокси |

### Outbounds

| Тег | Протокол | Особенности |
|-----|----------|------------|
| `proxy` | Selector | Ручной выбор прокси через Clash API |
| `hy2-sample` | Hysteria2 | obfs salamander, фиксированный порт |
| `hy2-sample2` | Hysteria2 | obfs salamander, port hopping (20000–50000) |
| `vls-sample` | VLESS | REALITY + gRPC + uTLS chrome |
| `wg-sample` | WireGuard | Корпоративная сеть (endpoint, не входит в selector) |
| `direct` | Direct | Прямое подключение |

### Правила маршрутизации (в порядке приоритета)

| # | Условие | Аутбаунд |
|---|---------|---------|
| 1 | `inbound: tun-in / mixed-in` | action: sniff |
| 2 | `protocol: dns` | action: hijack-dns |
| 3 | `protocol: bittorrent` | `direct` |
| 4 | `network: icmp` | `direct` |
| 5 | `port: 123` (NTP) | `direct` |
| 6 | IP в подсети WireGuard | `wg-sample` |
| 7 | Все остальные приватные IP | `direct` |
| 8 | Домен содержит `youtube.com`, `reddit` | `hy2-sample2` |
| 9 | Домен содержит `cursor`, `claude`, `anthropic` | `vls-sample` |
| 10 | Процесс `Claude Helper` (macOS) | `vls-sample` |
| 11 | Процесс `Telegram` (macOS) | `proxy` (selector) |
| 12 | Домен `.ru`/`.su`/`.xn--p1ai` ИЛИ `geoip-ru` / `geosite-category-ru` | `direct` |
| — | Всё остальное | `proxy` (selector) |

> **Важно:** WireGuard-подсети (`allowed_ips`) обязательно должны стоять в правилах **до** `ip_is_private`, иначе приватные адреса (10.x, 192.168.x и т.д.) перехватятся раньше и пойдут в `direct`, минуя туннель.

> **Кросс-платформенность:** правила `process_name` зависят от платформы. На macOS используются имена без расширения (`Telegram`, `Claude Helper`). На **Windows** нужно указывать имя исполняемого файла с `.exe` (`Telegram.exe`, `claude.exe`). На **Linux** имя зависит от дистрибутива (`telegram-desktop`, `telegram` и т.д.). Если имя процесса не совпадает — правило просто пропускается, трафик уходит по следующим условиям.

### Endpoints

WireGuard-туннели вынесены в секцию `endpoints` (не `outbounds`) — это требование sing-box 1.11+. В route rules они по-прежнему доступны по тегу как обычные аутбаунды. Для каждого туннеля указывается поле `allowed_ips` — только трафик к этим подсетям пойдёт через туннель.

Если на сервере WireGuard настроен `PreSharedKey`, его необходимо указать в поле `pre_shared_key` у peer — без него handshake завершится с ошибкой `received invalid response message`.

### Rule sets

Правила GeoIP и GeoSite скачиваются автоматически при первом запуске из репозитория [runetfreedom/russia-v2ray-rules-dat](https://github.com/runetfreedom/russia-v2ray-rules-dat):

| Тег | Файл |
|-----|------|
| `geoip-ru` | `rule-set-geoip/geoip-ru.srs` |
| `geosite-category-ru` | `rule-set-geosite/geosite-category-ru.srs` |
| `geosite-ru-available-only-inside` | `rule-set-geosite/geosite-ru-available-only-inside.srs` |
