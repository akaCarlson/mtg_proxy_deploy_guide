### Общее описание

Mtg -это сторонняя реализация MTProto proxy на Go. Авторская идея проекта: прокси должен быть **не публичной точкой на тысячи случайных пользователей**, а **маленьким приватным прокси для узкого круга**, который должен “летать ниже радара”, быстро переезжать на новый IP и хорошо маскироваться. 

Отличия от официального MTProxy

Главная фишка — **упор на anti-censorship и маскировку**:

- **FakeTLS only**: в текущей конфигурации mtg поддерживает только FakeTLS.
- **Domain fronting** как базовая идея проекта.
- **Replay attack protection** заявлен как одна из ключевых особенностей.
- **Proxy chaining через SOCKS5**: mtg можно использовать как фронт, а дальше уводить трафик через другой cloaked transport. Это одна из самых сильных практических фич.
- **IP blocklist / allowlist** встроены в конфиг.

### Самая сильная часть mtg: маскировка

У mtg сейчас есть не только FakeTLS, но и свежая функция **doppelganger**.  
Смысл такой: mtg пытается **статистически походить на реальный сайт**, а не просто “говорить через TLS”. Он:

- добавляет **искусственные задержки** TLS-пакетов,
- **реструктурирует** TLS-пакеты,
- пытается имитировать паттерны нормального веб-трафика,
- поддерживает режим **DRS (Dynamic Record Sizing)** для более правдоподобного поведения TLS.

У mtg **нет user management**. Этот проект — минималистичный прокси, у него **нет управления пользователями**, и это осознанное решение. Также у него **нет management WebUI**. И более того, автор отдельно подчеркивает, что он придерживается модели **single secret** и считает multiple secrets лишним усложнением.

## Подготовка
Тебе нужно создать виртуальный сервер Ubuntu 24.x на любом из облачных провайдеров за пределами РФ. Достаточный конфиг на 10-20 пользователей: 2 Гб ОЗУ, 40 Гб HDD, 1 vCPU.

К серверу подключаешься со своего компьютера через `ssh root@<server_ip>`.
Далее - по инструкции

## 1. Подготовить доступ к VPS только по SSH-ключу

На своем компьютере проверь, есть ли ключ:

```bash
ls -la ~/.ssh
```

Если ключа нет, создай:

```bash
ssh-keygen -t ed25519 -a 100
```

Посмотри публичный ключ:

```bash
cat ~/.ssh/id_ed25519.pub
```

Скопируй его целиком. Затем на VPS:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Вставь туда публичный ключ одной строкой. После этого **не закрывая текущую SSH-сессию**, открой новую и проверь вход по ключу:

```bash
ssh root@IP_СЕРВЕРА
```

Если вход по ключу работает, отключи пароль в SSH. Открой конфиг:

```bash
nano /etc/ssh/sshd_config
```

Проверь, чтобы были такие параметры:

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
UsePAM yes
PermitRootLogin prohibit-password
```

Проверь синтаксис и перезапусти SSH. На Ubuntu/Debian сервис обычно называется `ssh`, а не `sshd`:

```bash
sshd -t
systemctl restart ssh
systemctl status ssh --no-pager
```

## 2. Обновить VPS и установить Docker

```bash
apt update && apt -y upgrade
apt -y install docker.io
systemctl enable --now docker
docker --version
```

## 3. Создать каталог под mtg

```bash
mkdir -p /opt/mtg
cd /opt/mtg
```

## 4. Скачать образ mtg

Лучше использовать стабильный тег `nineseconds/mtg:2`, а не `latest`. Официальный README как раз показывает запуск образа `nineseconds/mtg:2`. ([GitHub](https://github.com/9seconds/mtg "GitHub - 9seconds/mtg: Highly opinionated MTPROTO proxy for Telegram · GitHub"))

```bash
docker pull nineseconds/mtg:2
```

## 5. Сгенерировать секрет

`mtg` умеет генерировать секрет сам. Для Docker официальный способ такой:

```bash
docker run --rm nineseconds/mtg:2 generate-secret --hex DOMAIN
```

Это подтверждено и в обсуждении проекта, и в общем описании запуска. ([GitHub](https://github.com/9seconds/mtg/discussions/250?utm_source=chatgpt.com "How to obtain a secret when using docker? · 9seconds mtg"))

Пример:

```bash
docker run --rm nineseconds/mtg:2 generate-secret --hex auriga.com
```

Ты получишь строку вида:

```text
ee....................................
```

### Какой домен подставлять в `generate-secret`

Здесь важен не “самый известный незаблокированный домен”, а **правдоподобная маскировка**. В актуальном `BEST_PRACTICES.md` автор прямо пишет, что в 2026 уже недостаточно притворяться условным `microsoft.com` на VPS в чужом ASN: DNS/SNI/IP должны выглядеть согласованно, иначе такой трафик проще пометить как подозрительный. Лучший вариант — свой домен, указывающий на твой VPS, и веб-сайт перед mtg. ([GitHub](https://github.com/9seconds/mtg/blob/master/BEST_PRACTICES.md "mtg/BEST_PRACTICES.md at master · 9seconds/mtg · GitHub"))

Для простого старта можно использовать нейтральный домен, но не стоит бездумно подставлять крупные внешние сайты типа `gosuslugi.ru` или `microsoft.com`, если твой IP к ним явно не относится. Автор отдельно предупреждает не маскироваться под `ok.ru` только потому, что у mtg есть хорошие предсобранные профили TLS-задержек: это не значит, что надо выдавать VPS-провайдерский IP за реальный IP этого сервиса. ([GitHub](https://github.com/9seconds/mtg "GitHub - 9seconds/mtg: Highly opinionated MTPROTO proxy for Telegram · GitHub"))

## 6. Создать минимальный конфиг

В минимальном варианте нужны только `secret` и `bind-to`. Официальный README прямо говорит, что этого достаточно, а остальные параметры уже имеют sensible defaults. В example config также указано, что `mtg` поддерживает только FakeTLS, а секрет должен быть base64 или начинаться с `ee`. ([GitHub](https://github.com/9seconds/mtg "GitHub - 9seconds/mtg: Highly opinionated MTPROTO proxy for Telegram · GitHub"))

Создай `/opt/mtg/config.toml`:

```bash
cat > /opt/mtg/config.toml <<'EOF'
secret = "ВСТАВЬ_СЮДА_СВОЙ_SECRET"
bind-to = "0.0.0.0:3128"
concurrency = 8192
EOF
```

Почему `3128`, а не `443`: внутри контейнера можно слушать `3128`, а наружу пробросить `443`. Официальный Docker-пример именно это и показывает: `-p 443:3128`, где `443` — внешний порт, а `3128` — порт из `bind-to`. ([GitHub](https://github.com/9seconds/mtg "GitHub - 9seconds/mtg: Highly opinionated MTPROTO proxy for Telegram · GitHub"))

## 7. Запустить контейнер

```bash
docker run -d \
  --name mtg-proxy \
  --restart unless-stopped \
  -v /opt/mtg/config.toml:/config.toml:ro \
  -p 443:3128 \
  nineseconds/mtg:2
