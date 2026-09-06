# Спецификация SDK для znt-core

Документ описывает фактический публичный контракт текущей реализации `znt-core`
для сторонних SDK на TypeScript, Python, Go, C#, Rust и других языках.

---

## 1. Транспорт и подключение

`znt-core` работает как пользовательский фоновый daemon и принимает
newline-delimited запросы JSON-RPC 2.0 через IPC:

- Linux/macOS: Unix Domain Socket;
- Windows: Named Pipe.

Daemon обслуживает один активный проект. Новый `scan` с другим путём переключает
его на этот проект. Одновременно может выполняться только один `scan`.

### Пути по умолчанию

| ОС | Транспорт | Endpoint |
|---|---|---|
| Linux/macOS | Unix Domain Socket (`unix`) | `~/.znt/znt.sock` |
| Windows | Named Pipe (`npipe`) | `\\.\pipe\znt-core` |

Переменные окружения:

- `ZNT_ENDPOINT` переопределяет endpoint daemon;
- `ZNT_RUNTIME_DIR` переопределяет каталог process-wide файлов, включая
  `znt.log`.

Точный приоритет вычисления endpoint:

1. endpoint, явно переданный процессу (`daemon --socket`) или SDK-клиенту;
2. непустой `ZNT_ENDPOINT`;
3. платформенный endpoint по умолчанию из таблицы выше.

`ZNT_RUNTIME_DIR` сам по себе endpoint не меняет: в текущей реализации он
определяет каталог process-wide файлов, прежде всего `znt.log`. Для совместного
использования нестандартного socket/pipe daemon и SDK должны получить одинаковый
`ZNT_ENDPOINT` либо одинаковое явное значение endpoint.

По умолчанию append-лог находится в `~/.znt/znt.log` на Unix и в каталоге
пользователя `.znt\znt.log` на Windows. Настройки отключения файлового лога нет.

Проектные данные хранятся отдельно, в `<workspace>/.znt/znt.db`.

CLI-флаг `daemon --config <path>` загружает конфигурацию из указанного файла.
Флаг `scan --config <path>` передаёт тот же путь автоматически запускаемому
daemon. Если daemon уже работает с другим конфигом, CLI отклоняет scan вместо
молчаливого игнорирования пути. Без флага используется `config.yaml` из рабочей
директории запуска daemon.
Явно указанный config должен существовать и корректно разбираться; иначе daemon
завершает запуск с ошибкой. Для пути по умолчанию сохраняется прежний fallback
на встроенные значения.

SDK рекомендуется всегда передавать абсолютный путь в `scan.file_path`.
Относительный путь разрешается относительно рабочей директории процесса daemon,
которая может отличаться от текущей директории SDK-клиента.

---

## 2. Формат кадров и JSON-RPC

- Кодировка сообщений: UTF-8.
- Каждый запрос и ответ завершается `\n`.
- Максимальный размер одной строки запроса в текущей реализации: 10 MiB.
- `params` должен быть JSON-объектом. Его также можно не передавать или передать
  как `null`, что эквивалентно `{}`.

Одно IPC-соединение допускает несколько последовательных newline-delimited
запросов. Текущий сервер обрабатывает кадры одного соединения последовательно,
однако SDK должен сопоставлять ответы с запросами по `id`, а не полагаться на
порядок ответов: это сохраняет корректность при нескольких соединениях и
будущей параллельной обработке.

JSON-RPC batch-массивы не поддерживаются. JSON-RPC notifications без `id` также
не являются публичным контрактом: daemon формирует ответ на каждый принятый
запрос, поэтому SDK должен всегда передавать `id`.

Лимит 10 MiB относится к строке запроса без завершающего `\n`. Отдельный
публичный лимит размера ответа сейчас не установлен; SDK должен буферизовать
ответ до получения `\n`.

### Запрос

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "znatok_semantic_search",
  "params": {
    "query": "SearchWithOptions",
    "mode": "lexical"
  }
}
```

`jsonrpc` должен быть равен `"2.0"`, а `method` не должен быть пустым. SDK
следует использовать уникальный числовой или строковый `id` для каждого
ожидающего ответа.

### Успешный ответ

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {}
}
```

