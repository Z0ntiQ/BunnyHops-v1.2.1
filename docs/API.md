# Bunny Hops v1.1.0 — справочник HTTP-API

База: `http://127.0.0.1:<порт>` — порт в заголовке окна (оконная версия) или значение `PORT`
(headless). Все эндпоинты — JSON, кроме явно указанных. Все ответы идут с `Cache-Control: no-store`.
Формат ошибок: `{"ok": false, "error": "<текст>"}` + честный HTTP-статус.

## Прокси

| Метод | Путь | За что отвечает |
|---|---|---|
| любой | `/proxy?url=<целевой URL>` | Стриминговый прокси против CORS. Метод — `-X`, тело — как есть (GET/HEAD игнорируется), query уже внутри `url`. |

Контракт:
- Заголовки цели — JSON-строкой в заголовке запроса `X-Proxy-Headers: {"Content-Type":"application/json",...}`; hop-by-hop вырезаются.
- Ответ: реальный статус и заголовки цели (кроме content-length/encoding), тело стримом.
- После редиректов (макс. 10) — заголовок `X-Proxy-Final-Url` с финальным адресом.
- Ошибки: нет `?url=` или невалидный URL → `400` (+`X-Proxy-Error: invalid-url`);
  сеть → `502` + `X-Proxy-Error: dns-not-found | connection-refused | timeout`.
- Лимит тела запроса: 32 МиБ → `413` до обращения к цели. Таймауты: 10 с соединение, 120 с всего.

Примеры:
```bash
curl -s "http://127.0.0.1:3000/proxy?url=https://api.example.com/v1/ping"
curl -s -X POST "http://127.0.0.1:3000/proxy?url=https://api.example.com/x" \
  -H 'X-Proxy-Headers: {"Content-Type":"application/json"}' -d '{"a":1}'
curl -s -o /dev/null -D- "http://127.0.0.1:3000/proxy?url=https://api.example.com/x"  # статус+заголовки
```

## Control API (автоматизация без UI)

| Метод | Путь | За что отвечает |
|---|---|---|
| GET | `/api/status` | Живость и снимок состояния: версия, uptime, активный профиль, число профилей/истории/журнала, снапшот мок-движка, состояние гейта debug-API. Точка health-check. |
| POST | `/api/request` | Выполнить запрос сервером и попасть в историю. Тело: `{url}` или `{profile, path}`, плюс `{method, headers, query, body, timeout_s}`. Ответ цели возвращается целиком. |
| GET | `/api/history` | Последние выполненные запросы (в памяти процесса). |
| POST | `/api/history/clear` | Очистить историю. |
| GET | `/api/profiles` | Список именованных баз (name, base_url, заголовки по умолчанию) + активная. |
| POST | `/api/profiles` | Создать/обновить базу: `{name, base_url?, headers?}`. Дальше `/api/request` работает по `profile + path`. |
| POST | `/api/profiles/activate` | Сделать базу активной: `{name}`. |
| POST | `/api/profiles/delete` | Удалить базу: `{name}`. |
| GET | `/api/theme` | Текущий режим темы (`dark`). |
| POST | `/api/theme` | Режим темы: `{mode: "dark"|"light"}` (UI сейчас только тёмный). |
| GET | `/api/action-log` | Журнал всех действий приложения: `{items: [...], total}` — cap 500, свежие сверху. |
| POST | `/api/log` | Приёмник заметок UI: `{area, action, details}` → тот же журнал. |
| GET | `/api/automation-rules` | Журнал прокси-вызовов НЕ из UI (curl/скрипты): метод, URL, реальный статус. Ответы UI-вызовов (с `x-hop-ui`) сюда не попадают. |

## Мок-режим (BunnyProx)

| Метод | Путь | За что отвечает |
|---|---|---|
| GET | `/api/mock/profiles` | Список мок-профилей. |
| GET | `/api/mock/profile` | Профиль целиком: правила, варианты ответов, порты. |
| POST | `/api/mock/profile` | Создать/обновить профиль: `{name, listen_port, upstream?, rules: [{path, method, variants: [...]}]}`. Сохраняется в `~/.bunny_hops/mock/<имя>.json`. |
| POST | `/api/mock/profile/delete` | Удалить профиль. |
| POST | `/api/mock/start` | Запустить мок-сервер на выбранный порт. |
| POST | `/api/mock/stop` | Остановить. |
| GET | `/api/mock/status` | Работает ли мок, на каком порту. |
| GET | `/api/mock/traffic` | Журнал: какие запросы приходили и чем отвечено; агрегация по вызовам. |
| POST | `/api/mock/preview` | Показать вариант ответа с раскрытыми токенами-рандомайзерами, до запуска мока. |

