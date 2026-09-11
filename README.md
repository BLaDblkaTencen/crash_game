# Воздушный Шар

Бонусная crash-игра с турнирной механикой — кейсовое задание Чемпионата России по
продуктовому программированию, ФСП, онлайн-этап 11–13 сентября 2026

Игрок ставит бонусные баллы и старается забрать растущий выигрыш до того, как
воздушный шар лопнет (crash-механика). Подробности — в `docs/zadacha-razbor.md`.

## Статус

Проект в разработке. Ядро игровой механики (сценарии 1–5 из ТЗ) — в работе.

## Стек

- **Backend:** Java 21, Spring Boot 3, WebSocket (STOMP over SockJS), H2 (in-memory)
- **Frontend:** React 18, Vite, TypeScript
- **Конфигурация:** `config/game.json` (hot-reload, без пересборки кода)

## Структура репозитория

```
backend/    Java/Spring Boot — игровая логика, WebSocket, REST API
frontend/   React/TS — экраны игры
config/     game.json — параметры игровой экономики
docs/       контракт API, правила игры, обоснование мат. модели, разбор ТЗ
```

## Документация

- [`docs/zadacha-razbor.md`](docs/zadacha-razbor.md) — разбор исходного ТЗ
- [`docs/api.md`](docs/api.md) — контракт REST + WebSocket
- [`docs/rules.md`](docs/rules.md) — правила игры (текст для игрока)
- [`docs/math-model.md`](docs/math-model.md) — математическая модель crash-раунда
- [`CLAUDE.md`](CLAUDE.md) — правила разработки, роли, git-флоу

## Запуск (заполняется по мере готовности backend/frontend)

### Backend

```bash
cd backend
./mvnw spring-boot:run
```
Поднимется на `http://localhost:8080`, WebSocket — `http://localhost:8080/ws`.

### Frontend

```bash
cd frontend
npm install
npm run dev
```
Поднимется на `http://localhost:5173`, использует backend по адресу выше.

### Демо-доступ

Отдельного логина нет — один фиксированный гостевой пользователь (`guest`) со
стартовым балансом из `config/game.json` (`demo_user.starting_balance`).

## Команда

Разработка ведётся двумя участниками параллельно по границе backend/frontend —
см. `CLAUDE.md`.
