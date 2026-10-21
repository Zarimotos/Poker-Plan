# API-контракт Planning Poker

**Задача:** PRE-03 — Определить интерфейсы взаимодействия для первого спринта  
**Владелец:** Системный аналитик  
**Связанные задачи:** S1-01–S1-06  
**Вход:** Модель данных из `data-model-specification.md`  
**Статус:** Черновик для согласования  

**Связанные документы:**

- [Модель данных](data-model-specification.md)

## 1. Цель

Определить минимальный API-контракт для первого пользовательского сценария Planning Poker.

API должен обеспечить:

- создание сессии;
- импорт задач из CSV;
- просмотр очереди;
- получение текущей задачи;
- выбор и подтверждение итогового SP;
- получение итогов сессии;
- восстановление состояния после обновления страницы.

## 2. Границы API

### В объёме

- Однопользовательский сценарий Host.
- Создание серверной сессии.
- Предпросмотр и подтверждённый импорт CSV.
- Получение очереди и текущей задачи.
- Фиксация итогового SP.
- Получение итогов сессии.
- Базовая обработка ошибок.

### Вне объёма

- Подключение Participant.
- Совместное голосование.
- Комментарии.
- Управление очередью.
- Таймер.
- Специальные карты.
- Переголосование.
- Поиск эталонов.
- Экспорт CSV.
- Интеграция с YouTrack.
- Импорт Excel.

## 3. Общие правила

Базовый префикс:

```text
/api/v1
```

Формат запросов и ответов: `application/json`, кроме загрузки CSV-файла.

API не должен вводить поля, отсутствующие в модели данных. Изменение модели данных требует проверки влияния на API-контракт.

## 4. Создание сессии

```http
POST /api/v1/sessions
```

**Назначение:** S1-01 — создание сессии и переход Host в лобби.

**Запрос:**

```json
{
  "name": "Спринт 1: 21.10.2026",
  "hostName": "Иванов Иван"
}
```

**Успешный ответ — `201 Created`:**

```json
{
  "id": "3f1c9d24-8e4b-4b7f-9a2e-8d6f0b1c1234",
  "name": "Спринт 1: 21.10.2026",
  "hostName": "Иванов Иван",
  "status": "CREATED",
  "createdAt": "2026-10-21T12:05:00Z",
  "updatedAt": "2026-10-21T12:05:00Z"
}
```

**Ошибки:**

| Код | HTTP | Текст |
|---|---:|---|
| `SESSION_NAME_REQUIRED` | 400 | «Не заполнено название сессии.» |
| `SESSION_NAME_TOO_LONG` | 400 | «Название сессии не должно превышать 100 символов.» |
| `HOST_NAME_REQUIRED` | 400 | «Не заполнено имя ведущего.» |
| `HOST_NAME_TOO_LONG` | 400 | «Имя ведущего не должно превышать 100 символов.» |
| `VALIDATION_ERROR` | 400 | «Проверьте корректность данных.» |
| `INTERNAL_ERROR` | 500 | «Не удалось создать сессию. Попробуйте ещё раз.» |

## 5. Получение сессии

```http
GET /api/v1/sessions/{sessionId}
```

**Назначение:** восстановление состояния после обновления страницы.

**Успешный ответ — `200 OK`:**

```json
{
  "id": "3f1c9d24-8e4b-4b7f-9a2e-8d6f0b1c1234",
  "name": "Спринт 1: 21.10.2026",
  "hostName": "Иванов Иван",
  "status": "TASKS_IMPORTED",
  "createdAt": "2026-10-21T12:05:00Z",
  "updatedAt": "2026-10-21T12:10:00Z"
}
```

**Ошибки:**

| Код | HTTP | Текст |
|---|---:|---|
| `SESSION_NOT_FOUND` | 404 | «Сессия не найдена или завершена.» |
| `INVALID_SESSION_ID` | 400 | «Некорректный идентификатор сессии.» |
| `INTERNAL_ERROR` | 500 | «Не удалось загрузить сессию.» |

## 6. Предпросмотр импорта CSV

```http
POST /api/v1/sessions/{sessionId}/csv/preview
```

**Назначение:** S1-02 — разбор CSV без изменения очереди.

**Запрос:**

```http
Content-Type: multipart/form-data
```

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| `file` | `file` | Да | CSV-файл |

**Успешный ответ — `200 OK`:**

```json
{
  "totalRows": 3,
  "validTasks": 2,
  "invalidRows": 1,
  "tasks": [
    {
      "rowNumber": 2,
      "externalId": "TASK-1111",
      "title": "Добавить фильтр заказов",
      "description": "Реализовать фильтрацию списка заказов по статусу.",
      "youtrackUrl": "[https://youtrack.example/issue/TASK-1111](https://youtrack.example/issue/TASK-1111)",
      "priority": "High",
      "components": ["API", "WEB"],
      "affectedScenarios": ["ЗАКАЗ"],
      "notes": "Учесть права доступа.",
      "tags": ["backend", "ux"],
      "initialStoryPoints": null,
      "isValid": true,
      "errors": []
    }
  ],
  "errors": [
    {
      "rowNumber": 4,
      "field": "Компоненты",
      "value": "BACKEND",
      "code": "INVALID_COMPONENT",
      "message": "Строка 4: указан недопустимый компонент „BACKEND“."
    }
  ]
}
```