Токены подстановки в вариантах ответа: `{{guid}}`, `{{int:}}`, `{{float:}}`, `{{bool}}`,
`{{date:±N}}`, `{{datetime:}}`, `{{string:}}`, `{{pick: A|B}}`, `{{email}}`, `{{phone}}`,
`{{hex:}}`, `{{inn}}`, `{{snils}}`, `{{name}}` — каждое обращение даёт свежее значение.
Совпал путь+метод → вариант ответа; не совпало → 404.

Важное изменение v1.1.0: запущенный мок слушает `0.0.0.0` — его порт доступен
из локальной сети (корпоративный стенд зовёт его как внешнюю систему).
Остальные HTTP-серверы приложения — только `127.0.0.1`.

## Кадр экрана

| Метод | Путь | За что отвечает |
|---|---|---|
| GET | `/api/eyes` | Снимок текущего окна приложения (PNG). Параметры: `json=1` — метаданные вместо кадра, `full=1` — без масштабирования. По умолчанию широкие кадры режутся к ширине 2060 (масштаб «разглядеть мелкий UI»). Только оконная версия. |
| GET | `/screenshot` | Склейка кадров окна приложения (панель запроса + ответ) через CDP. Публичный, без гейта debug-API; попадает в журнал действий. В headless без окна — `502` + `{"error": ...}`, в таймауте CDP — `504`. |

## JSON-RPC мост `/rpc` (новое в v1.1.0)

Единая POST-точка JSON-RPC 2.0 для внешнего клиента (MCP-мост). Под тем же гейтом
debug-API, что и `/debug/*`; запросы самого UI — без изменений.

Координаты подключения: `~/.bunny_hops/rpc.json` (права `0600`) — `{port, token}`.
Авторизация: `Authorization: Bearer <token>`; без токена — `-32600 Bearer token required`.

| Метод JSON-RPC | За что отвечает |
|---|---|
| `system.get_state` | Полный снимок состояния приложения (включая `elements` — карту живых элементов формы). |
| `ui.inspect` | Снимок элементов UI по области: id, имя, роль, координаты. |
| `ui.click` | Клик по id/имени элемента. |
| `ui.type` | Ввод текста в элемент. |
| `ui.wait` | Ждать условие над снимком состояния (предикат по element/state) с таймаутом. |
| `app.execute` | Действие из реестра `/debug/actions` (dispatch). |

Пример:
```bash
T=$(python3 -c "import json;print(json.load(open('$HOME/.bunny_hops/rpc.json'))['token'])")
P=$(python3 -c "import json;print(json.load(open('$HOME/.bunny_hops/rpc.json'))['port'])")
curl -s -X POST "http://127.0.0.1:$P/rpc" -H "Authorization: Bearer $T" \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"system.get_state","params":{}}'
```

Поверх моста работает MCP-сервер **broop** (mcp 2.x): тот же функциональ для агента
как MCP-инструменты (`inspect_ui_tree`, `ui_click_by_id`, `ui_set_text_by_id`,
`execute_business_action`, `get_application_state`, `wait_for`, …). Прямой HTTP не нужен.

## Debug API (MCP-поверхность для агентов)

Единая программная поверхность для автоматизации: состояние, действия, кадры, мок — всё под одним гейтом.
Приложение полностью работает при выключенном MCP: UI, `/proxy`, Control API, `/screenshot` — без изменений.

Гейт (доступен всегда, без гейта):

| Метод | Путь | За что отвечает |
|---|---|---|
| GET | `/api/debug-mode` | Состояние гейта: `{ok, enabled}`. |
| POST | `/api/debug-mode` | Включить/выключить: `{"on": true\|false}`. То же — тумблер «MCP» в UI. |

Пока гейт выключен, все `/debug/*` отвечают `404` + `{"error": "debug_api_disabled", "hint": "включи тумблер MCP в UI или POST /api/debug-mode {"on": true}"}`; WS `/debug/stream` отклоняет рукопожатие.
Исключения: `GET /debug/openapi.json` — контракт доступен всегда; запросы самого UI (с заголовком `x-hop-ui`) проходят независимо от гейта.

