# Контракт API — «Воздушный Шар»

Меняется только по согласованию обеих сторон (см. `CLAUDE.md`). Любое изменение —
отдельный коммит, сразу сказать напарнику.

Base URL: `http://localhost:8080/api`
WebSocket (STOMP over SockJS): `http://localhost:8080/ws`

## Демо-пользователь

Один фиксированный гостевой пользователь на весь прототип, без логина/пароля.

- id: `guest`
- стартовый баланс: `1000` бонусных баллов (задаётся в `config/game.json` →
  `demo_user.starting_balance`)
- баланс общий на всех, кто открыл прототип (не персонализирован по браузеру) —
  этого достаточно для проверки экспертом. Хранится в H2, не сбрасывается между
  запросами, сбрасывается при перезапуске сервера.

Все запросы ниже неявно выполняются от лица `guest` — отдельный auth-слой не
делаем, `userId` в путях/телах не передаётся.

## Общий формат ошибки

```json
{ "error": "INSUFFICIENT_BALANCE", "message": "Не хватает бонусов" }
```

Коды: `INSUFFICIENT_BALANCE`, `INVALID_BET_OPTION`, `ROUND_NOT_ACTIVE`,
`ALREADY_CASHED_OUT`, `VALIDATION_ERROR`.

## REST-эндпоинты

### `GET /api/state`
Стартовое состояние при загрузке фронта.

Response 200:
```json
{
  "balance": 1000,
  "theme": "green",
  "activeRound": null,
  "betOptions": {
    "green": [
      { "id": "no-boost", "cost": 50, "boostMultiplier": 1 },
      { "id": "boost-x2", "cost": 150, "boostMultiplier": 2 },
      { "id": "boost-x3", "cost": 300, "boostMultiplier": 3 },
      { "id": "boost-x4", "cost": 600, "boostMultiplier": 4 }
    ],
    "red": [ "...то же для красной темы, свои цены/множители из конфига..." ]
  },
  "levelsCount": { "green": 9, "red": 12 }
}
```
Если `activeRound` не null — фронт восстанавливает незавершённый раунд (перезагрузка
страницы), подписывается на его WebSocket-топик.

### `GET /api/rules`
Текст правил игры (см. `docs/rules.md` — источник контента). Отдаёт готовый HTML/markdown
блок, чтобы не дублировать текст на фронте и бэке.

Response 200:
```json
{ "content": "## Как играть\n\n..." }
```

### `GET /api/history?limit=20`
История завершённых раундов всех пользователей прототипа, свежие первыми.

Response 200:
```json
{
  "items": [
    {
      "roundId": "r-102",
      "theme": "green",
      "bet": 150,
      "result": "cashout",
      "multiplier": 1.85,
      "points": 278,
      "finishedAt": "2026-09-11T10:22:31Z"
    },
    {
      "roundId": "r-101",
      "theme": "red",
      "bet": 300,
      "result": "crash",
      "multiplier": 0.72,
      "points": 0,
      "finishedAt": "2026-09-11T10:20:05Z"
    }
  ]
}
```

### `POST /api/round/start`
Списывает ставку, сервер уже вычислил (но не раскрыл) точку краха и позицию
бустера, создаёт раунд, возвращает его id и provably-fair хеш.

Request:
```json
{ "theme": "green", "betOptionId": "boost-x2" }
```

Response 200:
```json
{
  "roundId": "r-103",
  "theme": "green",
  "bet": 150,
  "boostMultiplier": 2,
  "levelsCount": 9,
  "levelThresholds": [1.10, 1.25, 1.45, 1.70, 2.00, 2.50, 3.20, 4.50, 7.00],
  "resultHash": "3f9c2a...e1",
  "balanceAfter": 850
}
```
`levelThresholds` — коэффициенты, при пересечении которых засчитывается переход на
следующий уровень (длина = `levelsCount`). `resultHash` — sha256 от
`(crashPoint, boostLevelIndex, serverSeed)`, публикуется ДО полёта; серверный seed
раскрывается только на `GET /api/round/{roundId}` после завершения раунда — для
проверки честности.

