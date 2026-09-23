# AnyModule — Архитектура и взаимодействие компонентов

## Обзор системы

Система представляет собой распределённую платформу для автоматизированного выполнения задач с помощью LLM-агентов. Состоит из четырёх компонентов:

| Компонент | Язык | Тип | Роль |
|-----------|------|-----|------|
| **docmodule** | PHP 8.2 / Laravel 12 | Web-приложение | Центральный узел: задачи, пользователи, API |
| **agentmanager** | Go 1.23 | Микросервис | Оркестратор: жизненный цикл контейнеров |
| **agentmodule** | PHP 8.2 CLI | Worker-контейнер | Исполнитель задач, интеграция с LLM |
| **conversation** | PHP 8.2 | Composer-библиотека | Унифицированный протокол сообщений |

---

## Схема взаимодействия

```
┌──────────────────────────────────────────────────────────────────┐
│                     Внешние системы                              │
│           OpenAI / GigaChat │ Git-репозитории │ YouGile          │
└────────────┬────────────────┴───────────────────┬───────────────┘
             │                                     │
    ┌────────▼──────────────────────────────────────▼────────┐
    │                     docmodule                           │
    │   (Laravel: задачи, агенты, проекты, страницы, API)    │
    │                                                         │
    │  /api/v1/orchestrator/*  │  /api/v1/agent/*            │
    └─────────┬───────────────────────────────┬──────────────┘
              │ (REST: получить задачу,        │ (REST: получить задачу,
              │  зарезервировать)              │  сохранить результат)
              │                                │
    ┌─────────▼──────────┐          ┌──────────▼─────────────┐
    │   agentmanager     │          │     agentmodule         │
    │   (Go-оркестратор) │─────────►│  (PHP CLI, эфемерный   │
    │                    │  Docker  │   Docker-контейнер)     │
    │  - Опрос задач     │  запуск  │                         │
    │  - Запуск агентов  │          │  - Опрос задач          │
    │  - Управление том. │          │  - LLM-интеграция       │
    │  - Prometheus метр.│          │  - Инструменты (файлы,  │
    │  - BBolt DB        │          │    тесты, git)          │
    └────────────────────┘          └──────────┬──────────────┘
                                               │ использует
                                    ┌──────────▼──────────────┐
                                    │      conversation        │
                                    │  (Composer-библиотека)   │
                                    │                          │
                                    │  12 типов сообщений,     │
                                    │  Conversation, Message,  │
                                    │  Factory, Validator      │
                                    └──────────────────────────┘
```

---

## Жизненный цикл задачи

### 1. Создание задачи
Пользователь через веб-интерфейс **docmodule** создаёт задачу (`AgentTask`).
Статус: `pending`. Тип: `code`, `actualization`, `implementation` и др.

### 2. Получение задачи оркестратором
**agentmanager** каждые 5 секунд делает запрос:
```
GET /api/v1/orchestrator/tasks/next
Authorization: TASK_API_TOKEN
```
**docmodule** возвращает `OrchestratorTaskDTO` (задача + публичный SSH-ключ проекта) или `204 No Content`.

### 3. Резервирование задачи
```
POST /api/v1/orchestrator/tasks/{id}/reserve
Body: { agentUuid, reserveSeconds }
```
**docmodule** блокирует строку в БД — предотвращает гонку условий.
Статус задачи: `reserved`.

### 4. Запуск агента
**agentmanager** запускает Docker-контейнер с образом **agentmodule**, передавая через переменные окружения:
- `AGENT_UUID` — идентификатор агента
- `AGENT_API_TOKEN` — JWT/токен для авторизации
- `AGENT_MODEL` — имя LLM-модели (OpenAI, GigaChat, …)
- Монтирует Docker volume (контекст) → `/home/local/context`
- Пробрасывает Unix-сокет для Docker-in-Docker

### 5. Выполнение задачи агентом
**agentmodule** (`Runner.php`) в цикле:
1. Запрашивает задачу у **docmodule**: `GET /api/v1/agent/tasks/{id}`
2. Маршрутизирует по типу (`CodeWorkflow`, `Actualization`, …)
3. Строит разговор через библиотеку **conversation**
4. Отправляет сообщения в LLM (OpenAI / GigaChat)
5. Применяет результат: редактирует файлы, запускает тесты
6. Отправляет результат: `POST /api/v1/agent/tasks/{id}/update`

### 6. Завершение
**agentmanager** отслеживает события Docker-контейнера:
- Собирает логи, коды выхода
- Освобождает Docker volume
- Очищает ресурсы

**docmodule** обновляет статус задачи: `done` или `error`.

---

## Компонент: docmodule

**Технологии**: Laravel 12, Vue.js + Inertia.js, Eloquent ORM
**Роль**: Единый центр управления — хранит все данные, предоставляет API.

### Ключевые сущности
- `Agent` — зарегистрированный агент
- `AgentTask` — задача (статусы: `pending` → `reserved` → `processing` → `done`/`error`)
- `Project` — проект с SSH-ключами и настройками
- `Page` — страница документации (контент в Markdown)

