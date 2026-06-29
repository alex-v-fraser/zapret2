Кастомный тест YouTube для blockcheck2 (TEST=yt).

Состав:
  10-list.sh            — раннер списочного теста (копия из custom/)
  list_https_tls12.txt  — финалисты TLS, проверяются и под 1.2
  list_https_tls13.txt  — те же финалисты под 1.3 (цель: найти ОДНУ стратегию для обоих)
  list_quic.txt         — QUIC с "белым" google-блобом quic_google (SNI-трюк)
  list_http.txt         — пусто (HTTP не используем)

Запускать НА РОУТЕРЕ из каталога проекта (/config/zapret2), из-под root.
Лог пишем в /home/<user>/ (переживёт ребут), curl — /tmp/curl с HTTP3/TLS1.3.

--- 1. Контрольный прогон TLS (выбрать одну стабильную стратегию для 1.2+1.3+видео) ---
Главное: CURL_HTTPS_GET=1 (проверка "16 KB" на длинном GET) и REPEATS=5 (стабильность).
Подставьте СВЕЖИЙ googlevideo-хост (старый rr...-хост эфемерный, мог протухнуть).

cd /config/zapret2
nohup env BATCH=1 TEST=yt \
  DOMAINS="www.youtube.com i.ytimg.com s.ytimg.com yt3.ggpht.com rr1---sn-xxxx.googlevideo.com" \
  IPVS=4 ENABLE_HTTP=0 ENABLE_HTTPS_TLS12=1 ENABLE_HTTPS_TLS13=1 ENABLE_HTTP3=0 \
  REPEATS=5 CURL_HTTPS_GET=1 CURL=/tmp/curl CURL_CMD=1 \
  ./blockcheck2.sh > /home/Sashik259/bc_yt_tls.log 2>&1 &
echo "PID=$!"

Выбираем строку, которая в SUMMARY проходит и tls12, и tls13 по ВСЕМ доменам
(включая googlevideo) на всех 5 повторах. Её и кладём в NFQWS2_OPT (Фаза 4).

--- 2. QUIC с google-блобом (SNI-трюк) ---
cd /config/zapret2
nohup env BATCH=1 TEST=yt \
  DOMAINS="www.youtube.com rr1---sn-xxxx.googlevideo.com" \
  IPVS=4 ENABLE_HTTP=0 ENABLE_HTTPS_TLS12=0 ENABLE_HTTPS_TLS13=0 ENABLE_HTTP3=1 \
  REPEATS=5 CURL=/tmp/curl CURL_CMD=1 \
  ./blockcheck2.sh > /home/Sashik259/bc_yt_quic.log 2>&1 &
echo "PID=$!"

Если всё равно сплошной UNAVAILABLE (code=28) — QUIC/UDP-443 режется целиком,
google-блоб не помогает. Тогда в боевом конфиге QUIC ГЛУШИМ (drop), браузер уйдёт на TLS.
