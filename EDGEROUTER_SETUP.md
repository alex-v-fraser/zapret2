# zapret2 на EdgeRouter ER-POE5 — обход блокировки YouTube (рабочая конфигурация)

Готовый, проверенный вживую сетап обхода DPI для YouTube на роутере **Ubiquiti EdgeRouter
ER-POE5 (EdgeOS)**. Работает на всех клиентах: ПК-браузер, Android TV, iOS Safari **и нативное
iOS-приложение**. Переживает перезагрузку роутера.

---

## TL;DR — что в итоге работает

- **TLS-стратегия (одна на всё):** `tcpseg:pos=0,1:ip_id=rnd:repeats=1` (TLS 1.2 и 1.3).
- **QUIC (UDP/443):** обходу не поддаётся → **отбиваем ICMP-ом** (`REJECT icmp-port-unreachable`),
  клиенты сразу уходят на рабочий TLS.
- **Фильтр:** по hostlist (SNI), домены youtube **обязательно включая API**
  `youtubei.googleapis.com` + `googleapis.com` (иначе iOS-приложение не открывается).
- **Запуск:** из `/config/zapret2` через sysv-init; автозапуск — `/config/scripts/post-config.d/`.
- **DNS:** у провайдера НЕ травится (проверено) → DoH не нужен.
- **Offload:** оставлен включённым — обход работает (вмешиваемся только в первые пакеты).

---

## 1. Параметры этого роутера (подтверждены)

| Параметр | Значение |
|---|---|
| Платформа | Ubiquiti ER-POE5, Cavium Octeon, EdgeOS (Debian + systemd) |
| Ядро / userland | ядро 64-бит (`uname -m`=mips64), userland 32-бит, **big-endian** |
| Firewall | **iptables** (nft нет; `iptables` в `/sbin`, работать из-под root `sudo -i`) |
| WAN | **eth0** (например 93.95.165.x) |
| LAN | **br0** (192.168.1.1/24; в мост сведены eth1–eth4, switch0) |
| IPv6 | на WAN нет → `DISABLE_IPV6=1` |
| DNS | провайдер не травит (лакмус: `rutracker.org` отдаёт реальные Cloudflare IP и локально, и через 8.8.8.8) |

**Персистентность EdgeOS:** при обновлении прошивки `/opt`, `/usr`, часть `/etc` стираются —
переживает только `/config`. Поэтому проект живёт в `/config/zapret2`, автозапуск — в
`/config/scripts/post-config.d/`.

---

## 2. Бинарники (mips64 musl, big-endian)

`nfqws2`/`mdig`/`ip2net` берутся из официального релиза zapret2 и кладутся в `binaries/`.
На роутере из-под root:
```sh
cd /config/zapret2
./install_bin.sh                 # сам выберет рабочий вариант перебором
./nfq2/nfqws2 --version          # должен вывести версию без Illegal instruction
```
Правильная арх. — **mips64 musl big-endian** (статический 64-бит бинарник идёт нативно на 64-бит
ядре, несмотря на 32-бит userland). Никогда `*el` (little-endian) и не glibc-static (ломает DNS).
Если mips64 не пошёл — fallback на 32-бит `mips` (BE) musl.

---

## 3. curl с TLS1.3/HTTP3 (нужен только для blockcheck/поиска стратегий)

Системный curl на EdgeOS — без HTTP3. Берём статический: `stunnel/static-curl`,
**`curl-linux-mips64-musl-8.20.0.tar.xz`**, кладём в `/tmp` (tmpfs):
```sh
chmod +x /tmp/curl
/tmp/curl -V | grep -iE 'http3|TLSv1.3'   # должны быть HTTP3 и TLS1.3
```
Подтверждено: `curl 8.20.0 (mips64-pc-linux-gnu)`, OpenSSL/4.0.0 (quictls), ngtcp2/nghttp3 —
запускается нативно, умеет HTTP3 и TLS1.3.

---

## 4. Боевой конфиг

