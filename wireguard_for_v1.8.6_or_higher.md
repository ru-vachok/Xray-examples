Вот перевод инструкции по настройке WireGuard в Xray-core (v1.8.6+) на русский язык:

---

### Регистрация аккаунта WARP

**Вариант 1: [warp-reg](https://github.com/badafans/warp-reg)**
```bash
curl -sLo warp-reg https://github.com/badafans/warp-reg/releases/download/v1.0/main-linux-amd64 && chmod +x warp-reg && ./warp-reg && rm warp-reg
```

**Вариант 2: [warp-reg.sh](https://github.com/chise0713/warp-reg.sh)**
```bash
bash -c "$(curl -L warp-reg.vercel.app)"
```

**Вариант 3: API zeroteam.top**
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
                        "endpoint": "162.159.192.1:2408"  // Альтернативы: [2606:4700:d0::a29f:c001]:2408 или engage.cloudflareclient.com:2408
                    }
                ],
                "reserved":[0, 0, 0],  // Вставьте ваши reserved-значения
                "mtu": 1280,
                "domainStrategy": "ForceIPv6v4"  // Для IPv4 измените на "ForceIPv4"
            },
            "tag": "warp"  // Основной туннель
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
            "outboundTag": "warp"  // Весь трафик OpenAI через WARP
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
            "destOverride": ["http", "tls", "quic"]  // Анализ трафика
        }
    }
]
```

**3. DNS-настройки:**
```jsonc
"dns": {
    "servers": ["https://1.1.1.1/dns-query"],  // Cloudflare DNS
    "queryStrategy": "UseIP"  // Стратегия: UseIP (A+AAAA), UseIPv4, UseIPv6
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
                "outboundTag": "warp"  // OpenAI → WARP
            },
            {
                "ip": ["geoip:cn"],
                "outboundTag": "warp"  // Весь CN-трафик → WARP
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
            "tag": "block"  // Блокировка
        },
        {
            "protocol": "wireguard",
            "settings": {
                "secretKey": "",
                "address": ["172.16.0.2/32", "2606:4700::/128"],
                "peers": [{"publicKey": "bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=", "allowedIPs": ["0.0.0.0/0", "::/0"], "endpoint": "162.159.192.1:2408"}],
                "reserved":[0, 0, 0],
                "mtu": 1280,
                "domainStrategy": "ForceIPv6v4"  // Стратегия IPv6
            },
            "tag": "warp"
        }
    ]
}
```

---

### Стратегии domainStrategy

| Параметр        | [test-ipv6.com](https://test-ipv6.com/) | [bgp.he.net](https://bgp.he.net/) | [chat.openai.com](https://chat.openai.com/cdn-cgi/trace) |
|-----------------|:---------------------------------------:|:---------------------------------:|:-------------------------------------------------------:|
| **ForceIPv6v4** | IPv6v4 адрес                            | IPv6 адрес                        | IPv6 адрес                                             |
| **ForceIPv6**   | Сайт недоступен                         | IPv6 адрес                        | IPv6 адрес                                             |
| **ForceIPv4v6** | IPv6v4 адрес¹                           | IPv4 адрес                        | IPv4 адрес                                             |
| **ForceIPv4**   | IPv4 адрес                              | IPv4 адрес                        | IPv4 адрес                                             |
| **ForceIP**     | IPv6v4 адрес²                           | IPv6 адрес                        | IPv6 адрес                                             |

¹ Может показывать предупреждение: "У вас есть IPv6 адрес, но браузер его не использует"  
² Иногда показывает предупреждение: "У вас есть IPv6 адрес, но браузер его не использует"

---

### Применение конфигурации
1. Отредактируйте `/usr/local/etc/xray/config.json`
2. Проверьте JSON-формат (рекомендуется [JSONLint](https://jsonlint.com/))
3. Перезапустите сервис:  
   `systemctl restart xray`
4. Проверьте работу:  
   Посетите [https://chat.openai.com/cdn-cgi/trace](https://chat.openai.com/cdn-cgi/trace)  
   В ответе должен отображаться IP Cloudflare

---

<details><summary>Локальная gemma3</summary>
<p>

Общая цель: Этот гайд объясняет, как настроить Xray для использования VPN-сервиса Warp. Warp – это премиальный VPN-сервис, ориентированный на высокую скорость соединения.

Основные компоненты:

Warp: Сам VPN-сервис.
Xray: Прокси-сервер, который управляет соединениями и направляет трафик через Warp VPN.
warp-reg: Скрипт для регистрации вашего аккаунта Warp.
config.json: Файл конфигурации для Xray.
Шаги:

Регистрация вашего аккаунта Warp:

curl -sLo warp-reg https://github.com/badafans/warp-reg/releases/download/v1.0/main-linux-amd64 && chmod +x warp-reg && ./warp-reg && rm warp-reg:
Этот код загружает, устанавливает и затем удаляет скрипт warp-reg. Это удобный способ быстро зарегистрировать ваш аккаунт Warp. Следуйте подсказкам во время регистрации.
Альтернативная регистрация (использование warp-reg.sh)

bash -c "$(curl -L warp-reg.vercel.app)": Этот метод использует URL, предоставленный chise0713, для запуска скрипта, который регистрирует ваш аккаунт Warp. Это альтернативный способ добиться того же, что и скрипт warp-reg.
Получение данных Warp:

curl -sL "https://api.zeroteam.top/warp?format=sing-box" | grep -Eo --color=never '"2606:4700:[0-9a-f:]+/128"|"private_key":"[0-9a-zA-Z\/+]+="|"reserved":\[[0-9]+(,[0-9]+){2}\]':
Этот код использует сторонний API (api.zeroteam.top) для получения ключевой информации Warp:
IPv6 Адрес (2606:4700::): IPv6 адрес Warp VPN сервера. Вам нужно заменить этот заполнитель на фактический IPv6 адрес, предоставленный API.
private_key: Ваш приватный ключ Warp аккаунта. Вам нужно заменить этот заполнитель на фактическое значение, предоставленное API.
reserved: Список чисел, который нужно вставить в конфигурацию.
Настройка Xray (config.json)

"outbounds" Раздел: Определяет, как Xray отправляет и получает трафик.
warp-IPv4: Для Warp's IPv4 соединения.
warp-IPv6: Для Warp's IPv6 соединения.
wireguard: Использует протокол WireGuard с защищенным соединением к Warp VPN.
"inbounds" Раздел: Контролирует входящие соединения в Xray.
sniffing: Включает "обнюбывание", которое помогает Xray определять и направлять HTTP, TLS и QUIC трафик через Warp.
"routing" Раздел: Определяет, как Xray маршрутизирует трафик на основе доменных имен и IP-адресов.
geosite:openai: Маршрутизирует трафик к OpenAI сервисам через Warp.
geoip:cn: Маршрутизирует трафик в Китай через Warp (если Warp имеет серверы в Китае).
"dns" Раздел: Указывает DNS-серверы, используемые Xray.
UseIP: Заставляет Xray использовать как A, так и AAAA записи (IPv4 и IPv6) для DNS-запросов.
UseIP используется для DNS запросов.
Настройка сервера (пример):

JSON, представленный в конце, является образцом конфигурации для Warp сервера. Вам нужно скопировать и вставить его содержимое в ваш файл конфигурации Xray.
Важные заметки и устранение неполадок:

Замените заполнители: Всегда заменяйте заполнители (IPv6 адрес, приватный ключ, reserved) на фактические значения, предоставленные API.
Правильное форматирование JSON: JSON очень чувствителен к форматированию. Используйте инструмент валидации JSON, чтобы убедиться, что ваш файл config.json правильно отформатирован. Неверный JSON приведет к сбою Xray.
Перезапустите Xray: После внесения изменений в config.json, всегда перезапускайте службу Xray с помощью systemctl restart xray.
Подтверждение: После настройки все проведите. Указанная команда для проверки - chat.openai.com/cdn-cgi/trace. Если отслеживание инициировано через Warp, это подтверждает, что ваш трафик идет через VPN.
Брандмауэр: Убедитесь, что ваш брандмауэр позволяет необходимые порты для Xray и Warp VPN.
Этот исчерпывающий гайд должен позволить вам успешно настроить Warp с помощью Xray и наслаждаться безопасным и быстрым соединением в интернете. Всегда адаптируйте конфигурацию к вашим потребностям и при этом уделяйте особое внимание правильному форматированию JSON.

</p>
</details> 

https://ollama.com/library/gemma3:4b