**Ошибки уровня файла:**

| Код | HTTP | Текст |
|---|---:|---|
| `FILE_REQUIRED` | 400 | «Выберите CSV-файл.» |
| `FILE_EMPTY` | 400 | «Файл пустой или не содержит задач.» |
| `FILE_INVALID_FORMAT` | 400 | «Не удалось распарсить файл. Проверьте формат.» |
| `FILE_INVALID_ENCODING` | 400 | «Файл должен быть в кодировке UTF-8.» |
| `MISSING_REQUIRED_COLUMN` | 400 | «Отсутствует обязательная колонка: {название}.» |
| `TOO_MANY_TASKS` | 400 | «В файле слишком много задач.» |
| `FILE_TOO_LARGE` | 400 | «Размер файла превышает допустимый лимит.» |
| `SESSION_NOT_FOUND` | 404 | «Сессия не найдена или завершена.» |
| `INTERNAL_ERROR` | 500 | «Не удалось обработать файл.» |

## 7. Подтверждённый импорт CSV

```http
POST /api/v1/sessions/{sessionId}/csv/import
```

**Назначение:** S1-02 — сохранение подтверждённых корректных задач.

**Запрос:**

```json
{
  "taskExternalIds": [
    "TASK-1111",
    "TASK-3333"
  ]
}
```

**Успешный ответ — `201 Created`:**

```json
{
  "importedCount": 2,
  "tasks": [
    {
      "id": "8b2f6a91-4c3d-4f18-9d1e-7a5b2c9e4321",
      "externalId": "TASK-1111",
      "title": "Добавить фильтр заказов",
      "status": "PENDING",
      "position": 1
    },
    {
      "id": "1d7e9c02-5a8b-4f61-8b3c-2e9d7a1f5678",
      "externalId": "TASK-3333",
      "title": "Исправить ввод даты",
      "status": "PENDING",
      "position": 2
    }
  ]
}
```

**Ошибки:**

| Код | HTTP | Текст |
|---|---:|---|
| `NO_TASKS_SELECTED` | 400 | «Выберите хотя бы одну задачу для импорта.» |
| `TASK_NOT_FOUND_IN_PREVIEW` | 400 | «Некоторые задачи не найдены в результатах предпросмотра.» |
| `TASK_ALREADY_IMPORTED` | 409 | «Задача уже импортирована в сессию.» |
| `SESSION_NOT_FOUND` | 404 | «Сессия не найдена или завершена.» |
| `INTERNAL_ERROR` | 500 | «Не удалось импортировать задачи.» |

## 8. Получение очереди задач

```http
GET /api/v1/sessions/{sessionId}/tasks
```

**Назначение:** S1-03 — просмотр очереди импортированных задач.

**Успешный ответ — `200 OK`:**

```json
{
  "tasks": [
    {
      "id": "8b2f6a91-4c3d-4f18-9d1e-7a5b2c9e4321",
      "externalId": "TASK-1111",
      "title": "Добавить фильтр заказов",
      "priority": "High",
      "status": "PENDING",
      "position": 1
    },
    {
      "id": "1d7e9c02-5a8b-4f61-8b3c-2e9d7a1f5678",
      "externalId": "TASK-3333",
      "title": "Исправить ввод даты",
      "priority": "Medium",
      "status": "ESTIMATED",
      "position": 2
    }
  ]
}
```

**Ошибки:**

| Код | HTTP | Текст |
|---|---:|---|
| `SESSION_NOT_FOUND` | 404 | «Сессия не найдена или завершена.» |
| `INTERNAL_ERROR` | 500 | «Не удалось загрузить очередь задач.» |

## 9. Получение текущей задачи

```http
GET /api/v1/sessions/{sessionId}/tasks/current
```

**Назначение:** S1-04 — отображение текущей задачи для оценки.

**Успешный ответ — `200 OK`:**

```json
{
  "task": {
    "id": "8b2f6a91-4c3d-4f18-9d1e-7a5b2c9e4321",
    "externalId": "TASK-1111",
    "title": "Добавить фильтр заказов",
    "description": "Реализовать фильтрацию списка заказов по статусу.",
    "youtrackUrl": "[https://youtrack.example/issue/TASK-1111](https://youtrack.example/issue/TASK-1111)",
    "priority": "High",
    "components": ["API", "WEB"],
    "affectedScenarios": ["ЗАКАЗ"],
    "notes": "Учесть права доступа.",
    "tags": ["backend", "ux"],
    "initialStoryPoints": null,
    "finalStoryPoints": null,
    "status": "ESTIMATING",
    "position": 1
  },
  "progress": {
    "current": 1,
    "total": 2
  }
}
```

**Ошибки:**

| Код | HTTP | Текст |
|---|---:|---|
| `NO_PENDING_TASKS` | 404 | «Нет задач для оценки.» |
| `SESSION_NOT_FOUND` | 404 | «Сессия не найдена или завершена.» |
| `INTERNAL_ERROR` | 500 | «Не удалось загрузить текущую задачу.» |

