# sing-box-config

**English** | [Русский](README.ru.md)

[sing-box](https://github.com/SagerNet/sing-box) configuration with split routing for users in Russia. Corporate VPNs (WireGuard) always work, regardless of the active proxy.

Two schemes to choose from:

| Template | Default route | Exceptions |
|----------|---------------|------------|
| `desktop.example.json` | everything through the proxy | Russian resources go direct |
| `desktop-direct.example.json` | everything direct | resources blocked in Russia or unavailable from Russia go through the proxy |

Architecture details: [ARCHITECTURE.md](configs/ARCHITECTURE.md)

## Requirements

- sing-box 1.14.0+ (`http_clients` for downloading rule-sets)
- Administrator rights (for the TUN interface)

## Setup

Copy a template and fill in the placeholders:

```bash
cp configs/desktop.example.json configs/config.json
```

Or, for the "direct by default" scheme:

```bash
cp configs/desktop-direct.example.json configs/config.json
```

Proxies, WireGuard and the Clash API are the same in both templates, so everything below applies to either of them.

**Hysteria2**: three examples, keep the one that matches your server:
- `hy2-fixed-sample`: fixed port, self-signed certificate (`insecure: true`)
- `hy2-hop-sample`: port hopping with `server_ports` + `hop_interval`; the UDP port range must match the server. `server_port` is not set here: sing-box ignores it when `server_ports` is set
- `hy2-tls-sample`: valid TLS certificate (e.g. Let's Encrypt), certificate verification is on; `server_name` must match the domain (or IP) in the certificate

In all three:
- `server`: server address
- `REPLACE_WITH_PASSWORD`: connection password
- `REPLACE_WITH_OBFS_PASSWORD`: salamander obfs password (if the server has no obfs, remove the `obfs` block)

**VLESS**: three examples:
- `vls-vision-sample`: REALITY + Vision (`flow: xtls-rprx-vision`) over TCP; the most common REALITY setup (the default in 3x-ui, for example)
- `vls-sample`: REALITY over gRPC
- `vls-tls-sample`: plain TLS + Vision, for a server with its own domain and certificate

In all three:
- `server`: server address
- `REPLACE_WITH_UUID`: client UUID
- `flow`: must match the server; remove the field if Vision is not enabled there

For REALITY (`vls-vision-sample`, `vls-sample`):
- `REPLACE_WITH_PUBLIC_KEY`: REALITY public key
- `REPLACE_WITH_SHORT_ID`: REALITY short ID
- `server_name`: the third-party site the server masquerades as; take it from the server settings (`serverNames` / `dest`)

**Trojan** (`trojan-tls-sample`):
- `server`, `server_name`: the server domain from the certificate
- `REPLACE_WITH_PASSWORD`: password

If `vls-tls-sample` or `trojan-tls-sample` uses a self-signed certificate, add `"insecure": true` to `tls` and set `server_name` to whatever the certificate is issued for (often the server IP).

**WireGuard** (`wg-sample` and others):
- `address`: client IP in the VPN network; the IPv6 address is only needed if the server is dual-stack, otherwise remove it
- `peers[].address`: WireGuard server IP
- `peers[].port`: WireGuard server port
- `REPLACE_WITH_PRIVATE_KEY`: client private key
- `REPLACE_WITH_PUBLIC_KEY`: server public key
- `REPLACE_WITH_PRE_SHARED_KEY`: pre-shared key (remove the field if the server has none)
- `allowed_ips`: subnets reachable through this tunnel

**Routes**:
- In `route.rules`, replace the `ip_cidr` ranges with the real subnets of your VPN networks
- Add or remove rules for specific processes and domains as needed

**Clash API**:
- `change-me` in `experimental.clash_api.secret`: the API secret

## Check and run

```bash
sing-box check -c configs/config.json
sudo sing-box run -c configs/config.json
```

## HTTP/SOCKS proxy

If you don't need TUN, use the mixed inbound directly:

```
Host: 127.0.0.1
Port: 2080
```

## Switching proxies

The active proxy is selected through the Clash API (`127.0.0.1:9090`) or by changing `default` in the `selector`.

`auto` (`urltest`) checks the latency of every server in its list once a minute and switches to the fastest one; `tolerance` is the difference in ms below which it does not switch. Add new servers to both `selector` and `auto`.

If one server runs several protocols (e.g. Hysteria2, VLESS REALITY and Trojan), add all of them to `auto`: when DPI starts blocking one protocol, `urltest` switches to another.

## Project structure

```
sing-box-config/
├── configs/
│   ├── desktop.example.json         # Template: proxy by default, Russian resources direct
│   ├── desktop-direct.example.json  # Template: direct by default, blocked resources through the proxy
│   ├── config.json                  # Your config with keys (in .gitignore)
│   ├── ARCHITECTURE.md              # Config architecture with diagrams
│   └── ARCHITECTURE.ru.md           # The same in Russian
├── README.md                        # README in English
├── README.ru.md                     # README in Russian
└── LICENSE
```

## License

[MIT](LICENSE)
