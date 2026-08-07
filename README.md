# prometey_tg_ads — прокладка для Telegram Ads

Content-less «отскок» для трафика с **Telegram Ads**. Задача: собрать сигналы
матчинга Meta и сразу перекинуть человека в бота, чтобы его события
(StartTrial/Purchase) попадали в **Meta Events Manager** с хорошим match quality —
сид для **lookalike-аудитории**.

Это НЕ лендинг с контентом. Один файл `index.html`: быстрый лоадер → сбор данных →
авто-редирект в `t.me/PUMP_Prometheus_bot?start=fbc_<token>`.

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

## Использование

Ставим URL прокладки как ссылку объявления в Telegram Ads:

- по умолчанию: `https://<pages-url>/` → источник `tgads`;
- под конкретную кампанию: `https://<pages-url>/?src=<code>`, где `<code>` —
  `[A-Za-z0-9]{1,32}`. Чтобы этот `src` завёлся в воронке «Посевы» сервис-бота,
  код должен существовать в `ad_sources` (создать через админку). На CAPI это не
  влияет — fbp/IP/UA собираются в любом случае.

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