## 10. Фиксация итогового SP

```http
POST /api/v1/sessions/{sessionId}/tasks/{taskId}/estimate
```

**Назначение:** S1-04 и S1-05 — выбор и подтверждение итогового SP.

**Запрос:**

```json
{
  "storyPoints": 5
}
```

**Успешный ответ — `200 OK`:**

```json
{
  "task": {
    "id": "8b2f6a91-4c3d-4f18-9d1e-7a5b2c9e4321",
    "externalId": "TASK-1111",
    "title": "Добавить фильтр заказов",
    "status": "ESTIMATED",
    "finalStoryPoints": 5,
    "position": 1
  },
  "nextTaskId": "1d7e9c02-5a8b-4f61-8b3c-2e9d7a1f5678",
  "hasNextTask": true
}
```

**Ошибки:**

| Код | HTTP | Текст |
|---|---:|---|
| `STORY_POINTS_REQUIRED` | 400 | «Выберите значение Story Points.» |
| `INVALID_STORY_POINTS` | 400 | «Значение Story Points должно быть из шкалы оценки.» |
| `TASK_NOT_FOUND` | 404 | «Задача не найдена.» |
| `TASK_ALREADY_ESTIMATED` | 409 | «Задача уже оценена.» |
| `SESSION_NOT_FOUND` | 404 | «Сессия не найдена или завершена.» |
| `INTERNAL_ERROR` | 500 | «Не удалось сохранить оценку.» |

## 11. Получение итогов сессии

```http
GET /api/v1/sessions/{sessionId}/summary
```

**Назначение:** S1-06 — отображение итогов после оценки задач.

**Успешный ответ — `200 OK`:**

```json
{
  "sessionId": "3f1c9d24-8e4b-4b7f-9a2e-8d6f0b1c1234",
  "sessionName": "Спринт 1: 21.10.2026",
  "status": "COMPLETED",
  "totalTasks": 2,
  "estimatedTasks": 2,
  "totalStoryPoints": 8,
  "tasks": [
    {
      "id": "8b2f6a91-4c3d-4f18-9d1e-7a5b2c9e4321",
      "externalId": "TASK-1111",
      "title": "Добавить фильтр заказов",
      "youtrackUrl": "[https://youtrack.example/issue/TASK-1111](https://youtrack.example/issue/TASK-1111)",
      "status": "ESTIMATED",
      "finalStoryPoints": 5
    },
    {
      "id": "1d7e9c02-5a8b-4f61-8b3c-2e9d7a1f5678",
      "externalId": "TASK-3333",
      "title": "Исправить ввод даты",
      "youtrackUrl": "[https://youtrack.example/issue/TASK-3333](https://youtrack.example/issue/TASK-3333)",
      "status": "ESTIMATED",
      "finalStoryPoints": 3
    }
  ]
}
```

**Ошибки:**

| Код | HTTP | Текст |
|---|---:|---|
| `SESSION_NOT_FOUND` | 404 | «Сессия не найдена или завершена.» |
| `SESSION_NOT_COMPLETED` | 409 | «Сессия ещё не завершена.» |
| `INTERNAL_ERROR` | 500 | «Не удалось загрузить итоги сессии.» |

## 12. Общие правила ошибок

Формат ошибки:

```json
{
  "code": "INVALID_STORY_POINTS",
  "message": "Значение Story Points должно быть из шкалы оценки.",
  "details": [
    {
      "field": "storyPoints",
      "value": 7
    }
  ]
}
```

| HTTP | Значение |
|---:|---|
| `400` | Некорректный запрос или данные |
| `404` | Сессия, задача или ресурс не найдены |
| `409` | Конфликт состояния: повторное действие или недопустимый переход |
| `413` | Файл слишком большой |
| `415` | Неподдерживаемый тип файла |
| `500` | Внутренняя ошибка сервера |

## 13. Восстановление состояния

После обновления страницы Host выполняет:

1. `GET /api/v1/sessions/{sessionId}` — получить сессию.
2. `GET /api/v1/sessions/{sessionId}/tasks` — получить очередь.
3. В зависимости от статуса:
   - `CREATED` или `TASKS_IMPORTED` → открыть лобби;
   - `IN_PROGRESS` → `GET /tasks/current` и открыть экран оценки;
   - `COMPLETED` → `GET /summary` и открыть итоги.

Выбор SP до подтверждения не восстанавливается, поскольку является локальным состоянием UI.

## 14. Критерии приёмки API

- [ ] Согласован перечень API-операций для S1-01–S1-06.
- [ ] Для каждого эндпоинта описаны метод, путь, запрос, ответ и назначение.
- [ ] Для каждого эндпоинта описаны успешные и ошибочные сценарии.
- [ ] Определён единый формат ошибок.
- [ ] Согласованы коды и тексты ошибок с UI.
- [ ] Проверено восстановление состояния после обновления страницы.
- [ ] API-контракт соответствует модели данных.
- [ ] Контракт принят и передан в разработку.