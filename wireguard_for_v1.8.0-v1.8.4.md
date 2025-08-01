Вот перевод инструкции по настройке WireGuard в Xray-core на русский язык:

---

### Регистрация аккаунта WARP

**Вариант 1: Используйте [warp-reg](https://github.com/badafans/warp-reg)**
```bash
curl -sLo warp-reg https://github.com/badafans/warp-reg/releases/download/v1.0/main-linux-amd64 && chmod +x warp-reg && ./warp-reg && rm warp-reg
```

**Вариант 2: Используйте [warp-reg.sh](https://github.com/chise0713/warp-reg.sh)**
```bash
bash -c "$(curl -L warp-reg.vercel.app)"
```

**Вариант 3: Используйте API zeroteam.top**
```bash
curl -sL "https://api.zeroteam.top/warp?format=sing-box" | grep -Eo --color=never '"2606:4700:[0-9a-f:]+/128"|"private_key":"[0-9a-zA-Z\/+]+="|"reserved":\[[0-9]+(,[0-9]+){2}\]'
```
После выполнения:
1. Скопируйте IPv6-адрес (замените `2606:4700::` в конфиге)
2. Скопируйте значение `private_key` в `secretKey`
3. Скопируйте значение `reserved` в соответствующий массив

---

### Конфигурация "outbounds"

```jsonc
{
    "outbounds": [
        {
            "protocol": "freedom",
            "settings": {
                "domainStrategy": "UseIPv4"
            },
            "proxySettings": {
                "tag": "warp"
            },
            "tag": "warp-IPv4"  // Выход через IPv4
        },
        {
            "protocol": "freedom",
            "settings": {
                "domainStrategy": "UseIPv6"
            },
            "proxySettings": {
                "tag": "warp"
            },
            "tag": "warp-IPv6"  // Выход через IPv6
        },
        {
            "protocol": "wireguard",
            "settings": {
                "secretKey": "",       // Вставьте ваш private_key
                "address": [
                    "172.16.0.2/32",
                    "2606:4700::/128"  // Вставьте ваш IPv6 с /128
                ],
                "peers": [
                    {
                        "publicKey": "bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=",
                        "allowedIPs": ["0.0.0.0/0", "::/0"],
                        "endpoint": "162.159.192.1:2408"  // Альтернатива: [2606:4700:d0::a29f:c001]:2408 или engage.cloudflareclient.com:2408
                    }
                ],
                "reserved":[0, 0, 0],  // Вставьте ваши reserved-значения
                "mtu": 1280
            },
            "tag": "warp"  // Основной WireGuard-выход
        }
    ]
}
```

---

### Дополнительные настройки

**1. Маршрутизация ("routing"):**
```jsonc
"routing": {
    "rules": [
        {
            "domain": ["geosite:openai"],
            "outboundTag": "warp-IPv4"  // Для IPv6 измените на "warp-IPv6"
        }
    ]
}
```

**2. Входящие соединения ("inbounds"):**
```jsonc
"inbounds": [
    {
        "sniffing": {
            "enabled": true,
            "destOverride": ["http", "tls", "quic"]
        }
    }
]
```

**3. DNS-настройки:**
```jsonc
"dns": {
    "servers": ["https://1.1.1.1/dns-query"],
    "queryStrategy": "UseIP"  // По умолчанию: UseIP (A+AAAA). Варианты: UseIPv4/UseIPv6
}
```

---

### Полный пример конфигурации сервера

```jsonc
{
    "log": {"loglevel": "warning"},
    "dns": {
        "servers": ["https://1.1.1.1/dns-query"],
        "queryStrategy": "UseIP"
    },
    "routing": {
        "domainStrategy": "IPIfNonMatch",
        "rules": [
            {
                "domain": ["geosite:openai"],
                "outboundTag": "warp-IPv4"  // Маршрутизация для OpenAI
            },
            {
                "ip": ["geoip:cn"],
                "outboundTag": "warp"  // Весь китайский трафик через WARP
            }
        ]
    },
    "inbounds": [
        {
            // Ваши настройки inbounds
            "sniffing": {
                "enabled": true,
                "destOverride": ["http", "tls", "quic"]
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom",
            "settings": {"domainStrategy": "UseIP"},
            "tag": "direct"  // Прямое соединение
        },
        {
            "protocol": "blackhole",
            "tag": "block"  // Блокировка трафика
        },
        // ... остальные outbounds из предыдущего раздела ...
    ]
}
```

---

### Проверка работы
1. Отредактируйте `/usr/local/etc/xray/config.json`
2. Перезапустите сервис: `systemctl restart xray`
3. Проверьте IP: посетите [https://chat.openai.com/cdn-cgi/trace](https://chat.openai.com/cdn-cgi/trace)
4. В ответе должен отображаться IP Cloudflare

> **Важно:** Всегда проверяйте JSON-формат перед сохранением! Рекомендуется использовать [JSONLint](https://jsonlint.com/) для валидации.
