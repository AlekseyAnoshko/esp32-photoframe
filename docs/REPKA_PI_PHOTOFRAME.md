# Repka PI + PhotoFrame

Практическое руководство по подключению фоторамки **ESP32 PhotoFrame** к домашней
Wi-Fi сети с помощью одноплатного компьютера **Repka Pi 4** (клон Raspberry Pi 4
на базе Debian 12 / bookworm), когда прямой Wi-Fi провижининг с телефона или
ноутбука недоступен или неудобен.

Сценарий актуален, когда:

- у рамки нет прямого доступа к целевой домашней Wi-Fi сети во время настройки
  (например, рамка физически находится там, где ловит только сеть организации,
  а не домашний роутер);
- единственное доступное устройство для конфигурации — Repka Pi, подключённая
  к сети организации по кабелю (`end0`), у которой **один** Wi-Fi модуль
  (`wlan0`), то есть она не может одновременно быть Wi-Fi клиентом и точкой
  доступа.

## Содержание

1. [Подготовка Repka Pi: время и curl](#подготовка-repka-pi-время-и-curl)
2. [Особенности сети с корпоративным прокси](#особенности-сети-с-корпоративным-прокси)
3. [Формат запроса провижининга `/save`](#формат-запроса-провижининга-save)
4. [Проблема одного Wi-Fi модуля и тайминг 15 секунд](#проблема-одного-wi-fi-модуля-и-тайминг-15-секунд)
5. [Готовый скрипт полного цикла провижининга](#готовый-скрипт-полного-цикла-провижининга)
6. [Deep sleep рамки и мониторинг подключения](#deep-sleep-рамки-и-мониторинг-подключения)
7. [Доступ к веб-интерфейсу рамки после провижининга](#доступ-к-веб-интерфейсу-рамки-после-провижининга)
8. [Динамический image_url: датчики и время на экране](#динамический-image_url-датчики-и-время-на-экране)
9. [Диагностика: сводная таблица проблем](#диагностика-сводная-таблица-проблем)

---

## Подготовка Repka Pi: время и curl

На новой/долго не включавшейся Repka Pi часто нет `curl` и системные часы
могут быть сильно рассинхронизированы (в нашем случае — на **517 дней**),
если у платы нет RTC-батарейки. Это ломает проверку TLS/подписей `apt`-репозиториев
ещё до того, как дело дойдёт до самой рамки:

```
E: Release file for http://deb.debian.org/debian/... is not valid yet
   (invalid for another 517d 13h 45min 4s)
```

Порядок действий:

1. Установите/проверьте прокси для `apt`, если сеть требует его (см. следующий
   раздел).
2. Синхронизируйте время **до** попытки `apt-get install`:

   ```bash
   sudo apt-get install -y chrony
   sudo nano /etc/chrony/chrony.conf
   # добавьте строку:
   #   server <адрес_локального_NTP> iburst
   sudo systemctl restart chrony
   sudo systemctl enable chrony
   timedatectl status   # System clock synchronized: yes
   ```

   Если в сети нет своего NTP-сервера, временный ручной фикс:
   ```bash
   sudo date -s "2026-08-07 13:50:00"
   ```
   Учтите: `systemd-timesyncd` может отсутствовать на урезанных образах —
   в этом случае используйте `chrony`, который сам заменит устаревшие `ntp`/`ntpsec`.

3. Установите `curl`:
   ```bash
   sudo apt-get install -y curl
   ```

## Особенности сети с корпоративным прокси

Если Repka Pi находится в сети с обязательным HTTP-прокси (характерно для
университетских/корпоративных сетей), учтите два независимых момента:

**Для `apt`** — пропишите прокси в `/etc/apt/apt.conf.d/95proxy`:
```bash
echo 'Acquire::http::Proxy "http://<proxy-host>:<port>";'  | sudo tee /etc/apt/apt.conf.d/95proxy
echo 'Acquire::https::Proxy "http://<proxy-host>:<port>";' | sudo tee -a /etc/apt/apt.conf.d/95proxy
```

**Для `curl`/оболочки** — переменные `http_proxy`/`https_proxy` в `~/.bashrc`
заставят **все** HTTP-запросы, включая обращения к локальным IP-адресам
(`192.168.4.1`, `10.42.0.x`), уходить через прокси. Прокси-сервер организации
физически не может достучаться до локальных подсетей рамки/хотспота и вернёт
`ERR_CONNECT_FAIL` / `Connection refused` — при этом ошибка выглядит так, как
будто недоступна сама рамка, хотя на самом деле проблема только в прокси.

Решение — исключить локальные подсети через `NO_PROXY`:
```bash
echo 'export NO_PROXY="192.168.4.1,192.168.4.0/24,10.42.0.0/24,localhost,127.0.0.1"' >> ~/.bashrc
source ~/.bashrc
```
Разово, без правки `.bashrc`, можно добавить флаг `--noproxy '*'` к конкретному
вызову `curl`.

## Формат запроса провижининга `/save`

Провижининг-сервер рамки (файл `main/wifi_provisioning.c`) поднимается на
`192.168.4.1:80`, когда рамка сама выступает точкой доступа (`PhotoFrame - <ID>`).
Важный нюанс, легко упускаемый по аналогии с остальным REST API рамки
(`/api/config` и т.п., см. `docs/API.md`): обработчик `/save` **не** парсит JSON.

Он читает тело запроса через собственный простой парсер `get_form_field()`,
ожидающий `application/x-www-form-urlencoded` и короткие имена полей —
**`ssid`**, **`password`**, опционально **`deviceName`** (без префикса `wifi_`).

Неправильно (JSON, префиксованные поля) — вернёт `400 Bad Request`:
```bash
curl -X POST http://192.168.4.1/save \
  -H "Content-Type: application/json" \
  -d '{"wifi_ssid":"MySSID","wifi_password":"MyPass"}'
```

Правильно:
```bash
curl --noproxy '*' -X POST http://192.168.4.1/save \
  --data-urlencode "ssid=MySSID" \
  --data-urlencode "password=MyPass"
```

`curl` автоматически проставит `Content-Type: application/x-www-form-urlencoded`.

## Проблема одного Wi-Fi модуля и тайминг 15 секунд

Когда конфигурирующее устройство (Repka Pi) не имеет прямого доступа к целевой
домашней сети и приходится **временно поднимать точку доступа с тем же SSID
на самой Pi**, возникает состояние гонки, специфичное для плат с одним Wi-Fi
модулем.

Прошивка рамки после получения `/save` ждёт результата подключения к указанной
сети не бесконечно, а по таймеру (`main/wifi_provisioning.c`):
```c
EventBits_t bits = xEventGroupWaitBits(
    wifi_manager_get_event_group(), WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
    pdTRUE, pdFALSE,
    pdMS_TO_TICKS(15000)  // 15 second timeout
);
```

Последовательность, которая должна произойти **строго внутри этого 15-секундного
окна**:

1. Pi отключает свою временную AP (если уже была поднята), сканирует и
   подключается к точке доступа рамки как клиент.
2. Pi отправляет `/save` — рамка запускает 15-секундный таймер и пытается
   найти сеть с указанным SSID в эфире.
3. Pi должна успеть освободить `wlan0` и поднять свою собственную точку
   доступа с тем же SSID/паролем **до того**, как истекут 15 секунд, иначе
   рамка не найдёт сеть и вернёт `400 Bad Request` с телом:
   ```html
   <h1>WiFi Connection Failed</h1>
   <p>Could not connect to the WiFi network...</p>
   ```

Если ждать полного HTTP-ответа от `curl` до переключения Pi в режим AP,
теряются драгоценные секунды из этого окна. Решение — не дожидаться ответа:
запрос отправляется в фоне с коротким `--max-time`, а хотспот поднимается
почти сразу после того, как байты запроса гарантированно ушли на рамку.

```bash
curl -s --noproxy '*' --max-time 2 http://192.168.4.1/save \
  --data-urlencode "ssid=MySSID" \
  --data-urlencode "password=MyPass" &
sleep 1
sudo nmcli connection up MySSID-AP
```

### Совместимость хотспота Pi с ESP32

Хотспот, поднятый через `nmcli device wifi hotspot`, должен работать в
диапазоне **2.4 ГГц** (`band bg`) — ESP32 не поддерживает 5 ГГц. Если рамка
получает `WiFi Connection Failed` даже при верных SSID/пароле и правильном
таймере, проверьте PMF (Protected Management Frames) — старые Wi-Fi чипы
ESP32 иногда не совместимы с обязательным PMF, который некоторые версии
NetworkManager включают по умолчанию:

```bash
sudo nmcli connection modify <AP-имя> 802-11-wireless-security.pmf 1   # disable
sudo nmcli connection modify <AP-имя> 802-11-wireless.band bg
sudo nmcli connection modify <AP-имя> 802-11-wireless.channel 6
sudo nmcli connection down <AP-имя> && sudo nmcli connection up <AP-имя>
```

## Готовый скрипт полного цикла провижининга

```bash
#!/usr/bin/env bash
# Прогоняет полный цикл: отключение хотспота Pi -> подключение к AP рамки ->
# отправка /save -> возврат хотспота -> контроль появления рамки в сети.
#
# Переменные под вашу конфигурацию:
AP_CONN_NAME="K207ab-AP"          # имя nmcli-подключения хотспота на Pi
FRAME_AP_SSID="PhotoFrame - F89B0" # SSID точки доступа самой рамки
TARGET_SSID="K-207ab"              # SSID, который рамка должна запомнить
TARGET_PASSWORD="Home_Sweet-Home-207"

echo "==> Отключаем точку доступа ${AP_CONN_NAME}"
sudo nmcli connection down "$AP_CONN_NAME" 2>/dev/null || true
sleep 2

echo "==> Подключаемся к ${FRAME_AP_SSID}"
sudo nmcli device wifi rescan
sleep 3
sudo nmcli device wifi connect "$FRAME_AP_SSID"

IP=""
for i in $(seq 1 20); do
    IP=$(ip -4 addr show wlan0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
    [[ "$IP" == 192.168.4.* ]] && { echo "Получен IP: $IP"; break; }
    sleep 1
done

if [[ "$IP" != 192.168.4.* ]]; then
    echo "ОШИБКА: не удалось получить IP в сети рамки."
    exit 1
fi

echo "==> Отправляем /save в фоне (не ждём ответа — экономим 15-секундное окно)"
curl -s --noproxy '*' --max-time 2 http://192.168.4.1/save \
  --data-urlencode "ssid=${TARGET_SSID}" \
  --data-urlencode "password=${TARGET_PASSWORD}" &
CURL_PID=$!

echo "==> Сразу поднимаем хотспот ${AP_CONN_NAME}"
sleep 1
sudo nmcli connection up "$AP_CONN_NAME"
wait $CURL_PID 2>/dev/null

echo "==> Ждём подключения рамки (до 30 секунд)"
FOUND=0
for i in $(seq 1 30); do
    NEIGH=$(ip neigh show dev wlan0)
    if [[ -n "$NEIGH" ]]; then
        echo "Обнаружено устройство: $NEIGH"
        FOUND=1
        break
    fi
    sleep 1
done

[[ "$FOUND" -eq 0 ]] && echo "Рамка не подключилась за 30с — проверьте её экран."
```

## Deep sleep рамки и мониторинг подключения

ESP32 PhotoFrame уходит в **deep sleep** между обновлениями экрана, отключая
Wi-Fi модуль для экономии заряда (см. `main/power_manager.c`, `esp_sleep_enable_timer_wakeup`).
Это нормальное поведение, а не сбой — не пытайтесь трактовать пропажу
устройства из ARP-таблицы как ошибку сети.

Интервал пробуждения управляется **cron-подобным** планировщиком
(`main/cron.c`, `main/cron.h`), а не простым числом секунд:

```c
typedef struct {
    uint64_t minute;  // bits 0..59
    uint32_t hour;    // bits 0..23
    uint8_t  dow;     // bits 0..6 (Sunday = 0)
} cron_rule_t;
```

Если правил нет, используется запасной интервал `CRON_FALLBACK_SEC = 3600`
(1 час). Минимальный практический интервал ограничен только семантикой
cron-выражения (например, `* * *` — каждую минуту каждого часа каждого дня)
и здравым смыслом по энергопотреблению e-ink дисплея при частых обновлениях.

Полезные команды мониторинга состояния подключения рамки к хотспоту Pi:

```bash
ip neigh show dev wlan0     # REACHABLE / STALE / DELAY / FAILED
sudo ping -c3 <IP_рамки>    # физическая проверка связи (требует sudo из-за cap_net_raw)
watch -n 5 'ip neigh show dev wlan0'   # непрерывный мониторинг цикла сон/пробуждение
```

`FAILED` в ARP-таблице сразу после успешного провижининга обычно означает,
что рамка просто ушла в сон после применения настроек, а не что связь
разорвана навсегда — она вернётся в `REACHABLE` при следующем пробуждении.

## Доступ к веб-интерфейсу рамки после провижининга

После успешного `/save` рамка выходит из режима провижининга и поднимает
обычный рабочий веб-сервер (Vue SPA) на **порту 80** уже в целевой сети —
в нашем случае это сеть хотспота Pi (`10.42.0.x`):

```bash
curl --noproxy '*' http://<IP-рамки-в-целевой-сети>/
# HTTP/1.1 200 OK, <title>ESP32 PhotoFrame</title>
```

Не забудьте про `NO_PROXY`/`--noproxy` — тот же корпоративный прокси, что
мешал `/save`, будет мешать и здесь.

Основные REST-эндпоинты рабочего режима (полный список — `docs/API.md`):

- `GET /api/config` — текущая конфигурация (включает `ca_cert_set` для
  TLS-пиннинга при использовании HTTPS `image_url`).
- `PATCH /api/config` — изменение конфигурации, в т.ч.:
  ```json
  { "rotation_mode": "url", "image_url": "https://example.com/image" }
  ```
- `GET /api/ota/status` — статус OTA-обновления.

**TLS certificate pinning:** при смене `image_url` на HTTPS-адрес,
отличный от текущего, устройство скачивает и закрепляет TLS-сертификат
этого сервера перед применением конфигурации. Если получить сертификат не
удалось, запрос вернёт `400 Bad Request` с описанием ошибки в `message`, и
никаких изменений конфигурации не применяется. Переключение на HTTP или
очистка `image_url` сбрасывает ранее закреплённый сертификат.

## Динамический image_url: датчики и время на экране

Прошивка рамки **не** содержит встроенных виджетов (циферблатов, счётчиков,
часов) — она лишь периодически скачивает статичное изображение по указанному
`image_url` и отображает его на e-ink экране. Чтобы отображать живые данные
(температуру, уровень CO₂, время) в заранее подготовленных зонах макета
(например, отрисованных в стиле стимпанк-циферблатов), нужно генерировать
итоговое изображение **на стороне сервера** перед каждой отдачей рамке —
никаких изменений прошивки не требуется.

Схема:

```
[Датчики на Repka Pi] -> [Python/Flask сервер] -> генерирует PNG поверх фона -> рамка скачивает по image_url
```

Минимальный сервер на Flask + Pillow:

```python
from flask import Flask, send_file
from PIL import Image, ImageDraw, ImageFont
from datetime import datetime
import math, io

app = Flask(__name__)
BASE_IMAGE = "steampunk_base.png"  # заранее подготовленный фон

GAUGE_TEMP = {"center": (150, 480), "radius": 45,
              "min_val": 0, "max_val": 40,
              "start_angle": -120, "end_angle": 120,
              "needle_color": (120, 40, 20)}
GAUGE_CO2 = {"center": (280, 480), "radius": 45,
             "min_val": 400, "max_val": 2000,
             "start_angle": -120, "end_angle": 120,
             "needle_color": (30, 90, 60)}
DIGIT_DISPLAY = {"top_left": (600, 500), "font_size": 40, "color": (255, 140, 60)}

def read_temperature() -> float:
    ...  # чтение с DHT22 / BME280 и т.п.

def read_co2() -> int:
    ...  # чтение с MH-Z19 / SCD41 и т.п.

def draw_gauge_needle(draw, cfg, value):
    cx, cy = cfg["center"]
    frac = max(0.0, min(1.0, (value - cfg["min_val"]) / (cfg["max_val"] - cfg["min_val"])))
    angle = math.radians(cfg["start_angle"] + frac * (cfg["end_angle"] - cfg["start_angle"]) - 90)
    length = cfg["radius"] * 0.8
    x2, y2 = cx + length * math.cos(angle), cy + length * math.sin(angle)
    draw.line([(cx, cy), (x2, y2)], fill=cfg["needle_color"], width=4)
    draw.ellipse([cx-5, cy-5, cx+5, cy+5], fill=(80, 50, 20))

@app.route("/frame.png")
def frame():
    img = Image.open(BASE_IMAGE).convert("RGB")
    draw = ImageDraw.Draw(img)
    draw_gauge_needle(draw, GAUGE_TEMP, read_temperature())
    draw_gauge_needle(draw, GAUGE_CO2, read_co2())
    font = ImageFont.truetype("/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf",
                               DIGIT_DISPLAY["font_size"])
    draw.text(DIGIT_DISPLAY["top_left"], datetime.now().strftime("%H:%M"),
              font=font, fill=DIGIT_DISPLAY["color"])
    buf = io.BytesIO(); img.save(buf, format="PNG"); buf.seek(0)
    return send_file(buf, mimetype="image/png")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Затем укажите этот адрес рамке через `PATCH /api/config`:
```bash
curl -X PATCH --noproxy '*' http://<IP-рамки>/api/config \
  -H "Content-Type: application/json" \
  -d '{"rotation_mode": "url", "image_url": "http://<IP-Pi-в-сети-рамки>:5000/frame.png"}'
```

Для обновления раз в минуту настройте cron-правило автоповорота на
`* * *` (каждую минуту, каждый час, каждый день) через веб-интерфейс рамки
или соответствующее поле `PATCH /api/config`, помня, что это определяет,
как часто рамка **просыпается** и заново скачивает картинку — соответственно,
именно с этой частотой будут обновляться показания датчиков и время на экране.

## Диагностика: сводная таблица проблем

| Симптом | Причина | Решение |
|---|---|---|
| `apt-get install curl` виснет / `Release file... not valid yet` | Часы Pi рассинхронизированы (нет RTC-батарейки) | `chrony` + локальный NTP-сервер, либо `sudo date -s` |
| `wget`/`curl` к `192.168.4.1` — `Connection refused` / `ERR_CONNECT_FAIL` через squid | `http_proxy`/`https_proxy` перехватывают запросы к локальным адресам | `NO_PROXY` для `192.168.4.0/24`, `10.42.0.0/24` или флаг `--noproxy '*'` |
| `POST /save` → `400 Bad Request`, тело `Invalid JSON` | Обработчик ждёт `form-urlencoded`, а не JSON | `curl --data-urlencode` вместо `-d '{...}'` |
| `POST /save` → `400 Bad Request`, тело `Missing SSID` | Неверные имена полей (`wifi_ssid` вместо `ssid`) | Использовать точные имена `ssid`/`password`/`deviceName` |
| `POST /save` → `WiFi Connection Failed` | Сеть с нужным SSID не появилась в эфире за 15 секунд | Синхронизировать поднятие хотспота Pi с окном ожидания прошивки |
| Рамка периодически "пропадает" из `ip neigh` (`FAILED`) | Deep sleep — нормальное энергосберегающее поведение | Дождаться следующего цикла пробуждения по cron-расписанию |
| `ping` — `missing cap_net_raw+p capability` | У бинарника `ping` нет нужного capability в урезанном образе | `sudo ping ...` или `sudo setcap cap_net_raw+p /bin/ping` |
| `curl` к рамке зависает даже при рабочей сети (ping проходит) | Тот же прокси, теперь для обычного веб-интерфейса рамки | Тот же `NO_PROXY`/`--noproxy '*'` |
| Нужно показывать датчики/часы на статичной картинке рамки | Прошивка не поддерживает виджеты, только скачивание готового изображения | Локальный Flask-сервер, дорисовывающий данные на кадре перед отдачей по `image_url` |

---

*Раздел составлен по итогам практической настройки связки Repka Pi 4 +
ESP32 PhotoFrame в условиях сети с обязательным HTTP-прокси и без прямого
доступа к целевой домашней Wi-Fi сети во время провижининга.*