```

Это соответствует официальному способу запуска Docker-образа. ([GitHub](https://github.com/9seconds/mtg "GitHub - 9seconds/mtg: Highly opinionated MTPROTO proxy for Telegram · GitHub"))

## 8. Проверить, что mtg поднялся

```bash
docker ps
docker logs --tail 100 mtg-proxy
docker port mtg-proxy
ss -lntp | grep :443
```

Ожидаемо:

- контейнер запущен;
    
- `docker port mtg-proxy` покажет что-то вроде `3128/tcp -> 0.0.0.0:443`;
    
- в логах нет фатальных ошибок. Формат `443:3128` важен: это и есть внешний и внутренний порт соответственно. ([GitHub](https://github.com/9seconds/mtg "GitHub - 9seconds/mtg: Highly opinionated MTPROTO proxy for Telegram · GitHub"))
    

## 9. Открыть порт 443 в firewall

Если используешь `ufw`:

```bash
ufw allow 443/tcp
ufw reload
ufw status
```

Если у VPS-провайдера есть отдельный cloud firewall / security group, там тоже должен быть разрешен **TCP 443**.

Проверка снаружи:

```bash
nc -vz IP_СЕРВЕРА 443
```

## 10. Сгенерировать правильную ссылку для Telegram


```bash
docker exec mtg-proxy /mtg access -p 443 -i ВНЕШНИЙ_IP_СЕРВЕРА /config.toml
```

## 11. Проверить подключение из Telegram

Используй сгенерированную `tg://proxy?...` или `https://t.me/proxy?...` ссылку. В ней должны быть:

- `server=IP_СЕРВЕРА`
    
- `port=443`
    
- `secret=...`
    
## 12. Переключиться с минимального конфига на рабочий

