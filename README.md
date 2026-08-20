# Zabbix template for Ruijie / Reyee EW1200G-PRO (eWeb + Cloud)

Monitor a Ruijie **Reyee EW1200G-PRO** router in **Zabbix 7.4** — no SNMP, no SSH, no agent on the router. Everything is pulled over the router's own **eWeb** web-interface API, plus an optional second template that uses the **Ruijie Cloud** API for CPU / flash (which the local API doesn't expose).

> Made for and tested on Zabbix **7.4** and ReyeeOS **1.308**. Other Reyee "Home WiFi" models will probably work too, since they share the same eWeb API — just double-check the item keys after import.

🇬🇧 English below · 🇺🇦 [Українською нижче](#-українська)

---

## What you get

**Local template** (`reyee_ew1200g_pro.yaml`) — the main one:

- **WAN throughput** in / out (live rate)
- **Clients**: total, wired, wireless, per band (2.4G / 5G), per SSID
- **Per-client** monitoring for a whitelist you choose: online/offline, RSSI, Wi-Fi rate, channel, traffic, which AP it's on
- **Wi-Fi health per radio**: channel utilization (airtime), noise floor, TX power, channel — for **every** AP (main unit + mesh nodes)
- **Mesh**: each AP online/offline, client count per AP
- **Ports**: link status, speed, duplex
- **System**: uptime, memory, firmware, serial
- **Alarms** the router raises itself, and **IP-conflict** detection
- Triggers for all the important stuff, and 5 auto-discovery rules so new clients / APs / ports / SSIDs appear on their own

**Cloud template** (`reyee_cloud.yaml`) — optional add-on:

- **CPU**, **flash**, memory, process count — *not available from the local API*
- Online status as seen by Ruijie Cloud, WAN IP, port states

You can use just the local one, or both on the same host. Their item keys don't clash.

---

## ⚠️ The one thing everybody trips on

The router password macro **must be plain "Text", not "Secret text"**.

Zabbix does **not** expand *Secret* macros inside Script-item parameters — if you make `{$REYEE.PASSWORD}` a Secret macro, every item fails with a login error and nothing works. This is a Zabbix limitation, not a bug in the template.

So: set `{$REYEE.PASSWORD}` as **Text**. On a home Zabbix that only you can see, that's fine. (If you really need it hidden, the only proper option is a Vault — that's beyond this README.)

---

## Install (local template)

### 1. Import the template file

1. In Zabbix: **Data collection → Templates → Import** (top-right).
2. Choose `reyee_ew1200g_pro.yaml`.
3. Leave the default import rules ticked → **Import**.

You should see *"Imported successfully"*.

### 2. Create a host for your router (skip if you already have one)

1. **Data collection → Hosts → Create host**.
2. **Host name**: anything you like, e.g. `192.168.1.1` or `Reyee-Router`.
3. **Templates**: search for and add **`Ruijie Reyee EW1200G-PRO by eWeb API`**.
4. **Interfaces**: add an **Agent** interface with the router's IP (e.g. `192.168.1.1`), port can stay `10050` — it isn't actually used, the template talks HTTP by itself.
5. **Host groups**: pick or create any group.
6. Don't press *Add* yet — do step 3 first.

### 3. Set the macros (this is where the password goes)

On the host, open the **Macros** tab → **Inherited and host macros**. At minimum set:

| Macro | Value | Notes |
|---|---|---|
| `{$REYEE.PASSWORD}` | your eWeb login password | **Type must be `Text`** — see the warning above |
| `{$REYEE.SCHEME}` | `http` | use `https` only if your router's web UI is on HTTPS |

Optional macros you can tune later:

| Macro | Default | What it does |
|---|---|---|
| `{$REYEE.CLIENT.MATCHES}` | *(sample MACs)* | which clients get their own detailed items. A regex of MAC addresses, **lowercase**, separated by `\|`. Add a MAC here and its graphs appear on the next discovery run — no re-import needed. Set to `.*` to track every client (expect churn from phones/guests). |
| `{$REYEE.CLIENT.NOT_MATCHES}` | `^$` | MACs to exclude, applied after the one above. |
| `{$REYEE.CHUTIL.MAX}` | `70` | channel-utilization % that triggers the "congested" warning |
| `{$REYEE.RSSI.MIN}` | `-80` | dBm below which a whitelisted client is "weak signal" |
| `{$REYEE.MEM.PUSED.MAX}` | `90` | memory % for the high-memory trigger |

**Replace the sample MACs in `{$REYEE.CLIENT.MATCHES}` with your own devices** — the ones shipped are from the author's network and mean nothing on yours. To find a device's MAC: log into eWeb → Clients, or check `Latest data` after the first poll (the discovery will list everyone; you pick who to keep).

Now press **Add** to save the host.

### 4. Check it works

1. **Data collection → Hosts → your host → Items**.
2. Find **`Reyee: Get data`**, tick it, press **Execute now** at the bottom.
3. Do the same for **`Reyee: Get clients`**.
4. Go to **Monitoring → Latest data**, filter by your host. Within a minute you should see uptime, memory, WAN rate, client counts, etc.

If an item is red (unsupported), hover the error in the **Info** column — it usually says exactly what's wrong (wrong password, router unreachable, HTTPS vs HTTP).

---

## Install (cloud template — optional, for CPU / flash)

You only need this if you want CPU and flash usage, which the local API can't give.

**You need an appid + secret from Ruijie.** Email `service_rj@ruijienetworks.com` and ask for Ruijie Cloud API access — tell them your name, your cloud account, your country, and that it's for personal monitoring. They send back an `appid` and `secret`.

Then:

1. Import `reyee_cloud.yaml` the same way as before.
2. Add the template **`Ruijie Reyee by Cloud API`** to the *same* host (or a new one).
3. Set these macros — again, all **Text**, not Secret:

| Macro | Value |
|---|---|
| `{$RUIJIE.CLOUD.URL}` | your Ruijie Cloud region URL, no trailing slash (the same address you log into, e.g. `https://cloud-eu.ruijienetworks.com`) |
| `{$RUIJIE.APPID}` | the appid Ruijie gave you |
| `{$RUIJIE.SECRET}` | the secret Ruijie gave you |
| `{$RUIJIE.SN}` | your router's serial number |

4. **Execute now** on `Cloud: Get data`, then check Latest data.

> **Heads-up on the cloud template:** the EW1200G-PRO is a "Home WiFi" device, and Ruijie Cloud does **not** report clients or traffic for that device class — only CPU / memory / flash / online / ports. That's a limit on Ruijie's side; the local template remains your source for clients and WAN traffic. The cloud also depends on the internet: if your uplink drops, this template goes stale while the router itself is still fine — that's why the "offline in cloud" trigger is only a Warning.

---

## Dashboard

`reyee_dashboard.yaml` is a ready host-dashboard (WAN graph, clients, Wi-Fi health, mesh, system).

Note: Zabbix 7.4 **can't import a host dashboard from the web UI** — only via the API. If you're comfortable with the API:

```bash
ZBX_URL="http://YOUR-ZABBIX/api_jsonrpc.php"
ZBX_TOKEN="your-api-token"

jq -n --rawfile src reyee_dashboard.yaml '{
  jsonrpc:"2.0", method:"configuration.import", id:1,
  params:{ format:"yaml", source:$src,
    rules:{ dashboards:{createMissing:true, updateExisting:true} } }
}' | curl -s -X POST "$ZBX_URL" \
  -H "Content-Type: application/json-rpc" \
  -H "Authorization: Bearer $ZBX_TOKEN" -d @- | jq
```

The dashboard file references the host by name — it ships pointing at `192.168.1.1`. If your host has a different name, replace it first:

```bash
sed -i "s/value: '192.168.1.1'/value: 'YOUR-HOST-NAME'/g" reyee_dashboard.yaml
```