| Метод | Путь | За что отвечает |
|---|---|---|
| GET | `/debug/openapi.json` | OpenAPI 3.0 контракт всей debug-поверхности — источник истины по схемам. |
| GET | `/debug/state` | Полный снимок состояния приложения (включая `ui.state._clicks` — список кликабельных имён UI, и `elements` — карту живых элементов). |
| GET/POST/PATCH | `/debug/state` | POST — перезаписать путь целиком `{"path": "...", "value": ...}`; PATCH — deep merge. |
| GET/POST/PATCH | `/debug/state/<путь>` | Значение по пути, сегменты через точку (`mock.profiles.mdm.routes`); POST — записать, PATCH — deep merge. |
| GET | `/debug/actions` | Реестр действий: `name`, описание, пример `payload`. |
| POST | `/debug/dispatch` | Единая точка действий: `{"action": "...", "payload": ...}`. Действия: `mock.start/stop`, `mock.profile.save/delete`, `traffic.clear`, `logs.clear`, `ui.set` (путь состояния UI), `ui.click` (кнопка по имени из `_clicks`), `ui.reset`. |
| POST | `/debug/elements` | Снимок живых элементов формы (id, имя, роль, координаты) + round-trip обновления карты. |
| POST | `/debug/eval-out` | Выполнить выражение в контексте приложения, вернуть результат. |
| POST | `/debug/window/activate` | Поднять окно приложения на передний план. |
| GET | `/debug/broop`, `/debug/broop/get` | Состояние и снимок для MCP-сервера broop. |
| POST | `/debug/reset` | Сброс тестового окружения: трафик моков + журнал действий + UI к значениям по умолчанию. |
| GET | `/debug/logs` | Журнал действий (свежие первыми); POST `/debug/log` — заметка в тот же журнал. |
| GET | `/debug/eyes` | Кадр окна приложения (PNG), как `/api/eyes`. |
| GET | `/debug/stream` | WS-поток событий приложения. События действий идут в конверте `{"type":"action", ...}` (v1.1.0). |
| — | `/debug/request`, `/debug/history`, `/debug/profiles`, `/debug/theme`, `/debug/proxy`, `/debug/mock/*`, `/debug/automation-rules` | Те же функции, что у соответствующих `/api/*` разделов выше, в единой debug-поверхности. |

Примеры:
```bash
curl -s -X POST http://127.0.0.1:3000/api/debug-mode -H 'content-type: application/json' -d '{"on":true}'
curl -s http://127.0.0.1:3000/debug/actions | python3 -m json.tool      # что можно делать
curl -s http://127.0.0.1:3000/debug/state | jq '._clicks // .ui'       # имена кнопок UI
curl -s -X POST http://127.0.0.1:3000/debug/dispatch -H 'content-type: application/json' \
  -d '{"action":"ui.click","payload":{"name":"mock"}}'                  # клик кнопки
```

Контракт и практические заметки:
- Ответ `dispatch` для UI-действий может быть `{"status":"pending"}` («UI-клиент не подтвердил за 2 с») при уже применившемся действии — сверяй кадром или следующим снимком, не повторяй клик.
- Снимок состояния после UI-действия может отставать на одно действие — окончательный вердикт состояния снимай `/debug/eyes` (или diff двух кадров), `/debug/state` используй как направление.
- Имена `ui.click` не выдумывай: читай массив `_clicks` в снапшоте — он обновляется вместе с открытыми экранами.

## Статика

| Метод | Путь | За что отвечает |
|---|---|---|
| GET | `/`, любые пути SPA | Раздача веб-интерфейса из `dist/`; неизвестный путь → index.html (SPA-fallback). |

## Порядок работы агента с API (устоявшийся процесс)

1. Живость: `GET /api/status` → `ok: true`.
2. Запрос к внешней системе: через `/proxy` (тот же путь, что у UI, без CORS) или `/api/request`.
3. Что уходило в API — читать в `/api/automation-rules`, а не по памяти.
4. Мок вместо внешней системы: создать профиль → preview → start → работать с его портом
   (он в LAN, стенд зовёт сам) → traffic как доказательство → удалить профиль.
5. Любое действие уже в `/api/action-log` — отдельного логирования не нужно.
6. Полная автоматизация состоянием и UI (агент-тестировщик): включить гейт `POST /api/debug-mode {"on":true}` → читать контракт `/debug/openapi.json` → работать через `/debug/state` и `/debug/dispatch` → вердикт по кадру `/debug/eyes` или `/screenshot`.
7. Внешний MCP-клиент: читать `~/.bunny_hops/rpc.json` (порт+токен) → работать через broop или `/rpc` напрямую.