### Ответ с ошибкой

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32009,
    "message": "Target symbol 'Search' is ambiguous",
    "data": {
      "target": "Search",
      "candidates": ["pkg.First.Search", "pkg.Second.Search"]
    }
  }
}
```

При ошибке поле `result` отсутствует. `data` опционально и содержит
машинно-читаемый контекст.

### Коды ошибок

| Код | Имя | Фактическое применение |
|---|---|---|
| `-32700` | Parse error | Входная строка не декодируется в объект запроса `Request`, включая malformed JSON и неподходящую top-level JSON-форму |
| `-32600` | Invalid request | После декодирования отсутствует `jsonrpc: "2.0"` или непустой `method` |
| `-32601` | Method not found | Метод не поддерживается либо `shutdown` недоступен у конкретного handler |
| `-32602` | Invalid params | `params` не является объектом либо сработала явная проверка параметров метода |
| `-32603` | Internal error | Непредвиденная ошибка, в том числе недоступный до первого scan graph/semantic service |
| `-32001` | Busy | Уже выполняется scan либо запрошен `shutdown` во время scan |
| `-32004` | Not found | Workspace, файл или символ не найден |
| `-32009` | Ambiguous | Имя символа соответствует нескольким кандидатам; кандидаты находятся в `error.data.candidates` |

SDK должно сохранять `code`, `message` и `data` в типизированной ошибке.
Автоматически повторять запрос имеет смысл только для `-32001`; остальные
ошибки требуют проверки запроса или состояния daemon.

### Совместимость типов параметров

Внутренний compatibility-адаптер большинства методов преобразует параметры в
строки:

- JSON string остаётся строкой;
- boolean преобразуется в `"true"` или `"false"`;
- JSON number преобразуется в `int`, дробная часть отбрасывается;
- `null` пропускается.

Исключения со строгой проверкой на JSON-RPC-входе:

- `semantic_search.mode` должен быть строкой и одним из `hybrid`, `lexical`,
  `vector`;
- `subgraph.depth` должен быть целым JSON-числом `>= 1`;
- cursor-параметры `logs.stream_id` и `logs.after_id` должны передаваться
  вместе; `stream_id` должен быть непустой строкой, а `after_id` — целым
  JSON-числом в диапазоне `0..9007199254740991`.

Несмотря на compatibility-преобразование, SDK следует отправлять параметры в
типах, указанных ниже.

---

## 3. Алиасы параметров

Алиасы зависят от метода; единого набора алиасов для всех методов нет.

| Метод | Канонический параметр | Поддерживаемые алиасы |
|---|---|---|
| `scan` | `file_path` | `path` |
| `scan` | `language` | `lang` |
| `semantic_search` | `query` | `q` |
| `semantic_search` | `type` | `type_boost` |
| `semantic_search` | `role` | `role_boost` |
| `semantic_search` | `file_path` | `file_pattern`, `path` |
| `find_similar` | `target` | `name`, `symbol` |
| `find_similar` | `query` | `q` |
| `find_similar` | `type` | `type_boost` |
| `find_similar` | `role` | `role_boost` |
| `find_similar` | `file_path` | `file_pattern`, `path` |
| `file_outline` | `file_path` | `path`, `file`, `filepath` |

У параметров `subgraph.from`, `subgraph.to`, `depth`, `edge_types` и `format`
алиасов нет.

У параметров `logs.stream_id` и `logs.after_id` алиасов нет.

---

## 4. Методы и DTO

### 1. `info`

Возвращает версию протокола, версию сервера, канонический список JSON-RPC методов
и capabilities. Метод не создаёт сессию, не изменяет состояние daemon и не
проверяет совместимость клиента — это ответственность SDK.

- Params DTO: `{}` или не передавать.
- Result DTO:

```json
{
  "protocol_version": "1.0.0",
  "server_version": "1.0.0",
  "config": "/absolute/path/to/config.yaml",
  "methods": [
    "info",
    "status",
    "logs",
    "scan",
    "scan_status",
    "znatok_semantic_search",
    "znatok_find_similar",
    "znatok_get_subgraph",
    "znatok_file_outline",
    "shutdown"
  ],
  "languages": [
    {"name": "auto", "extensions": [], "has_config": false},
    {"name": "csharp", "extensions": [".cs"], "has_config": true},
    {"name": "go", "extensions": [".go"], "has_config": true},
    {"name": "java", "extensions": [".java"], "has_config": true},
    {"name": "javascript", "extensions": [".cjs", ".js", ".jsx", ".mjs", ".ts", ".tsx", ".vue"], "has_config": true},
    {"name": "php", "extensions": [".php"], "has_config": true},
    {"name": "python", "extensions": [".py"], "has_config": true},
    {"name": "resource", "extensions": [".dockerfile", ".env", ".json", ".markdown", ".md", ".sql", ".toml", ".xml", ".yaml", ".yml", "Dockerfile", "Makefile"], "has_config": true}
  ],
  "capabilities": {
    "semantic_search": {
      "modes": ["hybrid", "lexical", "vector"],
      "supports_type_filter": true,
      "supports_role_filter": true,
      "supports_compact": true,
      "supports_include_code": true
    },
    "find_similar": {
      "supports_target": true,
      "supports_query": true,
      "edge_types": ["contains", "call", "inherits", "implements"]
    },
    "subgraph": {
      "formats": ["text", "json", "mermaid"],
      "supports_tracing": true
    },
    "file_outline": {
      "supports_code": true
    },
    "scan": {
      "async": true,
      "supports_scan_id": true,
      "supports_scan_status": true,
      "progress_phases": [
        "ast",
        "building_graph",
        "persisting_ast",
        "extracting",
        "tagging",
        "embedding",
        "indexing",
        "completed",
        "error"
      ]
    },
    "parsers": ["csharp", "go", "java", "javascript", "php", "python", "resource"],
    "transports": ["unix_socket", "named_pipe"]
  }
}
```

`methods` содержит только канонические имена: псевдонимы в массив не включаются.
`config` содержит абсолютный путь к конфигурационному файлу активного daemon.
`languages` является аналогом старого `/api/languages`: `auto` всегда находится
первым, после него зарегистрированные парсеры перечисляются по имени. `extensions`
содержит поддерживаемые расширения файлов, а `has_config` сообщает, существует ли
секция языка в активной конфигурации daemon.
Список `parsers` формируется динамически из парсеров, зарегистрированных в
конкретном экземпляре Engine. В стандартном daemon используются значения из
примера.

Названия capabilities `supports_type_filter` и `supports_role_filter` не
отражают точную семантику `semantic_search`: эти параметры фактически являются
ranking boosts, а не строгими фильтрами.

---

### 2. `status`

Возвращает статистику активной базы `<workspace>/.znt/znt.db`.

- Params DTO: `{}` или не передавать.
- Result DTO:

```json
{
  "declarations": 756,
  "dependencies": 4,
  "edges": 4113,
  "files": 51,
  "functions": 609,
  "members": 459,
  "status": "Monitoring",
  "project_root": "/absolute/path/to/workspace"
}
```

`status` принимает значения `Idle`, `Scanning`, `Monitoring`, `Error`.
До выбора проекта счётчики равны нулю, а `project_root` — пустая строка.

---

### 3. `scan`

Асинхронно запускает индексацию и сразу возвращает `scan_id`.

- Params DTO:

```json
{
  "file_path": "/absolute/path/to/workspace",
  "language": "auto",
  "restart": false
}
```

Параметры:

- `file_path`: string, опционально, значение по умолчанию `"."`; алиас `path`;
- `language`: string, опционально, значение по умолчанию `"auto"`; алиас `lang`;
- `restart`: boolean, опционально, `false` по умолчанию. При `true` daemon
  закрывает активные сервисы проекта, удаляет только `<workspace>/.znt` и
  выполняет полное сканирование с созданием новой базы. Операция отклоняется,
  если активный config находится внутри удаляемой директории `.znt`.

Стандартный daemon поддерживает `auto`, `csharp`, `go`, `java`, `javascript`,
`php`, `python`, `resource`. Неизвестный язык возвращает `-32602`.

`file_path` должен указывать на существующую директорию. Отсутствующий путь
возвращает `-32004`, путь к файлу — `-32602`.

- Result DTO:

```json
{
  "scan_id": "scan_1787930069069181000",
  "status": "scanning",
  "message": "Scan initiated",
  "path": "/absolute/path/to/workspace",
  "language": "auto"
}
```

`path` в ответе всегда нормализован в абсолютный путь.

---

### 4. `scan_status` (псевдоним: `znatok_scan_status`)

Возвращает текущий `ScanProgress`.

- Params DTO: `{}`.
- Метод не принимает `scan_id`: состояние является глобальным для daemon и
  описывает текущий либо последний scan. Если несколько клиентов координируют
  scan, SDK должен сравнивать возвращаемый `scan_id` с идентификатором,
  полученным от собственного вызова `scan`.
- Result DTO:

```json
{
  "scan_id": "scan_1787930069069181000",
  "status": "scanning",
  "path": "/absolute/path/to/workspace",
  "language": "auto",
  "scanned_files": 24,
  "total_files": 50,
  "current_file": "service.go",
  "percent": 48,
  "phase": "ast",
  "phase_done": 24,
  "phase_total": 50,
  "started_at": "2026-08-28T18:14:00Z"
}
```

Значения `status`: `idle`, `scanning`, `completed`, `error`.

Значения `phase`: `idle`, `ast`, `building_graph`, `persisting_ast`,
`extracting`, `tagging`, `embedding`, `indexing`, `completed`, `error`.

Поля `current_file`, `started_at`, `completed_at`, `last_error` имеют
`omitempty` и отсутствуют в JSON, когда пусты.

После AST-разбора идут `building_graph` и `persisting_ast`, затем `extracting`.
При включённых embeddings семантическая перестройка отображается фазой
`embedding`, при отключённых — `tagging`; затем идут `indexing` и `completed`.
Поле `percent` монотонно распределено между фазами.

---

### 5. `znatok_semantic_search` (псевдоним: `semantic_search`)

Выполняет лексический, векторный или гибридный поиск.

- Params DTO:

```json
{
  "query": "GetPendingSemanticData",
  "mode": "hybrid",
  "limit": 10,
  "type": "function",
  "role": "service",
  "file_path": "pkg/semantic/*.go",
  "compact": false,
  "include_code": true,
  "max_code_lines": 30,
  "callers_level": 1,
  "callees_level": 1
}
```

Параметры:

- `query`: string, обязательный; алиас `q`;
- `mode`: string, `hybrid` по умолчанию; допустимы `hybrid`, `lexical`,
  `vector`; неизвестное значение или нестроковый JSON-тип возвращает `-32602`;
- `limit`: положительное целое, `10` по умолчанию;
- `type`: string, ranking boost по совпадению AST-типа; алиас `type_boost`;
- `role`: string, ranking boost по совпадению роли; алиас `role_boost`;
- `file_path`: string, фильтрующая маска пути; алиасы `file_pattern`, `path`;
- `compact`: boolean, `false` по умолчанию; при `true` связи не строятся;
- `include_code`: boolean, `false` по умолчанию;
- `max_code_lines`: положительное целое, `30` по умолчанию;
- `callers_level`: неотрицательное целое, `0` по умолчанию;
- `callees_level`: неотрицательное целое, `0` по умолчанию.

`type` и `role` не исключают несовпадающие элементы, а добавляют совпавшим
результатам boost `+4.0`. Типичные AST-типы: `declaration`, `function`, `member`,
`dependency`, `namespace`. Генерируемые роли включают `controller`, `repository`,
`service`, `model`, `utility`, `contract`, `factory`, `adapter`, `handler`,
`entrypoint`, `state`, `dependency`, `callsite`, `test`, `generic`.

Маска `file_path` является фильтром. Если она не начинается с `**/`, такой
префикс проверяется автоматически. Например, `pkg/parser/ids.go` сопоставляется
как `**/pkg/parser/ids.go`.

В режиме `lexical`, если лексический поиск не дал результатов, реализация
пытается выполнить vector fallback.

- Result DTO: `SearchResultItem[] | null`.

```json
[
  {
    "id": "pkg/semantic/store.go::SemanticStore.SearchFTS5",
    "name": "SemanticStore.SearchFTS5",
    "type": "function",
    "role": "repository",
    "file": "pkg/semantic/store.go",
    "start_line": 893,
    "end_line": 945,
    "score": 9.5,
    "summary": "type:function name:SemanticStore.SearchFTS5...",
    "description": "SearchFTS5 performs lexical search...",
    "code": "func (s *SemanticStore) SearchFTS5(...) ...",
    "callers": [
      {
        "name": "SemanticService.SearchWithOptions",
        "type": "function",
        "file": "pkg/semantic/service.go",
        "start_line": 1202,
        "end_line": 1343,
        "edge_type": "call"
      }
    ]
  }
]
```

Опциональные поля с `omitempty` отсутствуют, когда пусты. При валидном запросе
без результатов текущая реализация сериализует nil slice как `null`; SDK может
нормализовать его в `[]`.

---

### 6. `znatok_find_similar` (псевдоним: `find_similar`)

Ищет похожие реализации по символу, текстовому запросу или их сочетанию.

- Params DTO:

```json
{
  "target": "SemanticService.SearchWithOptions",
  "query": "hybrid semantic search",
  "limit": 10,
  "type": "function",
  "role": "service",
  "file_path": "pkg/semantic/*.go",
  "edge_types": "contains,call",
  "include_code": true,
  "max_code_lines": 30
}
```

Параметры:

- должен быть передан хотя бы один из `target` или `query`;
- `target`: string; алиасы `name`, `symbol`;
- `query`: string; алиас `q`;
- `limit`: положительное целое, `10` по умолчанию;
- `type`: string, фильтр кандидатов по частичному совпадению типа; алиас
  `type_boost`;
- `role`: string, фильтр кандидатов по частичному совпадению роли; алиас
  `role_boost`;
- `file_path`: string, маска пути; алиасы `file_pattern`, `path`;
- `edge_types`: comma-separated string из `contains`, `call`, `inherits`,
  `implements`;
- `include_code`: boolean, `false` по умолчанию;
- `max_code_lines`: положительное целое, `30` по умолчанию.

`find_similar` зависит от embeddings. При поиске по `query` сервис должен
успешно построить embedding запроса; при поиске по `target` у найденного символа
должен быть сохранён embedding. Если исходный embedding получить не удалось или
он отсутствует, текущая реализация возвращает пустой массив `[]` без ошибки.

- Result DTO: `SimilarResultItem[]`.

```json
[
  {
    "name": "SemanticStore.SearchFTS5",
    "type": "function",
    "file": "pkg/semantic/store.go",
    "start_line": 893,
    "end_line": 945,
    "score": 0.82,
    "similarity_reason": "82% calibrated semantic similarity",
    "code": "func (s *SemanticStore) SearchFTS5(...) ...",
    "relations": {
      "callers_count": 2,
      "callees_count": 3,
      "internal_callees_count": 2,
      "external_callees_count": 1
    }
  }
]
```

Если аналогов нет, возвращается `[]`. Если `target` не найден или неоднозначен,
возвращается `-32004` или `-32009` соответственно.

---

### 7. `znatok_get_subgraph` (псевдоним: `subgraph`)

Поддерживает два режима:

1. `from` без `to`: BFS-окружение начального символа до `depth`;
2. `from` вместе с `to`: поиск пути между символами.

- Params DTO:

```json
{
  "from": "SemanticService.SearchWithOptions",
  "to": "SemanticStore.SearchFTS5",
  "depth": 2,
  "edge_types": "call,contains",
  "format": "json"
}
```

Параметры:

- `from`: string, обязательный начальный символ, алиасов нет;
- `to`: string, опциональный конечный символ, алиасов нет;
- `depth`: целое JSON-число `>= 1`, значение по умолчанию `1`; строка вроде
  `"oops"`, дробное или неположительное число возвращает `-32602`;
- `edge_types`: comma-separated string из `contains`, `call`, `inherits`,
  `implements`;
- `format`: string, `text` по умолчанию. Значение обрезается по краям и
  приводится к нижнему регистру перед проверкой. Допустимы `text`, `json`,
  `mermaid`; любой другой формат возвращает `-32602 Invalid params`.

Если `edge_types` не указан, обход использует `call`, `implements`, `inherits`.

При режиме `from + to` найденный путь возвращается полностью. Если он длиннее
`depth`, результат содержит warning о том, что ограничение глубины было
проигнорировано.

Пустой или отсутствующий `from` возвращает `-32602`. Параметр `target` также
явно отклоняется с `-32602`; `symbol` не поддерживается.

- Базовый Result DTO:

```json
{
  "from": "SemanticService.SearchWithOptions",
  "to": "SemanticStore.SearchFTS5",
  "depth": 2,
  "format": "json"
}
```

Пустые `to`, `warning`, `status` опускаются.

Для `format: "text"` добавляется поле `text`.

Для `format: "json"` добавляются:

```json
{
  "nodes": [
    {
      "name": "SemanticService.SearchWithOptions",
      "type": "function",
      "file": "pkg/semantic/service.go",
      "start_line": 1202,
      "end_line": 1343
    }
  ],
  "edges": [
    {
      "from": "SemanticService.SearchWithOptions",
      "to": "SemanticStore.SearchFTS5",
      "type": "call"
    }
  ]
}
```

Для `format: "mermaid"` добавляется строковое поле `mermaid`; поля `nodes` и
`edges` в этом формате отсутствуют.

Не найденный символ возвращает `-32004`, неоднозначный — `-32009`.

---

### 8. `znatok_file_outline` (псевдоним: `file_outline`)

Возвращает текстовое оглавление проиндексированного файла.

- Params DTO:

```json
{
  "file_path": "main.go",
  "include_code": false
}
```

Параметры:

- `file_path`: string, обязательный; алиасы `path`, `file`, `filepath`;
- `include_code`: boolean, `false` по умолчанию; при `true` snippets включаются
  в `formatted_text`.

Текущий публичный handler всегда использует `Compact: true`,
`TopLevelOnly: true`, `Format: "text"`. Узлы типа `member` исключаются. Поле
`symbols` внутреннего `FileOutlineResult` очищается перед ответом.

- Result DTO:

```json
{
  "file": "main.go",
  "total_symbols": 12,
  "formatted_text": "[file_outline] main.go (12 symbols)\n..."
}
```

Не найденный файл возвращает `-32004`; в `error.data` передаётся `file_path` и,
если найден близкий путь, `suggestion`.

---

### 9. `logs`

Возвращает начальный снимок либо новые записи кольцевого буфера логов daemon
после переданного cursor.

- Для первого запроса Params DTO: `{}` или не передавать.
- Для последующих запросов `stream_id` и `after_id` должны передаваться вместе:

```json
{
  "stream_id": "stream_1234_1788500000000000000",
  "after_id": 42
}
```

- `stream_id`: непустая строка из предыдущего ответа;
- `after_id`: неотрицательное целое JSON-число в безопасном диапазоне
  JavaScript (`0..9007199254740991`), обычно значение предыдущего
  `next_cursor`;
- наличие только одного параметра, неправильный тип, дробное, отрицательное или
  слишком большое значение возвращает `-32602 Invalid params`.

Result DTO:

```json
{
  "stream_id": "stream_1234_1788500000000000000",
  "entries": [
    {
      "id": 43,
      "message": "Initial scan complete",
      "level": "success",
      "time": "14:32:08"
    }
  ],
  "next_cursor": 43,
  "truncated": false
}
```

- `stream_id` создаётся при запуске Engine и меняется после перезапуска daemon;
- `entries` содержит не более 100 записей в хронологическом порядке;
- `id` монотонно увеличивается внутри одного `stream_id`, начиная с `1`;
- `next_cursor` — ID последней созданной записи текущего потока либо `0`, если
  записей ещё не было; его следует передать как следующий `after_id`;
- начальный запрос без cursor возвращает весь текущий буфер;
- при совпадающем `stream_id` возвращаются только записи с `id > after_id`;
- если `stream_id` не совпадает, `after_id` находится впереди текущего потока
  либо нужные записи уже вытеснены из 100-элементного буфера, возвращается весь
  доступный буфер с `truncated: true`; клиент должен заменить локальный снимок;
- если новых записей нет, `entries` равен `[]`, `next_cursor` не меняется и
  `truncated` равен `false`.

Буфер хранится только в памяти и очищается при перезапуске daemon. Метод не
читает постоянный файл `znt.log` и не является подпиской на новые события.
`time` содержит локальное время процесса daemon в формате `HH:mm:ss`, без даты
и часового пояса. `level` является строкой; текущая реализация обычно использует
`info`, `parse`, `success`, `warning` и `error`, но SDK не должен ограничивать
поле этим списком.

---

### 10. `shutdown` (псевдоним: `stop`)

- Params DTO: `{}`.
- Во время активного scan возвращает `-32001 Busy`.
- После scan возвращает `{"status":"stopping"}` и закрывает глобальный
  endpoint.
- Handler без настроенного shutdown callback возвращает `-32601`.

---

## 5. Рекомендации для SDK

1. Генерируйте уникальный числовой или строковый `id` для каждого запроса.
2. Храните ожидающие запросы в `Map<id, resolve/reject/timeout>` или эквиваленте.
3. Буферизуйте входной поток и разбивайте его по `\n`.
4. При наличии `error` отклоняйте запрос типизированной ошибкой с сохранением
   `code`, `message`, `data`.
5. Нормализуйте `semantic_search` result `null` в пустой массив, если SDK
   предоставляет массивный API.
6. Передавайте абсолютный `scan.file_path` и документированные JSON-типы, не
   полагаясь на compatibility-преобразование ядра.
7. Перед вызовами search/subgraph/file_outline выполните `scan` и дождитесь
   `scan_status.status == "completed"`.
8. Таймауты запросов и polling interval для `scan_status` являются политикой
   SDK, а не частью server DTO. При локальном timeout SDK не должен считать,
   что сервер отменил уже отправленную операцию.
9. После успешного `shutdown` SDK должен считать соединение закрывающимся и не
   повторять запрос автоматически: повтор может остановить уже другой daemon,
   запущенный на том же endpoint.

---

## 6. Управление установленным core в Node.js SDK

Следующие методы относятся к `znt-sdk-nodejs` и не являются JSON-RPC методами
daemon. Скачивание, запуск и удаление должны выполняться только явным вызовом;
`startCore()` не должен автоматически вызывать `downloadCore()`.

### `isCoreInstalled(expectedVersion?)`

Возвращает `Promise<boolean>`.

- Без аргумента проверяет наличие доступного для запуска executable.
- С `expectedVersion` дополнительно сравнивает версию в локальных метаданных,
  записанных `downloadCore()`.
- Для вручную скопированного executable наличие определяется, но версия считается
  неизвестной, поэтому versioned-проверка возвращает `false`.

### `downloadCore(options)`

Явно скачивает и устанавливает core. Текущий options DTO:

```ts
interface DownloadCoreOptions {
  url?: string;
  version?: string;
  sha256?: string;
  signal?: AbortSignal;
}
```

До утверждения структуры GitHub Releases URL передаётся через `options.url` или
`ZNT_CORE_DOWNLOAD_URL`. SDK скачивает во временный файл, проверяет непустой
результат и опциональную SHA-256, выставляет executable-права на Unix и только
после этого заменяет установленный файл. Рядом сохраняются метаданные версии,
URL, SHA-256 и времени установки. Метод не запускает daemon.

### `startCore()`

Если endpoint уже отвечает, переиспользует существующий daemon. Иначе проверяет
установленный executable и запускает `znt-core daemon --socket <endpoint>`.
Если binary отсутствует, возвращает `ZntCoreNotInstalledError`; скачивание не
выполняется.

### `stopCore()`

Отправляет JSON-RPC `shutdown`, затем закрывает клиентское IPC-соединение.
Ограничение `Busy` во время scan соответствует контракту `shutdown`.

### `removeCore()`

Удаляет установленный executable и его metadata-файл, возвращая, существовал ли
executable. Метод не останавливает работающий daemon; вызывающая сторона должна
сначала выполнить `stopCore()`.

### Каталоги установки по умолчанию

| ОС | Каталог |
|---|---|
| Linux | `$XDG_DATA_HOME/znt/bin` или `~/.local/share/znt/bin` |
| macOS | `~/Library/Application Support/Znt/bin` |
| Windows | `%LOCALAPPDATA%\\Znt\\bin` |

Имя executable: `znt-core` на Linux/macOS и `znt-core.exe` на Windows.
`ZNT_CORE_HOME` переопределяет каталог, а `ZNT_CORE_BINARY` — полный путь.