It shows up under **Monitoring → Hosts → your host → Dashboards**, not in the global Dashboards list. If you'd rather have it global, easiest is to rebuild it by hand in the UI.

---

## How it works (for the curious)

The template logs into eWeb (`POST /cgi-bin/luci/api/auth`), gets a session id, then calls the internal RPC modules the web UI uses. A couple of hard-won details baked into the scripts:

- Requests are **signed**: each call carries `Content-Accept` = `md5("Web@Rj$2020!" + <body length in bytes>)` and `Contents-Accept` = `md5("Web@Rj$2020!" + <body>)`. The template computes these for you.
- **WAN rate** comes from the `flow` module with `data:{func:"interface_info"}` — a live bits-per-second value, not a byte counter. Byte counters don't exist anywhere in this router's local API, so the WAN graph is an instantaneous rate sampled each poll (bump `Reyee: Get data` to a 1-minute interval if you want finer resolution).
- Client data comes from `user_list`; per-AP Wi-Fi metrics from `ap_list`.

---
---

## 🇺🇦 Українська

# Шаблон Zabbix для Ruijie / Reyee EW1200G-PRO (eWeb + Cloud)

Моніторинг роутера Ruijie **Reyee EW1200G-PRO** у **Zabbix 7.4** — без SNMP, без SSH, без агента на роутері. Усе тягнеться через власний веб-інтерфейс роутера **eWeb**, плюс окремий необов'язковий шаблон, що бере CPU / flash через **Ruijie Cloud** (локальний API їх не віддає).

> Зроблено й перевірено на Zabbix **7.4** та ReyeeOS **1.308**. Інші моделі Reyee «Home WiFi», найпевніше, теж працюватимуть — просто перевірте ключі ітемів після імпорту.

---

## Що всередині

**Локальний шаблон** (`reyee_ew1200g_pro.yaml`) — основний:

- **Швидкість WAN** вхід / вихід (миттєва)
- **Клієнти**: усього, дротові, бездротові, по діапазонах (2.4G / 5G), по SSID
- **По кожному клієнту** зі списку, який ви оберете: онлайн/офлайн, RSSI, швидкість Wi-Fi, канал, трафік, до якої точки під'єднаний
- **Здоров'я Wi-Fi по кожному радіо**: утилізація ефіру (airtime), рівень шуму, потужність, канал — для **всіх** точок (головна + mesh)
- **Mesh**: кожна точка онлайн/офлайн, кількість клієнтів на точці
- **Порти**: стан лінка, швидкість, дуплекс
- **Система**: аптайм, пам'ять, прошивка, серійник
- **Аларми**, які роутер піднімає сам, і виявлення **конфлікту IP**
- Тригери на все важливе і 5 правил автовиявлення — нові клієнти / точки / порти / SSID з'являються самі

**Хмарний шаблон** (`reyee_cloud.yaml`) — необов'язкове доповнення:

- **CPU**, **flash**, пам'ять, кількість процесів — *локальний API цього не дає*
- Статус онлайн з погляду Ruijie Cloud, WAN IP, стан портів

Можна ставити лише локальний, або обидва на один хост — ключі ітемів не конфліктують.

---

## ⚠️ Головна пастка, на якій спотикаються всі

Макрос із паролем **має бути звичайним «Text», а не «Secret text»**.

Zabbix **не підставляє** *Secret*-макроси в параметри Script-ітемів — якщо зробити `{$REYEE.PASSWORD}` типу Secret, усі ітеми впадуть з помилкою логіну і нічого не працюватиме. Це обмеження Zabbix, а не помилка шаблону.

Тому: `{$REYEE.PASSWORD}` — тип **Text**. На домашньому Zabbix, який бачите тільки ви, це нормально. (Якщо пароль конче треба ховати — єдиний правильний варіант це Vault, але це поза межами цього README.)

---

## Встановлення (локальний шаблон)

### 1. Імпорт файлу шаблону

