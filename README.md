# cheburbox

Конфигурация [sing-box](https://github.com/SagerNet/sing-box) с раздельной маршрутизацией трафика: российские ресурсы идут напрямую, все остальные — через выбранный прокси, корпоративные VPN работают всегда независимо от активного прокси.

Подробнее об архитектуре: [ARCHITECTURE.md](configs/desktop/ARCHITECTURE.md)

## Требования

- sing-box 1.11.0+
- Права администратора (для TUN-интерфейса)

## Настройка

Скопируй пример конфига и заполни плейсхолдеры:

```bash
cp configs/desktop/desktop.example.json configs/desktop/config.json
```

**Hysteria2** (`hy2-sample`, `hy2-sample2`):
- `server` — IP-адрес сервера
- `REPLACE_WITH_PASSWORD` — пароль подключения
- `REPLACE_WITH_OBFS_PASSWORD` — пароль obfs salamander

**VLESS** (`vls-sample`):
- `server` — IP-адрес сервера
- `REPLACE_WITH_UUID` — UUID клиента
- `REPLACE_WITH_PUBLIC_KEY` — публичный ключ REALITY
- `REPLACE_WITH_SHORT_ID` — short ID REALITY

**WireGuard** (`wg-sample` и другие):
- `address` — IP-адрес клиента в сети VPN
- `peers[].address` — IP-адрес сервера WireGuard
- `peers[].port` — порт сервера WireGuard
- `REPLACE_WITH_PRIVATE_KEY` — приватный ключ клиента
- `REPLACE_WITH_PUBLIC_KEY` — публичный ключ сервера
- `REPLACE_WITH_PRE_SHARED_KEY` — pre-shared key (если настроен на сервере, иначе убери поле)
- `allowed_ips` — подсети, доступные через этот туннель

**Маршруты**:
- В `route.rules` замени IP-диапазоны `ip_cidr` на реальные подсети твоих VPN-сетей
- При необходимости добавь или убери правила для конкретных процессов и доменов

**Clash API:**
- `change-me` в `experimental.clash_api.secret` — секрет для доступа к API

## Проверка и запуск

```bash
sing-box check -c configs/desktop/config.json
sudo sing-box run -c configs/desktop/config.json
```

## HTTP/SOCKS прокси

Если TUN не нужен, можно использовать mixed-inbound напрямую:

```
Host: 127.0.0.1
Port: 2080
```

## Переключение прокси

Активный прокси выбирается через Clash API (`127.0.0.1:9090`) или изменением поля `default` в секции `selector`.

## Структура проекта

```
cheburbox/
├── configs/
│   └── desktop/
│       ├── desktop.example.json  # Шаблон конфига с плейсхолдерами
│       ├── config.json           # Рабочий конфиг с ключами (в .gitignore)
│       └── ARCHITECTURE.md       # Архитектура конфига со схемой
└── README.md
```