### API-эндпоинты
| Метод | Путь | Потребитель | Назначение |
|-------|------|-------------|-----------|
| GET | `/api/v1/orchestrator/tasks/next` | agentmanager | Следующая задача |
| POST | `/api/v1/orchestrator/tasks/{id}/reserve` | agentmanager | Зарезервировать задачу |
| POST | `/api/v1/orchestrator/tasks/{id}/update-key` | agentmanager | Обновить SSH-ключ |
| GET | `/api/v1/agent/tasks/{id}` | agentmodule | Получить детали задачи |
| POST | `/api/v1/agent/tasks/{id}/update` | agentmodule | Сохранить результат |

### Ключевые сервисы
- `OrchestratorTaskService` — выдача и резервирование задач
- `AgentTaskManagerService` — жизненный цикл задач
- `PromptService` + `PromptTemplateRenderer` — генерация системных промптов
- `AgentJwtService` — JWT для агентов
- `TaskTrackerService` + YouGile-интеграция — синхронизация с внешним трекером

---

## Компонент: agentmanager

**Технологии**: Go 1.23, Docker SDK, BBolt, Prometheus, zap
**Роль**: Оркестратор — запускает агентов, управляет ресурсами, собирает метрики.

### Сервисы
- `OrchestratorService` — главный цикл: опрос задач, резервирование, запуск
- `AgentService` — pull образов, создание/старт контейнеров
- `ContextService` — управление Docker volumes (изолированные контексты)
- `QueueService` — локальная очередь задач на контекст
- `MemoryService` — контроль лимитов памяти контейнеров
- `TerminalProxy` — Unix-сокет для Docker-in-Docker

### Хранилище
BBolt (встраиваемая key-value БД):
- Состояния агентов
- Контексты (volumes)
- Очереди задач

### Метрики
Prometheus endpoint: `:8080/metrics`
Интегрируется с Loki + Promtail для сбора логов.

---

## Компонент: agentmodule

**Технологии**: PHP 8.2 CLI, Composer, OpenAI SDK, GigaChat SDK
**Роль**: Эфемерный воркер — выполняет одну задачу и завершается.

### Поддерживаемые LLM
- **OpenAI** (ChatProcessor)
- **GigaChat** (GigaChatProcessor)
- **StupidJoe** (заглушка для тестов)

### Инструменты агента (Tools)
- **Editor** — создание, редактирование, замена файлов
- **Tasks** — управление подзадачами
- **Tester** — запуск тестов
- **RepositoryService** — работа с Git

### Типы задач (Workflow)
- **CodeWorkflow** — генерация/изменение кода
- **ActualizationWorkflow** — обновление документации

### Зависимость от conversation
Агент строит `Chat` (объект разговора) из сообщений библиотеки **conversation**, сериализует его для передачи в LLM и обратно.

---

## Компонент: conversation

**Технологии**: PHP 8.2, без фреймворка
**Роль**: Общая библиотека — стандартизирует формат сообщений между компонентами.

### Типы сообщений (12 штук)
| Тип | Назначение |
|-----|-----------|
| `UserMessage` | Ввод пользователя |
| `AssistantMessage` | Ответ LLM |
| `SystemMessage` | Системная инструкция |
| `ServiceMessage` | Сервисное сообщение (с двунаправленной ссылкой) |
| `ToolMessage` | Результат вызова инструмента |
| `CallToolMessage` | Запрос вызова инструмента |
| `UserTaskMessage` | Инструкция-задача |
| `InfoMessage` | Информационное сообщение |
| `DisappearingMessage` | Временное сообщение (не сериализуется) |
| `GitFileMessage` | Привязка к файлу в Git |
| `PageVersionMessage` | Версия страницы документа |
| `SliceMessage` | Срез/пагинация сообщений |

### Ключевые интерфейсы
```php
interface Conversation {
    public function addMessage(Message $message): void;
    public function getMessages(): array;
    public function getInstructions(): Generator;  // UserTaskMessage
    public function getServices(): Generator;       // ServiceMessage
    public function serialize(): array;
}
```

---

## Аутентификация

| Соединение | Метод |
|-----------|-------|
| agentmanager → docmodule | Static token (`TASK_API_TOKEN` в заголовке) |
| agentmodule → docmodule | JWT (`AgentJwtService`) |
| agentmodule ↔ Git | SSH-ключи (генерируются agentmanager, хранятся в docmodule) |

---

## Деплой

```
Docker Host
├── docmodule           (Laravel, :8000)
│   └── PostgreSQL / MySQL / SQLite
│
├── agentmanager        (Go, :8080/metrics)
│   ├── BBolt DB        (./data/orchestrator.db)
│   └── SSH Keys Dir    (./keys/)
│
├── agentmodule-*       (эфемерные PHP-контейнеры)
│   ├── Docker Volume   (/home/local/context)
│   └── Unix Socket     (DinD proxy)
│
└── Logging Stack       (опционально)
    ├── Loki
    └── Promtail
```

---

## Ключевые архитектурные решения

1. **Pull-модель задач** — агенты сами опрашивают очередь, что упрощает горизонтальное масштабирование.
2. **Изоляция через Docker** — каждая задача выполняется в отдельном контейнере с собственным файловым контекстом.
3. **BBolt для оркестратора** — встраиваемая БД без внешних зависимостей для хранения состояния агентов.
4. **Единый протокол сообщений** — библиотека `conversation` исключает дублирование логики форматирования в разных компонентах.
5. **Pluggable LLM** — агент переключается между OpenAI, GigaChat и другими моделями через переменную окружения.