1. У Zabbix: **Data collection → Templates → Import** (вгорі праворуч).
2. Виберіть `reyee_ew1200g_pro.yaml`.
3. Галки правил імпорту лишіть за замовчуванням → **Import**.

Має з'явитися *«Imported successfully»*.

### 2. Створіть хост для роутера (пропустіть, якщо вже є)

1. **Data collection → Hosts → Create host**.
2. **Host name**: будь-яка назва, напр. `192.168.1.1` або `Reyee-Router`.
3. **Templates**: знайдіть і додайте **`Ruijie Reyee EW1200G-PRO by eWeb API`**.
4. **Interfaces**: додайте інтерфейс **Agent** з IP роутера (напр. `192.168.1.1`), порт хай лишається `10050` — він не використовується, шаблон сам ходить по HTTP.
5. **Host groups**: виберіть або створіть будь-яку групу.
6. Ще не тисніть *Add* — спершу зробіть крок 3.

### 3. Задайте макроси (сюди йде пароль)

На хості відкрийте вкладку **Macros → Inherited and host macros**. Мінімум задайте:

| Макрос | Значення | Примітки |
|---|---|---|
| `{$REYEE.PASSWORD}` | ваш пароль від eWeb | **Тип має бути `Text`** — див. попередження вище |
| `{$REYEE.SCHEME}` | `http` | `https` лише якщо веб-інтерфейс роутера на HTTPS |

Необов'язкові макроси, які можна підлаштувати згодом:

| Макрос | За замовч. | Що робить |
|---|---|---|
| `{$REYEE.CLIENT.MATCHES}` | *(приклад MAC-ів)* | які клієнти отримають власні детальні ітеми. Регулярка з MAC-адрес, **малими літерами**, через `\|`. Додали MAC — його графіки з'являться на наступному циклі виявлення, переімпорт не потрібен. Значення `.*` = стежити за всіма (тоді буде «сміття» від телефонів/гостей). |
| `{$REYEE.CLIENT.NOT_MATCHES}` | `^$` | MAC-и, які виключити, застосовується після попереднього. |
| `{$REYEE.CHUTIL.MAX}` | `70` | % утилізації ефіру, за яким спрацьовує попередження «канал перевантажений» |
| `{$REYEE.RSSI.MIN}` | `-80` | dBm, нижче якого клієнт зі списку вважається «слабкий сигнал» |
| `{$REYEE.MEM.PUSED.MAX}` | `90` | % пам'яті для тригера високого навантаження |

**Замініть приклади MAC-ів у `{$REYEE.CLIENT.MATCHES}` на свої пристрої** — ті, що в комплекті, з мережі автора і на вашій нічого не означають. Щоб дізнатися MAC пристрою: зайдіть в eWeb → Clients, або гляньте `Latest data` після першого опитування (виявлення покаже всіх — ви оберете, кого лишити).

Тепер натисніть **Add**, щоб зберегти хост.

### 4. Перевірка

1. **Data collection → Hosts → ваш хост → Items**.
2. Знайдіть **`Reyee: Get data`**, поставте галку, внизу натисніть **Execute now**.
3. Те саме для **`Reyee: Get clients`**.
4. Зайдіть у **Monitoring → Latest data**, відфільтруйте по хосту. За хвилину мають з'явитися аптайм, пам'ять, швидкість WAN, кількість клієнтів тощо.

Якщо ітем червоний (unsupported) — наведіть на помилку в колонці **Info**, там зазвичай прямо написано, що не так (неправильний пароль, роутер недоступний, HTTPS замість HTTP).

---

## Встановлення (хмарний шаблон — необов'язково, для CPU / flash)

Потрібен лише якщо хочете бачити CPU і flash, які локальний API дати не може.

**Потрібні appid + secret від Ruijie.** Напишіть на `service_rj@ruijienetworks.com` і попросіть доступ до Ruijie Cloud API — вкажіть ім'я, ваш обліковий запис у хмарі, країну і що це для персонального моніторингу. У відповідь надішлють `appid` і `secret`.

