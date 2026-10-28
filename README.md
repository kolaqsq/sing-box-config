# cheburbox

Конфигурация [sing-box](https://github.com/SagerNet/sing-box) с раздельной маршрутизацией трафика. Корпоративные VPN работают всегда, независимо от активного прокси.

Две схемы на выбор:

| Шаблон | По умолчанию | Исключения |
|--------|--------------|------------|
| `desktop.example.json` | всё через прокси | российские ресурсы — напрямую |
| `desktop-direct.example.json` | всё напрямую | заблокированные в России и недоступные из России ресурсы — через прокси |

Подробнее об архитектуре: [ARCHITECTURE.md](configs/ARCHITECTURE.md)

## Требования

- sing-box 1.14.0+ (`http_clients` для загрузки rule-set)
- Права администратора (для TUN-интерфейса)

## Настройка

Скопируй пример конфига и заполни плейсхолдеры:

```bash
cp configs/desktop.example.json configs/config.json
```

Или, для схемы «по умолчанию напрямую»:

```bash
cp configs/desktop-direct.example.json configs/config.json
```

Прокси, WireGuard и Clash API в обоих шаблонах одинаковые, поэтому описание ниже подходит для любого из них.

**Hysteria2** — три примера, оставь тот, что подходит под твой сервер:
- `hy2-fixed-sample` — фиксированный порт, самоподписанный сертификат (`insecure: true`)
- `hy2-hop-sample` — порт-хоппинг: `server_ports` + `hop_interval`, диапазон UDP-портов должен совпадать с настройкой сервера; `server_port` здесь не указывается — при заданном `server_ports` sing-box его игнорирует
- `hy2-tls-sample` — валидный TLS-сертификат (например, Let's Encrypt): проверка сертификата включена, `server_name` должен совпадать с доменом (или IP) из сертификата

Во всех трёх:
- `server` — адрес сервера
- `REPLACE_WITH_PASSWORD` — пароль подключения
- `REPLACE_WITH_OBFS_PASSWORD` — пароль obfs salamander (если obfs на сервере не настроен, убери блок `obfs`)

**VLESS** (`vls-sample`):
- `server` — IP-адрес сервера
- `REPLACE_WITH_UUID` — UUID клиента
- `REPLACE_WITH_PUBLIC_KEY` — публичный ключ REALITY
- `REPLACE_WITH_SHORT_ID` — short ID REALITY

**WireGuard** (`wg-sample` и другие):
- `address` — IP-адрес клиента в сети VPN; IPv6-адрес нужен только если сервер выдаёт dual-stack, иначе убери его
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
sing-box check -c configs/config.json
sudo sing-box run -c configs/config.json
```

## HTTP/SOCKS прокси

Если TUN не нужен, можно использовать mixed-inbound напрямую:

```
Host: 127.0.0.1
Port: 2080
```

## Переключение прокси

Активный прокси выбирается через Clash API (`127.0.0.1:9090`) или изменением поля `default` в секции `selector`.

Вариант `auto` (`urltest`) раз в минуту проверяет задержку до каждого сервера из своего списка и сам переключается на самый быстрый; `tolerance` — разница в мс, при которой переключение не происходит. Новые серверы добавляй и в `selector`, и в `auto`.

## Структура проекта

```
cheburbox/
├── configs/
│   ├── desktop.example.json         # Шаблон: по умолчанию прокси, РФ напрямую
│   ├── desktop-direct.example.json  # Шаблон: по умолчанию напрямую, заблокированное через прокси
│   ├── config.json                  # Рабочий конфиг с ключами (в .gitignore)
│   └── ARCHITECTURE.md              # Архитектура конфига со схемой
└── README.md
```
