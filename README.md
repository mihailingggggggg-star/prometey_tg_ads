# prometey_tg_ads — прокладка/Mini App для Telegram Ads

Content-less «отскок»: собирает сигналы матчинга Meta (fbp/IP/UA + fbc/src) и
уводит человека в бота, чтобы его события (StartTrial/Purchase) попадали в **Meta
Events Manager** с хорошим match quality — сид для **lookalike-аудитории**.

Один файл `index.html`, работает в **двух режимах** (определяется автоматически):

- **Telegram Mini App** (боевой заход из Telegram Ads). Официальный кабинет Telegram
  Ads НЕ пускает внешние ссылки — только `t.me/…`, поэтому реклама ведёт на
  **Direct Link Mini App** `t.me/PUMP_Prometheus_bot/<app>?startapp=<код>`. Страница
  открывается в webview Telegram: есть `initData` (юзер + `start_param`), код
  кампании берётся из `start_param`, а уход в бота — через `tg.openTelegramLink`.
- **Обычная веб-страница** (посевы: клик по ссылке `…/?src=<код>` в посте канала/
  чата или внешняя реклама). `initData` пуст → код из `?src`, уход через
  `window.location`.

Пиксель, сбор атрибуции, токен «гардероба» и телеметрия — **одинаковые** в обоих
режимах. Бота и Worker менять не нужно.

## Как работает

1. Грузится Meta-пиксель (`1345428627003718`), host-gated — ставит cookie `_fbp`
   и шлёт браузерный `PageView`.
2. JS собирает атрибуцию (порт `prometey_landing/src/lib/fbAttribution.ts`):
   - `fbc` — из cookie `_fbc` → `?fbclid` → `localStorage` (у ТГ-трафика обычно нет);
   - `src` — из `?src=<code>`; если ни `fbc`, ни `src` — подставляется `DEFAULT_SRC`;
   - `fbp` — ждём cookie `_fbp` до 2с.
3. `POST` в Cloudflare Worker «гардероб» (`/store`, заголовок `X-Write-Key`) →
   короткий токен. Worker сам дописывает **IP** и **UA** из заголовков запроса.
4. Редирект в бота с `?start=fbc_<token>`. Бот резолвит токен
   (`meta_capi.resolve_attribution`), пишет `fbp/client_ip/client_ua` на юзера и
   шлёт CAPI с матчем **fbp + IP + UA + имя** (имя — из профиля Telegram).

### Почему обязателен `src`

В Telegram Ads **нет `fbclid`**, поэтому `fbc` пустой. Worker `/store` и резолвер
бота требуют `fbc` ИЛИ `src`. Без `src` запись не создастся и fbp/IP/UA не
сохранятся. Поэтому прокладка **всегда** шлёт `src` (из URL или `DEFAULT_SRC`).

## Константы (в `index.html`)

| Что | Значение |
|---|---|
| Бот | `PUMP_Prometheus_bot` |
| Worker | `https://prometheus-fbc.mihailingggggggg.workers.dev` |
| Write-key | `prm_9f3Kx7Qa2Ld8Zt` (тот же, что на основной прокладке) |
| Meta Pixel | `1345428627003718` |
| `DEFAULT_SRC` | `tgads` |

Тот же Worker и pixel, что у `prometey_landing` — **менять их и бота не нужно**.

## Регистрация Mini App (BotFather) — один раз

1. @BotFather → `/newapp` → выбрать бота `PUMP_Prometheus_bot`.
2. Title / короткое описание / картинку (640×360) — по вкусу.
3. **Web App URL:** `https://mihailingggggggg-star.github.io/prometey_tg_ads/`
4. **Short name:** напр. `go` → появится ссылка `t.me/PUMP_Prometheus_bot/go`.

Ссылка кампании для рекламы: `t.me/PUMP_Prometheus_bot/go?startapp=<код>` — код
прилетит в `start_param`. Проверить, какие формы принимает кабинет Telegram Ads
(с `?startapp=` и без).

## Использование

- **Telegram Ads (официальный кабинет):** ссылка объявления —
  `t.me/PUMP_Prometheus_bot/go?startapp=<код>` (Mini App режим).
- **Посевы (посты в каналах/чатах) и внешняя реклама:** ссылка —
  `https://<pages-url>/?src=<код>` (веб-режим).

`<код>` — `[A-Za-z0-9]{1,32}`. Чтобы код завёлся в воронке «Посевы» сервис-бота,
он должен существовать в `ad_sources` (создать через админку → «🔵 Telegram Ads»).
На CAPI это не влияет — fbp/IP/UA собираются в любом случае. Без кода —
`DEFAULT_SRC=tgads`.

## Деплой (GitHub Pages)

1. Репозиторий: https://github.com/mihailingggggggg-star/prometey_tg_ads
2. Settings → Pages → Source: **GitHub Actions** (workflow `.github/workflows/pages.yml`
   публикует корень репо как есть, без сборки). Либо «Deploy from branch: main / root».
3. URL будет `https://mihailingggggggg-star.github.io/prometey_tg_ads/`.

## Локальная проверка

```
cd prometey_tg_ads && python3 -m http.server 8080
```

- `http://localhost:8080/?fbclid=TEST123` → в Network виден `POST /store` → редирект
  на `t.me/PUMP_Prometheus_bot?start=fbc_<token>` (длина `start` = 26).
- `http://localhost:8080/?src=abc` → store с `src=abc` → редирект.
- `http://localhost:8080/` → сработал `DEFAULT_SRC=tgads` → store → редирект.
- На `localhost` пиксель НЕ инициализируется (host-gate), но редирект-логика та же.