### 4.1 `/config/zapret2/config`
Ключевые значения (остальное — из `config.default`):
```sh
NFQWS2_ENABLE=1
MODE_FILTER=hostlist
FWTYPE=iptables
IFACE_WAN=eth0
IFACE_LAN=br0
DISABLE_IPV6=1
INIT_APPLY_FW=1
FILTER_TTL_EXPIRED_ICMP=1

NFQWS2_OPT="
--filter-tcp=443 --filter-l7=tls <HOSTLIST> --out-range=-d10 --payload=tls_client_hello --lua-desync=tcpseg:pos=0,1:ip_id=rnd:repeats=1
"
```
- Один TCP/443-профиль. HTTP(80) и QUIC-профиль в nfqws **не нужны** (QUIC закрываем firewall-ом,
  п. 4.3). `<HOSTLIST>` подставляется из hostlist. Lua-библиотеки init-скрипт подключает сам.

### 4.2 hostlist `/config/zapret2/ipset/zapret-hosts-user.txt`
```
youtube.com
googlevideo.com
ytimg.com
ggpht.com
youtubei.googleapis.com
googleapis.com
```
> ⚠️ `youtubei.googleapis.com` и `googleapis.com` — **критичны для iOS-приложения** (см. §7).

### 4.3 QUIC → ICMP reject
QUIC/UDP-443 заблокирован DPI и не обходится. «Тихий» drop нельзя (iOS-cronet виснет, не
откатываясь на TCP). Отбиваем ICMP-ом:
```sh
iptables -I FORWARD 1 -p udp --dport 443 -j REJECT --reject-with icmp-port-unreachable
```
Это правило не ведёт zapret — вешаем его персистентно (§6).
- Сейчас стоит **глобальный** REJECT (весь HTTP/3 на LAN падает на TCP — для сайтов незаметно).
- Опционально точнее: тот же REJECT с `-m set --match-set <google> dst` по ipset из диапазонов
  Google (`https://www.gstatic.com/ipranges/goog.json`), чтобы чужой QUIC не трогать.

---

## 5. Запуск (из `/config`, без install_easy/без /opt)

`install_easy.sh` отказывается ставить из `/config` (хочет `/opt`, который стирается при апгрейде).
Поэтому запускаем штатный sysv-init прямо из `/config/zapret2` — он сам вычисляет `ZAPRET_BASE`:
```sh
cd /config/zapret2
./init.d/sysv/zapret2 restart        # nfqws2 (qnum 300) + NFQUEUE-правила iptables
pgrep -a nfqws2                       # демон должен быть живой
iptables -t mangle -nvL | grep -i NFQUEUE   # правила qnum 300, счётчики растут
```
Демон поднимается под юзером `tpws` (init создаёт сам); `--lua-init` для `zapret-lib/antidpi/auto`
добавляются автоматически (`init.d/sysv/functions`).

---

## 6. Автозапуск + персистентность (EdgeOS)

Создать `/config/scripts/post-config.d/zapret2.sh`:
```sh
#!/bin/sh
# zapret2 YouTube bypass — autostart on EdgeOS
sleep 10                               # дать EdgeOS поднять firewall/интерфейсы
/config/zapret2/init.d/sysv/zapret2 restart
# QUIC -> ICMP reject (форсим откат на TCP)
iptables -C FORWARD -p udp --dport 443 -j REJECT --reject-with icmp-port-unreachable 2>/dev/null \
  || iptables -I FORWARD 1 -p udp --dport 443 -j REJECT --reject-with icmp-port-unreachable
```
```sh
chmod +x /config/scripts/post-config.d/zapret2.sh
```
После `reboot` проверить: `pgrep -a nfqws2`, наличие NFQUEUE-правил и REJECT, и YouTube на
устройствах. (Проверено: после перезагрузки всё поднимается само.)

> Если будете менять firewall EdgeOS через `configure/commit` — он может перестроить iptables;
> тогда перезапустить `post-config.d/zapret2.sh` или `./init.d/sysv/zapret2 restart`.

---

## 7. РЕЗЕРВНЫЕ СТРАТЕГИИ (важно сохранить)

DPI меняется со временем — рабочая сегодня стратегия завтра может отвалиться. Ниже — **все
стратегии, найденные blockcheck2 на этом провайдере**, с пометкой что они покрывают. Меняется
только строка `--lua-desync=...` в `NFQWS2_OPT` (профиль остаётся
`--filter-tcp=443 --filter-l7=tls <HOSTLIST> --out-range=-d10 --payload=tls_client_hello ...`).

