# AnyDevModules

Мета-репозиторий платформы AnyModule — распределённой системы для автоматического выполнения задач LLM-агентами. Компоненты подключены как git submodule, а корневой `docker-compose.yaml` поднимает их вместе.

| Компонент | Стек | Роль |
|-----------|------|------|
| [docmodule](https://github.com/vasenin26/docmodule) | PHP 8.4, Laravel 12, Vue + Inertia, PostgreSQL | Центральный узел: проекты, задачи, агенты, документация, API |
| [agentmanager](https://github.com/vasenin26/agentmanager) | Go, Docker SDK, BBolt, Prometheus | Оркестратор: опрашивает задачи и запускает контейнеры агентов |
| [agentmodule](https://github.com/vasenin26/agentmodule) | PHP 8.3 CLI | Эфемерный воркер: выполняет одну задачу с помощью LLM |
| [conversation](https://github.com/vasenin26/conversation) | PHP 8.2, composer-библиотека | Общий формат сообщений разговора (используется agentmodule) |

Подробнее — [docs/architecture.md](docs/architecture.md) и [docs/agent-update-algorithm.md](docs/agent-update-algorithm.md).

## Как это связано

```
браузер ──► docmodule :8000 ◄──── /api/orchestrator ──── agentmanager :8080
                ▲                                             │ docker.sock
                │ /api/agent (через хост)                     ▼
                └──────────────────────────────────── agentmodule (контейнер на задачу)
```

- `agentmanager` каждые `TASK_POLL_INTERVAL` берёт задачу из docmodule, резервирует её и запускает контейнер из образа `AGENT_IMAGE`.
- Агент получает детали задачи и отправляет результаты в docmodule по `AGENT_API_HOST`.
- docmodule обращается к оркестратору по `http://agentmanager:8080`.

## Клонирование

```bash
git clone --recurse-submodules git@github.com:vasenin26/anydevmodules.git
```

Если репозиторий уже склонирован без сабмодулей:

```bash
git submodule update --init --recursive
```

## Запуск

1. Подготовить окружение:

   ```bash
   cp .env.example .env
   ```

   Обязательно заполнить `APP_KEY` (`echo "base64:$(openssl rand -base64 32)"`), `OPENAI_API_KEY`, `GIT_USER_NAME`, `GIT_USER_EMAIL`.

2. Собрать и поднять стек:

   ```bash
   docker compose up -d --build
   ```

   Сервис `agentmodule` только собирает образ агента и сразу завершается — это нормально.

3. Выпустить токен оркестратора. Это JWT системного агента «Orchestrator Agent» с `has_cross_project_access`: он получает задачи всех проектов, и агенты отправляют результаты от его имени.

   Сначала зарегистрировать первого пользователя на http://localhost:8000 — сидер создаёт системный проект с `owner_id=1`. Затем:

   ```bash
   docker compose exec docmodule php artisan db:seed --class=OrchestratorAgentSeeder --force
   ```

   Сидер выведет `JWT Token: ...`. Если агент уже создан, сидер токен не печатает — его можно достать так:

   ```bash
   docker compose exec docmodule php artisan tinker --execute="echo App\Models\Agent::where('has_cross_project_access', true)->value('token');"
   ```

   Записать токен в `ORCHESTRATOR_TOKEN` в `.env` и пересоздать оркестратор:

   ```bash
   docker compose up -d agentmanager
   ```

   Токен подписан `APP_KEY`: после смены `APP_KEY` его нужно выпустить заново.

4. Сбор логов (Loki + Promtail), по желанию:

   ```bash
   docker compose --profile logging up -d
   ```

### Сеть агентов

Контейнеры агентов agentmanager создаёт в стандартной bridge-сети Docker, а не в сети compose, поэтому до docmodule они добираются через хост:

- Docker Desktop (Windows / macOS / WSL2): `AGENT_API_HOST=http://host.docker.internal:8000/api`
- Docker Engine на Linux: `AGENT_API_HOST=http://172.17.0.1:8000/api`

## Работа с сабмодулями

Каждый компонент — отдельный репозиторий со своей историей и релизами. Изменения коммитятся и пушатся внутри сабмодуля, а в корневом репозитории фиксируется новая ссылка на коммит:

```bash
cd docmodule && git commit -am "..." && git push && cd ..
git add docmodule && git commit -m "Bump docmodule"
```

Обновить все сабмодули до последних коммитов их веток:

```bash
git submodule update --remote --merge
```
