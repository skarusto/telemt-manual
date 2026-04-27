> [!NOTE]
> В первую очередь я пишу этот мануал для себя, собирая информацию по профильным чатам и тестируя решения методом проб и ошибок. Написанное здесь - не истина в последней инстанции, а лишь мой опыт.

> [!IMPORTANT]
> Все упомянутые в статье инструменты и решения написаны не мной. Соответствующие ссылки на первоисточники указаны.

## О чем пойдет речь?
Мануал расскажет, как поднять MTProxy для Telegram на своём сервере с маскировкой под собственный SNI и блокировкой TCP RST пакетов.

### Используемые инструменты / сервисы
- **Telemt** - быстрый, безопасный и многофункциональный сервер, написанный на Rust: он полностью реализует официальный алгоритм прокси Telegram и добавляет множество улучшений, готовых к использованию в производственной среде.

>Ссылка на репозиторий: https://github.com/telemt/

- **Caddy** - современный и простой в использовании веб-сервер, прокси и менеджер TLS-сертификатов

> Ссылка не репозиторий: https://github.com/caddyserver/caddy/

- **ТСПУ RST Block List** - живой список IP адресов DPI/ТСПУ оборудования, которое шлёт TCP RST пакеты на MTProxy серверы и рвёт соединения пользователей.

> Ссылка не репозиторий: https://github.com/upe4d/tspublock/

## Пререквизиты
- Наличие арендованного VPS с ОС на базе Debian (Ubuntu, Mint, Kali, MX и тд)
> Автор использует Ubuntu 22.04
- Базовые навыки работы с системой и её инструментами через консоль
- Наличие зарегистрированного домена с A-записью, указывающую на IP-адрес вашего VPS
> В том числе вы можете использовать любой доступный бесплатный DNS-сервис

## Часть первая. Установка Caddy

1. Устанавливаем Caddy

```
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

2. Открываем конфигурационный файл Caddy
> В качестве текстового редактора автор использует nano

```
nano /etc/caddy/Caddyfile
```

3. Вписываем конфиг
> [!WARNING]
> Замените "domain.com" на ваш домен

```
domain.com {
    root * /usr/share/caddy
    file_server
    try_files {path} /index.html
}
```

3. Сохраняем файл

```
Ctrl-X + Y + Enter
```

5. Перезапускаем Caddy

```
systemctl restart caddy
```
5. Идем в браузер и стучимся в свой домен. Вы должны увидеть дефолтную заглушку сервера Caddy.
6. Возвращаемся в консоль и снова идём редактировать конфиг Caddy:

```
nano /etc/caddy/Caddyfile
```

Новый конфиг должен выглядеть так:
> [!WARNING]
> Замените "domain.com" на ваш домен

```
domain.com:80 {
        redir https://domain.com{uri} permanent
}

domain.com:8443 {
        bind 127.0.0.1
        root * /usr/share/caddy
        file_server
        try_files {path} /index.html
}
```
7. Сохраняем файл
8. Файлы сайта расположены в папке ```/usr/share/caddy/```. Сейчас там лежит только ```index.html``` со стандартной заглушкой Caddy, которую следует заменить.
  
> Вы можете сгенерировать любой нейросетью нужную вам страничку либо воспользоваться [готовыми вариантами](https://github.com/Famebloody/SNI-Templates)

## Часть вторая. Блокируем RST пакеты

1. Устанавливаем iptables и iptables-persistent

```
apt install ipset iptables-persistent
```

2. Применяем блок-лист CyberOK 

```
curl -fsSL "https://stats.gptru.pro:4443/rst/api.php?action=export&fmt=iptables&src=cyberok" \
  -o /tmp/tspublock.sh && bash /tmp/tspublock.sh
```

3. Сохраняем правила

```
ipset save > /etc/ipset.conf && iptables-save > /etc/iptables/rules.v4
```

4. Включаем автообновление списков IP раз в сутки

```
(crontab -l 2>/dev/null; echo "0 3 * * * curl -fsSL https://stats.gptru.pro:4443/rst/update_cyberok.sh | bash >> /var/log/cyberok.log 2>&1") | crontab -
```

> [!NOTE]
> Для проверки количества заблокированных пакетов используется команда ```iptables -L TSPUBLOCK -v -n```.
> 
> Нас интересует колонка ```pkts```

  
## Часть третья. Установка и конфигурация Telemt

1. Устанавливаем Telemt:

```
curl -fsSL https://raw.githubusercontent.com/telemt/telemt/main/install.sh | sh
```

2. Открываем конфигурационный файл и вносим изменения. В конечном счете он должен выглядеть как-то так:

> [!WARNING]
> Замените "domain.com" на ваш домен
>
> Не меняйте **своё** значение параметра ```hello``` из блока ```[access.users]```

```           
# === General Settings ===
[general]
# ad_tag = "00000000000000000000000000000000"
use_middle_proxy = false

[general.modes]
classic = false
secure = false
tls = true

[general.links]
public_host = "domain.com"

[server]
port = 443

[server.api]
enabled = true
listen = "0.0.0.0:9091"
whitelist = ["127.0.0.0/8"]
minimal_runtime_enabled = false
minimal_runtime_cache_ttl_ms = 1000
# read_only = true

# === Anti-Censorship & Masking ===
[censorship]
tls_domain = "domain.com"
mask = true
mask_host = "127.0.0.1"
mask_port = 8443
tls_emulation = true

[access.users]
# format: "username" = "32_hex_chars_secret"
hello = "32_hex_chars_secret"
```
3. Перезапускаем Telemt

```
systemctl restart telemt
```
4. Получаем ссылку для Telegram

```
curl -s http://127.0.0.1:9091/v1/users | jq -r '.data[] | "[\(.username)]", (.links.classic[]? | "classic: \(.)"), (.links.secure[]? | "secure: \(.)"), (.links.tls[]? | "tls: \(.)"), ""'
```

> [!NOTE]
> Для просмотра логов Telemt используйте команду
>
> ```
> journalctl -u telemt -f
> ```
> 
## Обновление Telemt

```
systemctl stop telemt
wget -qO- "https://github.com/telemt/telemt/releases/latest/download/telemt-$(uname -m)-linux-$(ldd --version 2>&1 | grep -iq musl && echo musl || echo gnu).tar.gz" | tar -xz
mv telemt /bin
systemctl start telemt
```