### Основная (развёрнута)
```
--lua-desync=tcpseg:pos=0,1:ip_id=rnd:repeats=1
```
**Единственная**, что пробивает `youtubei.googleapis.com` (API нативного приложения), и при этом
универсальна для страницы/видео/превью, TLS 1.2 и 1.3. Простая, дешёвая по CPU.

### Резерв №1 — wssize + multidisorder
```
--lua-desync=wssize:wsize=1:scale=6 --lua-desync=multidisorder:pos=1,midsld
```
Универсал по доменам (страница, видео-CDN googlevideo, ytimg, ggpht), TLS 1.2 и 1.3,
GET-проверена (переживает «16 KB» на видео). Без fake-пакетов и badsum → дружелюбна к offload/NAT.
**Минус:** НЕ пробивает `youtubei.googleapis.com` → ломает ленту/авторизацию в iOS-приложении.
Годится, если нативное iOS-приложение не нужно (браузер/Android работают).

### Резерв №2 — «белый» SNI (антидот «16 KB» на видео)
```
--lua-desync=fake:blob=fake_default_tls:tcp_seq=1000000:tls_mod=rnd,dupsid,sni=www.google.com:repeats=1 --lua-desync=multidisorder:pos=1,midsld
```
Подменяет SNI на `www.google.com` (белый список) — спасает, если видео начнёт упираться в
16-КБ-обрезку на крупных CDN. Универсал по доменам, TLS 1.2/1.3.

### Резерв №3 — fake + multisplit (из COMMON, без badsum)
```
--lua-desync=fake:blob=fake_default_tls:tcp_seq=1000000:repeats=1 --lua-desync=multisplit:pos=midsld
# вариант с двумя точками реза:
--lua-desync=fake:blob=fake_default_tls:tcp_seq=1000000:repeats=1 --lua-desync=multisplit:pos=1,midsld
```

### Прочие рабочие варианты, замеченные в прогонах (на крайний случай)
- `--in-range=-s1 --lua-desync=oob:urp=midsld` (или `oob:urp=b`) — простой OOB, проходил TLS 1.3.
- Семейство `fake:blob=0x00000000:badsum + fake:blob=fake_default_tls:badsum:tls_mod=rnd,dupsid +
  fakeddisorder:pos=midsld` — **badsum рискован** на offload/NAT (битая контрольная сумма может
  отбрасываться железом), пробовать в последнюю очередь.

### Если одна стратегия перестанет покрывать и API, и видео — мультипрофиль
Разнести по двум профилям (разделитель `--new`): `tcpseg` для API-хостлиста и, например,
`wssize+multidisorder` для видео:
```
NFQWS2_OPT="
--filter-tcp=443 --filter-l7=tls --hostlist=ipset/list-yt-api.txt --out-range=-d10 --payload=tls_client_hello --lua-desync=tcpseg:pos=0,1:ip_id=rnd:repeats=1 --new
--filter-tcp=443 --filter-l7=tls --hostlist=ipset/list-yt-video.txt --out-range=-d10 --payload=tls_client_hello --lua-desync=wssize:wsize=1:scale=6 --lua-desync=multidisorder:pos=1,midsld
"
```

### QUIC
Рабочей стратегии обхода QUIC **не найдено** — даже с готовым «белым» google-блобом
(`files/fake/quic_initial_www_google_com.bin`, `--lua-desync=fake:blob=quic_google:repeats=...`).
UDP/443 режется целиком → решение только одно: **ICMP-reject** (§4.3), клиенты уходят на TLS.

---

## 8. Проверка (на всех типах клиентов!)

1. `pgrep -a nfqws2` — демон жив; правила NFQUEUE qnum 300 на месте, счётчики растут.
2. Голый curl без обхода: `youtubei.googleapis.com` = reset/timeout (блок подтверждён); с обходом
   — открывается.
3. **ПК-браузер** — страница + видео.
4. **Android-приложение** (Smart TV) — работает.
5. **iOS Safari** (m.youtube.com) — работает.
6. **iOS нативное приложение** — авторизация + лента + видео (ключевой критерий: именно оно
   завязано на `youtubei.googleapis.com`).