Ошибки: `INSUFFICIENT_BALANCE`, `INVALID_BET_OPTION`.

Сразу после успешного ответа фронт подписывается на WebSocket-топик
`/topic/round/{roundId}` и получает поток тиков коэффициента (см. ниже).

### `POST /api/round/{roundId}/cashout`
Фиксирует выигрыш по текущему коэффициенту. Коэффициент на момент нажатия
пересчитывается и валидируется на сервере по времени старта раунда — клиентское
значение коэффициента не принимается как аргумент.

Response 200:
```json
{
  "roundId": "r-103",
  "cashedOutAt": 2.35,
  "winAmount": 352,
  "pointsSoFar": 180
}
```
Ошибки: `ROUND_NOT_ACTIVE` (раунд уже завершён crash-ом), `ALREADY_CASHED_OUT`.

Раунд после этого продолжает жить до фактического crash — `winAmount` уже
зафиксирован и не меняется, шар долетает визуально по тем же WS-событиям.

### `GET /api/round/{roundId}`
Итог завершённого раунда (после события `round.finished` по WS) — экран
результата запрашивает эту ручку, чтобы не терять данные при разрыве соединения.

Response 200:
```json
{
  "roundId": "r-103",
  "theme": "green",
  "bet": 150,
  "outcome": "cashout",
  "cashedOutAt": 2.35,
  "crashAt": 3.10,
  "winAmount": 352,
  "points": 278,
  "reward": { "type": "puzzle-piece", "id": "green-7" },
  "serverSeed": "8b7a...",
  "resultHash": "3f9c2a...e1"
}
```
`outcome`: `cashout` | `crash`. При `crash` — `winAmount: 0`, `cashedOutAt: null`,
`reward` заполняется по тем же правилам (награда даётся всегда, не только при
выигрыше).

### `GET /api/config`
Текущая игровая конфигурация (для админки — доп. модуль). Отдаёт содержимое
`config/game.json` как есть.

### `PUT /api/config`
Обновление конфигурации (доп. модуль, админка). Валидирует значения, применяет с
hot-reload, следующий раунд считается по новым параметрам. Требует пере-проверки
допустимых диапазонов на сервере (не доверять фронту).

## WebSocket (STOMP)

Подключение: `SockJS` на `/ws`, дальше STOMP `CONNECT`.

### Подписка: `/topic/round/{roundId}`

События, `type` — дискриминатор:

**`tick`** — обновление коэффициента, ~10 раз/сек, пока раунд активен:
```json
{ "type": "tick", "multiplier": 1.42, "elapsedMs": 820 }
```

**`level`** — пересечён уровень (индекс с 0):
```json
{ "type": "level", "levelIndex": 2, "pointsAwarded": 10, "totalPoints": 30 }
```

**`boost`** — шар долетел до уровня с бустером (только если cashout ещё не было):
```json
{ "type": "boost", "levelIndex": 5, "boostMultiplier": 2, "multiplierAfter": 4.20 }
```

**`round.finished`** — крах, раунд завершён:
```json
{
  "type": "round.finished",
  "crashAt": 3.10,
  "outcome": "cashout",
  "winAmount": 352,
  "points": 278,
  "serverSeed": "8b7a..."
}
```
После этого события фронт отписывается от топика и запрашивает
`GET /api/round/{roundId}` для итогового экрана.

### Подписка: `/topic/leaderboard` (доп. модуль — живой рейтинг)

```json
{
  "type": "leaderboard.update",
  "entries": [
    { "playerId": "guest", "points": 1240, "isCurrentUser": true },
    { "playerId": "bot-3", "points": 980, "isCurrentUser": false }
  ]
}
```
Не обязателен для MVP — реализуется после стабилизации основного цикла.

## Договорённости по типам данных

- Все денежные/очковые величины — целые числа (`int`), баллы не дробные.
- Коэффициенты (`multiplier`, `crashAt`, `levelThresholds`) — `number`, 2 знака
  после запятой на отображении, на бэке храним с большей точностью.
- Время — ISO 8601 UTC (`2026-09-11T10:22:31Z`).
- `theme`: строковый enum `"green" | "red"`, никаких магических чисел на фронте.