Далі:

1. Імпортуйте `reyee_cloud.yaml` так само, як раніше.
2. Додайте шаблон **`Ruijie Reyee by Cloud API`** на *той самий* хост (або новий).
3. Задайте макроси — знову всі **Text**, не Secret:

| Макрос | Значення |
|---|---|
| `{$RUIJIE.CLOUD.URL}` | URL вашого регіону Ruijie Cloud, без слеша в кінці (та сама адреса, під якою заходите, напр. `https://cloud-eu.ruijienetworks.com`) |
| `{$RUIJIE.APPID}` | appid від Ruijie |
| `{$RUIJIE.SECRET}` | secret від Ruijie |
| `{$RUIJIE.SN}` | серійний номер роутера |

4. **Execute now** на `Cloud: Get data`, потім гляньте Latest data.

> **Важливо про хмарний шаблон:** EW1200G-PRO — це пристрій класу «Home WiFi», і Ruijie Cloud для цього класу **не** віддає клієнтів чи трафік — лише CPU / пам'ять / flash / онлайн / порти. Це обмеження на боці Ruijie; джерелом клієнтів і WAN-трафіку лишається локальний шаблон. Хмара також залежить від інтернету: якщо впаде аплінк, цей шаблон «застигне», хоча роутер живий — тому тригер «офлайн у хмарі» лише Warning.

---

## Дашборд

`reyee_dashboard.yaml` — готовий дашборд хоста (графік WAN, клієнти, здоров'я Wi-Fi, mesh, система).

Зверніть увагу: Zabbix 7.4 **не вміє імпортувати дашборд хоста через веб-інтерфейс** — тільки через API. Якщо вам зручно з API:

```bash
ZBX_URL="http://ВАШ-ZABBIX/api_jsonrpc.php"
ZBX_TOKEN="ваш-api-токен"

jq -n --rawfile src reyee_dashboard.yaml '{
  jsonrpc:"2.0", method:"configuration.import", id:1,
  params:{ format:"yaml", source:$src,
    rules:{ dashboards:{createMissing:true, updateExisting:true} } }
}' | curl -s -X POST "$ZBX_URL" \
  -H "Content-Type: application/json-rpc" \
  -H "Authorization: Bearer $ZBX_TOKEN" -d @- | jq
```

Файл дашборда посилається на хост за назвою — у комплекті вказано `192.168.1.1`. Якщо ваш хост зветься інакше, спершу замініть:

```bash
sed -i "s/value: '192.168.1.1'/value: 'НАЗВА-ВАШОГО-ХОСТА'/g" reyee_dashboard.yaml
```

Він з'явиться у **Monitoring → Hosts → ваш хост → Dashboards**, а не в загальному списку Dashboards. Якщо хочете глобальний — найпростіше зібрати його руками в інтерфейсі.

---

## Як воно працює (для допитливих)

Шаблон логіниться в eWeb (`POST /cgi-bin/luci/api/auth`), отримує id сесії й далі викликає ті самі внутрішні RPC-модулі, що й веб-інтерфейс. Кілька важких деталей, зашитих у скрипти:

- Запити **підписуються**: кожен виклик несе `Content-Accept` = `md5("Web@Rj$2020!" + <довжина тіла в байтах>)` і `Contents-Accept` = `md5("Web@Rj$2020!" + <тіло>)`. Шаблон рахує це за вас.
- **Швидкість WAN** береться з модуля `flow` з `data:{func:"interface_info"}` — це миттєве значення в біт/с, не лічильник байтів. Лічильників байтів у локальному API цього роутера немає ніде, тому графік WAN — це моментальний замір на кожному опитуванні (поставте `Reyee: Get data` на інтервал 1 хвилина, якщо треба детальніше).
- Дані клієнтів — з `user_list`; метрики Wi-Fi по точках — з `ap_list`.

---

## License / Ліцензія

MIT — роби що хочеш, гарантій жодних. / MIT — do whatever, no warranty.