7. После `reboot` — всё поднимается из `post-config.d` без рук.

> ⚠️ Проверять обязательно на КАЖДОМ типе клиента: у браузера/Android/iOS-приложения разное
> поведение по QUIC, ECH и API. «Safari работает» НЕ значит «приложение работает».

---

## 9. Ключевые уроки этой отладки (чтобы не наступать снова)

- **iOS-приложение vs Safari.** Web (Safari) тянет InnerTube API с `www.youtube.com/youtubei/...`
  (этот хост мы обходим), а **нативное приложение** — с отдельного `youtubei.googleapis.com`,
  который **блокируется DPI** и пробивается только `tcpseg` (не `wssize+multidisorder`). Поэтому
  Safari «работал», а приложение не открывалось. Вывод: `youtubei.googleapis.com`/`googleapis.com`
  должны быть в hostlist, а стратегия — `tcpseg`.
- **QUIC и iOS-cronet.** Нельзя «тихо» дропать QUIC — iOS не откатывается на TCP. Только ICMP
  reject. Android и Safari откатываются сами, iOS — нет.
- **ECH.** ПК-Chrome к Google использует настоящий ECH (SNI зашифрован) → hostlist по SNI на ПК
  может не матчить. Нативные приложения (iOS/Android) шлют GREASE-ECH (SNI видим) → hostlist
  работает. Если понадобится «железобетонно» для ПК — фильтровать по IP-диапазонам Google
  (`MODE_FILTER=ipset`), а не по SNI.
- **«16 KB»-блок.** Бьёт по видео-потоку `*.googlevideo.com` (мегабайты), не по странице.
  blockcheck по умолчанию шлёт HEAD (умещается в 16 КБ → ложный OK) — для видео-CDN гонять с
  `CURL_HTTPS_GET=1`. Антидот — белый SNI `www.google.com` (резерв №2).
- **Offload.** Оставлен включённым и работает (вмешиваемся в первые пакеты, они успевают на
  медленный путь). Если после апдейта прошивки обход перестанет ловиться — отключить:
  `configure; set system offload ipv4 forwarding disable; commit; save`.
- **Мусор от blockcheck.** Прерванные прогоны blockcheck оставляют per-IP NFQUEUE-правила
  (queue 1000–65535 **без `bypass`** → дропают трафик). Чистятся перезагрузкой или
  `iptables -nvL | grep blockcheck`. Прерывать blockcheck `kill <PID>` (НЕ `-9`).

---

## 10. Как заново искать стратегию (если провайдер сменит блокировку)

Кратко (детали — в комментариях к `blockcheck2.d/yt/`):
```sh
cd /config/zapret2
# остановить боевой обход, чтобы не мешал
./init.d/sysv/zapret2 stop
iptables -D FORWARD -p udp --dport 443 -j REJECT --reject-with icmp-port-unreachable 2>/dev/null
pkill -9 nfqws2

# прогон TLS по реальным доменам (свежий googlevideo-хост из DevTools), GET для проверки 16 KB
nohup env BATCH=1 \
  DOMAINS="www.youtube.com i.ytimg.com rr1---sn-xxxx.googlevideo.com youtubei.googleapis.com" \
  IPVS=4 ENABLE_HTTP=0 ENABLE_HTTPS_TLS12=1 ENABLE_HTTPS_TLS13=1 ENABLE_HTTP3=0 \
  SCANLEVEL=standard REPEATS=5 CURL_HTTPS_GET=1 CURL=/tmp/curl \
  ./blockcheck2.sh > /home/Sashik259/bc_tls.log 2>&1 &
```
Итог: `grep -E 'working strategy found|^\* COMMON|not working' /home/Sashik259/bc_tls.log`.
Выбрать стратегию, проходящую **все** домены (включая `youtubei.googleapis.com`) на TLS 1.2 и 1.3,
вписать её в `NFQWS2_OPT`, `./init.d/sysv/zapret2 restart`, проверить на всех клиентах (§8).

Готовый кастомный тест с финалистами и QUIC-google-блобом — в `blockcheck2.d/yt/`.
