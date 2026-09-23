# Алгоритм отправки обновлений агентом (sendResult)

## Эндпоинты

| Метод | Путь | Назначение |
|-------|------|-----------|
| `PUT` | `/api/agent/task/{id}/process` | Пометить задачу как "в обработке" |
| `PUT` | `/api/agent/task/{id}` | Отправить результат / промежуточное обновление |

---

## Когда вызывается update

Update вызывается **в двух случаях**:

### 1. Промежуточный update — на каждое новое сообщение в разговоре

`HandledConversation` оборачивает `Chat` и перехватывает `addMessage()` и `addServiceMessage()`.
После каждого добавленного сообщения автоматически вызывается `fire()`:

```
addMessage() / addServiceMessage()
    └─ fire()
        └─ handler->handle(ProcessingResult(completed: false, ...нули...))
            └─ PUT /api/agent/task/{id}
```

В промежуточном update:
- `completed = false`
- `answer = null`
- `context = null`
- все токены = 0
- `chat` — текущее состояние разговора (без `DisappearingMessage`)

### 2. Финальный update — после завершения итерации агента

`CodeProcessor` вызывает `processHandler->handle(processingResult->withAnswer($answer))`
после каждой итерации генератора `agent->execute(...)`:

```
foreach ($generator as $processingResult)
    └─ $processHandler->handle($processingResult->withAnswer($answer))
        └─ PUT /api/agent/task/{id}
```

В финальном update заполнены реальные данные.

---

## Структура тела запроса `PUT /api/agent/task/{id}`

```json
{
    "agent_uuid": "uuid агента",
    "completed": true | false,
    "result": "текстовый ответ агента или null",
    "model": "название LLM-модели или null",
    "context_fill": 0.75,
    "chat": [ /* сериализованная история разговора */ ],
    "context": {
        "tasks": [ /* список подзадач */ ],
        "payload": { /* произвольные namespace-данные */ }
    },
    "stats": {
        "prompt_tokens": 1500,
        "completion_tokens": 800,
        "total_tokens": 2300
    }
}
```

### Описание полей

| Поле | Тип | Промежуточный | Финальный | Описание |
|------|-----|:---:|:---:|---------|
| `agent_uuid` | string | + | + | UUID агента-исполнителя |
| `completed` | bool | `false` | `true`/`false` | Завершена ли задача |
| `result` | string\|null | `null` | текст | Итоговый текстовый ответ (`store-description` tool) |
| `model` | string\|null | `null` | имя модели | LLM-модель, которую использовал агент |
| `context_fill` | float | `0` | 0.0–1.0 | Заполненность контекстного окна LLM |
| `chat` | array | частичный | полный | История диалога (см. ниже) |
| `context` | object\|null | `null` | объект | Состояние контекста задачи |
| `context.tasks` | array | — | список | Подзадачи агента и их статусы |
| `context.payload` | object | — | данные | Произвольные данные по namespace-ам |
| `stats.prompt_tokens` | int\|null | `0` | число | Токены на вход в LLM |
| `stats.completion_tokens` | int\|null | `0` | число | Токены на выход из LLM |
| `stats.total_tokens` | int\|null | `0` | число | Сумма токенов |

---

## Сериализация `chat`

Поле `chat` формируется через `Chat::serialize()`.

**Включаются** все типы сообщений, кроме одного:
- `DisappearingMessage` — **исключается** (временные подсказки агенту, например "завершите все задачи")

Остальные 11 типов попадают в массив:

```json
[
    { "type": "user", "message": { ... } },
    { "type": "assistant", "message": { ... } },
    { "type": "tool_call", "message": { ... } },
    { "type": "tool_result", "message": { ... } },
    ...
]
```

Примеры `DisappearingMessage`, которые **не сохраняются**:
- `"Complete the work according to the current task list."` (CODE_WORK_PROMPT)
- `"You have uncompleted tasks. Complete all tasks."` (REMEMBER_FINISH_TASKS)

---

## Ответ от docmodule

```json
{ "status": "...", "message": "..." }
```

Если `status === "stopped"` — агент выбрасывает `AgentTaskStopped` и немедленно завершает работу.

---

## Полный поток вызовов

```
Runner.run()
 └─ api->getTask()                         GET /api/agent/task (получить задачу)
 └─ api->setStartProcessing()              PUT /api/agent/task/{id}/process
 └─ router->resolve(task)->process(task, handler)
     └─ HandledConversation (обёртка)
         └─ addMessage() → fire()          PUT /api/agent/task/{id}  [completed=false]
         └─ addMessage() → fire()          PUT /api/agent/task/{id}  [completed=false]
         └─ ...
     └─ agent->execute() → generator
         └─ processHandler->handle()       PUT /api/agent/task/{id}  [completed=true/false]
         └─ (если остались задачи — повтор цикла)
```