`mtg` сам пишет, что большинство настроек имеют разумные значения по умолчанию и не нужно без нужды перечислять всё подряд. В свежих релизах также отмечено развитие doppelganger и то, что `DRS` сделан опциональным. README отдельно описывает doppelganger: можно дать 2–3 HTTPS URL домена, под который ты хочешь статистически подстраивать TLS-поведение. ([GitHub](https://github.com/9seconds/mtg/blob/master/example.config.toml "mtg/example.config.toml at master · 9seconds/mtg · GitHub"))

Я бы рекомендовал такой практичный конфиг:

```toml
secret = "ТВОЙ_SECRET"
bind-to = "0.0.0.0:3128"
concurrency = 8192

prefer-ip = "prefer-ipv4"
auto-update = false
tolerate-time-skewness = "5s"
allow-fallback-on-unknown-dc = true

[domain-fronting]
port = 443
# ip = "X.X.X.X"   # указывай только если DNS fronting-хоста реально режут

[network.timeout]
tcp = "5s"
http = "10s"
idle = "1m"

[defense.doppelganger]
urls = [
  "https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js",
  "https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css",
  "https://ajax.googleapis.com/ajax/libs/jquery/3.7.1/jquery.min.js"
]
repeats-per-raid = 6
raid-each = "6h"
drs = false

[defense.anti-replay]
enabled = true
max-size = "1mib"
error-rate = 0.001

[defense.blocklist]
enabled = true
download-concurrency = 2
urls = [
  "https://iplists.firehol.org/files/firehol_level1.netset"
]
update-each = "24h"

[defense.allowlist]
enabled = false

[stats.prometheus]
enabled = true
bind-to = "127.0.0.1:3129"
http-path = "/metrics"
metric-prefix = "mtg"
```

Почему так:

- `auto-update = false` соответствует example config и обычно безопаснее как базовый режим. ([GitHub](https://github.com/9seconds/mtg/blob/master/example.config.toml "mtg/example.config.toml at master · 9seconds/mtg · GitHub"))
    
- `allow-fallback-on-unknown-dc = true` помогает не ронять соединение на неожиданных DC-запросах; это описано в example config как полезный fallback для живучести. ([GitHub](https://github.com/9seconds/mtg/blob/master/example.config.toml "mtg/example.config.toml at master · 9seconds/mtg · GitHub"))
    
- doppelganger стоит настраивать 2–3 HTTPS URL того домена, под который ты реально маскируешься; README прямо советует брать 2–3 URL и не переусложнять. ([GitHub](https://github.com/9seconds/mtg "GitHub - 9seconds/mtg: Highly opinionated MTPROTO proxy for Telegram · GitHub"))
    
- `drs = false` — нормально, потому что в свежих релизах его сделали опциональным, а в example config по умолчанию он не является обязательным must-have. ([GitHub](https://github.com/9seconds/mtg/releases?utm_source=chatgpt.com "Releases · 9seconds/mtg"))
    
- `domain-fronting.ip` лучше не задавать, пока реально не упираешься в блокировку DNS fronting-hostname; example config прямо описывает этот параметр как обход blocked DNS resolution. ([GitHub](https://github.com/9seconds/mtg/blob/master/example.config.toml "mtg/example.config.toml at master · 9seconds/mtg · GitHub"))
    

После изменения конфига:

```bash
docker restart mtg-proxy
docker logs --tail 100 mtg-proxy
```

## 13. Обновление mtg

```bash
docker pull nineseconds/mtg:2
docker stop mtg-proxy
docker rm mtg-proxy

docker run -d \
  --name mtg-proxy \
  --restart unless-stopped \
  -v /opt/mtg/config.toml:/config.toml:ro \
  -p 443:3128 \
  nineseconds/mtg:2
```

Потом снова при необходимости:

```bash
docker exec mtg-proxy /mtg access -p 443 -i ВНЕШНИЙ_IP /config.toml
```

## 14. Что реально повышает устойчивость к блокировкам

По актуальному `BEST_PRACTICES.md`, сегодня уже недостаточно просто притвориться известным сайтом. Более сильная схема выглядит так: свой домен, свой DNS на VPS, простой сайт на этом домене, TLS-сертификат, `mtg` перед этим веб-сервером, а при необходимости — даже SOCKS5/VPNized uplink за ним. То есть лучшая практика — сделать связку **DNS + SNI + IP правдоподобной**, а не просто выбрать “громкий” домен. ([GitHub](https://github.com/9seconds/mtg/blob/master/BEST_PRACTICES.md "mtg/BEST_PRACTICES.md at master · 9seconds/mtg · GitHub"))

Практически я бы разделил так:

- **Быстрый рабочий старт**: Docker, `443:3128`, один secret, базовый doppelganger.
    
- **Более живучая схема**: свой домен, A-запись на VPS, простой сайт, Let’s Encrypt, doppelganger URL с этого же домена.
    
- **Максимум живучести**: собственный домен + правдоподобный сайт + при необходимости uplink через SOCKS5/VPN, как советует автор best practices. ([GitHub](https://github.com/9seconds/mtg/blob/master/BEST_PRACTICES.md "mtg/BEST_PRACTICES.md at master · 9seconds/mtg · GitHub"))
    
## 16. Минимальный набор команд одной пачкой

```bash
apt update && apt -y upgrade
apt -y install docker.io
systemctl enable --now docker

mkdir -p /opt/mtg
cd /opt/mtg

docker pull nineseconds/mtg:2

docker run --rm nineseconds/mtg:2 generate-secret --hex auriga.com
```

Создай `/opt/mtg/config.toml`:

```toml
secret = "ТВОЙ_SECRET"
bind-to = "0.0.0.0:3128"
concurrency = 8192
```

Запусти:

```bash
docker run -d \
  --name mtg-proxy \
  --restart unless-stopped \
  -v /opt/mtg/config.toml:/config.toml:ro \
  -p 443:3128 \
  nineseconds/mtg:2
```

Открой порт:

```bash
ufw allow 443/tcp
ufw reload
```

Сгенерируй правильную ссылку:

```bash
docker exec mtg-proxy /mtg access -p 443 -i ВНЕШНИЙ_IP_СЕРВЕРА /config.toml
```

Если хочешь, я следующим сообщением могу оформить эту инструкцию еще и в виде **короткого чеклиста-командника без пояснений**, чтобы можно было просто держать под рукой.
