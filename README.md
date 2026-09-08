# Znt Core — Standalone Code Graph & Semantic Engine

Автономное высокопроизводительное ядро системы **[Znt (Знаток)](https://github.com/boottaa/znt)** на Go для построения AST-графа кодовой базы, семантического анализа и трассировки вызовов без сторонних веб- или HTTP-зависимостей.

Работает как один пользовательский daemon только через **IPC Sockets**: Unix Domain Socket на Linux/macOS и Named Pipe на Windows. Daemon обслуживает один активный проект, а все данные проекта хранятся в единой SQLite базе `.znt/znt.db`.

---

## 🚀 Компиляция и запуск

### 1. Сборка бинарника
Для сборки требуется Go 1.20+ и C-компилятор (`gcc` / `clang`) для работы CGo и парсеров Tree-sitter:

```bash
cd znt-core
go build -o znt-core .
```

### 2. Команды CLI

| Команда | Описание |
|---------|----------|
| `./znt-core scan --path .` | Запуск/reuse daemon, сканирование проекта и дальнейший мониторинг |
| `./znt-core scan --path . --restart` | Удаление `<workspace>/.znt` и полное повторное сканирование |
| `./znt-core status` | Живой статус daemon и активного проекта через IPC |
| `./znt-core query --method <name>` | Выполнение JSON-RPC команды для активного проекта |
| `./znt-core stop` | Корректная остановка пользовательского daemon |

Для обычной работы достаточно команды `scan`: если daemon ещё не запущен, она
автоматически запустит его в фоне, выполнит индексацию и включит мониторинг
проекта.

## 📦 Публикация релиза

Production-артефакты для macOS, Linux и Windows можно собрать локально на macOS:

```bash
make build-release
```

Для публикации установите GitHub CLI, один раз выполните `gh auth login`, затем:

```bash
make release
```

Команда определяет последний тег в `znt-app/core`, создаёт следующую версию
`1.0`, `1.1`, `1.2` и так далее, отправляет содержимое `dist` в ветку `main` и
создаёт GitHub Release со всеми бинарниками, `manifest.json` и контрольными
суммами. Node.js SDK читает manifest и автоматически скачивает подходящий
бинарник из последнего релиза.

## 🛠️ Расширенное использование: ручной запуск daemon

Команда `daemon` запускает IPC-сервис в foreground, но сама не выбирает и не
сканирует проект. Сразу после запуска daemon имеет статус `Idle` и ждёт команду
`scan`. Проект начинает мониториться только после отдельного вызова `scan`.

Ручной режим полезен для:

- отладки запуска и просмотра ошибок непосредственно в терминале;
- использования собственного config или IPC endpoint;
- запуска через systemd, launchd или Windows Service;
- разработки и ручного тестирования JSON-RPC клиентов.

Запуск с настройками по умолчанию:

```bash
./znt-core daemon
```

Запуск с явным config и socket:

```bash
./znt-core daemon \
  --config /absolute/path/to/config.yaml \
  --socket /absolute/path/to/znt.sock
```

После запуска выберите проект из другого терминала:

```bash
./znt-core scan --path /absolute/path/to/workspace
```

Если используется нестандартный socket, клиентские команды должны подключаться
к тому же endpoint через `ZNT_ENDPOINT`. Пока проект не выбран через `scan`,
daemon только принимает IPC-запросы и ничего не мониторит. При закрытии
foreground-процесса daemon останавливается.

---

## 📡 Протоколы транспорта и подключения

`znt-core` предоставляет единую систему JSON-RPC 2.0 команд для взаимодействия с клиентами.

### 1. IPC Sockets (Unix Domain Sockets / Windows Named Pipes)

Первый `./znt-core scan <path>` автоматически запускает daemon в фоне. Последующие CLI-команды подключаются к глобальному endpoint независимо от текущей директории. Повторный `scan` переключает единственный daemon на новый активный проект.

#### 🐧 Linux / 🍎 macOS (Unix Domain Socket)
По умолчанию создается пользовательский сокет `~/.znt/znt.sock`.

- **Как подключиться и протестировать (через `nc` / `netcat` / `socat`)**:
  ```bash
  nc -U ~/.znt/znt.sock
  ```
  После подключения отправьте JSON-RPC запрос:
  ```json
  {"jsonrpc":"2.0","id":1,"method":"status"}
  ```

#### 🪟 Windows (Named Pipes)
По умолчанию создается именованный канал `\\.\pipe\znt-core`.

- **Как подключиться и протестировать (через PowerShell)**:
  ```powershell
  $pipe = New-Object System.IO.Pipes.NamedPipeClientStream(".", "znt-core", [System.IO.Pipes.PipeAccessRights]::ReadWrite)
  $pipe.Connect()
  $writer = New-Object System.IO.StreamWriter($pipe)
  $reader = New-Object System.IO.StreamReader($pipe)
  $writer.WriteLine('{"jsonrpc":"2.0","id":1,"method":"status"}')
  $writer.Flush()
  $reader.ReadLine()
  $pipe.Close()
  ```

## ⚙️ Конфигурация (config.yaml)

Настройки хранятся в YAML-файле `config.yaml`. При управляемой установке MCP
создаёт его рядом с бинарником после интерактивного setup. Для ручной установки
встроенный шаблон можно получить без запуска daemon:

```bash
./znt-core config defaults
./znt-core config validate --config /absolute/path/to/config.yaml
./znt-core config validate --config /absolute/path/to/config.yaml --check-provider
```

Путь можно переопределить для foreground daemon или при автоматическом запуске
daemon командой `scan`:

```bash
./znt-core daemon --config /absolute/path/to/config.yaml

./znt-core scan \
  --path /absolute/path/to/workspace \
  --config /absolute/path/to/config.yaml
```

Если daemon уже работает, `scan --config` проверяет его активный конфиг. При
другом пути команда завершится ошибкой: сначала остановите daemon через
`znt-core stop`. Без `--config` сохраняется прежнее поведение — используется
`config.yaml` из директории запуска daemon.
Явно указанный файл должен существовать и содержать корректный YAML, иначе
daemon не запускается.

Для принудительного полного сканирования передайте `--restart`. Core закроет
активный индекс, удалит только `<workspace>/.znt` и создаст его заново:

```bash
./znt-core scan --path /absolute/path/to/workspace --restart
```

Если активный config расположен внутри удаляемого `.znt`, операция отклоняется,
чтобы не удалить конфигурацию daemon.

### Пример config.yaml

```yaml
llm:
  provider: openapi             # "ollama" | "openapi" | "" (пусто если без LLM)
  url: https://openrouter.ai/api/v1
  token_ref: znt-keyring://openrouter/default
  model: qwen/qwen-2.5-7b-instruct
  embed_model: google/gemini-embedding-001
  request_timeout_minutes: 20
  retry_delays_seconds: [60, 120, 300]
  max_bytes: 25000
  max_embed_bytes: 32768

semantic:
  mode: fast                    # "fast" (0ms LLM оверхеда) или "llm"
  description_language: ru      # Язык описаний ("ru" / "en") при mode: llm
  concurrency:
    semantic_workers: 6
  throttling:
    delay_ms: 10
    adaptive: true

exclude:
  - "dist/**"
  - "sample-project/**"
  - "docs/**"
  - ".github/**"
  - ".git/**"

languages:
  go:
    include: []
    exclude:
      - "vendor/**"
    entrypoints:
      - "main.go"
      - "cmd/**/main.go"
    dependency_dirs:
      - "vendor/**"
```

### Разбор секций

#### `[llm]` — Подключение к языковым моделям
* `provider`: Провайдер LLM (`ollama` или `openapi`).
* `url`: Base URL API эндпоинта.
* `token_ref`: ссылка на API-ключ в macOS Keychain, Windows Credential Manager или Linux Secret Service.
* `token_env`: имя переменной окружения с API-ключом.
* `token`: устаревший plaintext-вариант для обратной совместимости; не рекомендуется.
* `model`: Модель генерации описаний кода.
* `embed_model`: Модель для векторного эмбеддинга.

#### `[semantic]` — Режимы семантического индекса
* `mode`:
  * `fast` (по умолчанию) — автономный сверхбыстрый режим без обращения к LLM. Описания строятся из AST-дерева и комментариев, теги генерируются через TF-IDF.
  * `llm` — семантические описания генерируются языковой моделью.
* `description_language`: Язык для формирования LLM-описаний (`ru`, `en`).

#### `[exclude]` — Исключения из сканирования
Глобальные маски папок и файлов, которые пропускаются при индексации (например, `.git/**`, `dist/**`).

---

## 🗄️ Хранение данных: Единая БД `.znt/znt.db`

В `znt-core` все данные хранятся в **одной SQLite базе `.znt/znt.db`** (в режиме WAL):

1. **AST-Таблицы (`nodes`, `edges`, `files`)**: Точная структура графа вызовов, типов, наследования и расположения строк.
2. **Семантические таблицы (`semantic_nodes`, `fts_nodes`)**: Виртуальная таблица FTS5 для лексического поиска BM25, теги и семантические описания.
3. **Состояние кэша (`project_state`)**: Версия анализатора, язык, fingerprint workspace и признак завершённого анализа.

Socket и лог не являются данными проекта: на Unix они находятся в `~/.znt/znt.sock` и `~/.znt/znt.log`; на Windows используется `\\.\pipe\znt-core`, а лог остаётся в `%USERPROFILE%\.znt\znt.log`. Файловый лог создаётся всегда и ведётся в append-режиме.

---

## 🛠 Доступные JSON-RPC Методы

Каждый запрос к `znt-core` передается в формате JSON-RPC 2.0:

| Метод | Параметры | Описание |
|-------|-----------|----------|
| `info` | `{}` | Версии, список методов и capabilities daemon |
| `status` | `{}` | Статистика индексированных узлов и файлов |
| `logs` | `{}` или `{"stream_id":"...","after_id":42}` | Начальный снимок или новые записи журнала после cursor |
| `scan` | `{"path": ".", "language": "auto"}` | Асинхронный запуск индексации директории |
| `znatok_semantic_search` | `{"query": "Search", "mode": "lexical"}` | Гибридный, лексический (FTS5) или векторный поиск |
| `znatok_find_similar` | `{"target": "SymbolName"}` | Поиск дубликатов, аналогов и паттернов реализации |
| `znatok_get_subgraph` | `{"from": "FuncA", "to": "FuncB"}` | Построение графа зависимостей и трассировка вызовов |
| `znatok_file_outline` | `{"file_path": "main.go"}` | Компактный атлас всех символов файла по строкам |
| `shutdown` | `{}` | Корректная остановка daemon (отклоняется во время scan) |

### Пример JSON-RPC запроса и ответа

**Запрос:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "znatok_file_outline",
  "params": {
    "file_path": "main.go"
  }
}
```

**Ответ:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "file": "main.go",
    "total_symbols": 5,
    "outline": "..."
  }
}
```

---

## 📚 Документация

| Документ | Описание |
|----------|----------|
| 🛠 **[Спецификация разработчика SDK](docs/sdk-specification.md)** | Полная техническая спецификация для написания клиентов/SDK для `znt-core` (типы DTO, сокеты, транспорт) |
| 🔌 **[Добавление нового языкового парсера](docs/adding-parser.md)** | Инструкция по расширению `znt-core` поддержкой новых языков программирования |
| ⚙️ **[Добавление режима генерации семантики](docs/adding-semantic-mode.md)** | Как реализовать собственный генератор описаний для узлов AST |
