Ниже приведён полный SQL-скрипт для создания базы данных и всех таблиц, необходимых для выполнения 10 задач по Npgsql. Скрипт включает тестовые данные, чтобы задачи можно было запускать сразу после выполнения.

---

## Полный скрипт инициализации БД

```sql
-- ============================================================
-- База данных для задач по асинхронной работе с Npgsql
-- PostgreSQL 12+
-- ============================================================

-- Создание БД (выполнить отдельно, подключившись к postgres)
-- CREATE DATABASE npgsql_tasks;
-- \c npgsql_tasks

-- ============================================================
-- Задача 1: Сотрудники (пагинация, отмена, таймауты)
-- ============================================================
DROP TABLE IF EXISTS employees CASCADE;
CREATE TABLE employees (
    id          SERIAL PRIMARY KEY,
    first_name  VARCHAR(100) NOT NULL,
    last_name   VARCHAR(100) NOT NULL,
    email       VARCHAR(255) UNIQUE,
    salary      NUMERIC(12,2),
    hired_at    TIMESTAMPTZ DEFAULT now()
);

-- 50 сотрудников для тестов пагинации
INSERT INTO employees (first_name, last_name, email, salary)
SELECT
    'First' || i,
    'Last' || i,
    'employee' || i || '@example.com',
    (random() * 100000)::NUMERIC(12,2)
FROM generate_series(1, 50) AS i;

-- ============================================================
-- Задача 2: Заказы (COPY / массовая вставка)
-- ============================================================
DROP TABLE IF EXISTS orders CASCADE;
CREATE TABLE orders (
    id          INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    amount      NUMERIC(12,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Индекс для отчётов и выборок по клиенту
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_created ON orders(created_at);

-- 10 заказов для начальных тестов
INSERT INTO orders (id, customer_id, amount, created_at)
SELECT
    i,
    (i % 10) + 1,
    (random() * 10000)::NUMERIC(12,2),
    now() - (i || ' hours')::INTERVAL
FROM generate_series(1, 10) AS i;

-- ============================================================
-- Задача 3: Счета (транзакции, переводы)
-- ============================================================
DROP TABLE IF EXISTS accounts CASCADE;
CREATE TABLE accounts (
    id          INTEGER PRIMARY KEY,
    owner_name  VARCHAR(200) NOT NULL,
    balance     NUMERIC(15,2) NOT NULL DEFAULT 0,
    CONSTRAINT chk_balance_non_negative CHECK (balance >= 0)
);

-- Два счёта с начальными балансами
INSERT INTO accounts (id, owner_name, balance) VALUES
    (1, 'Alice', 10000.00),
    (2, 'Bob', 5000.00);

-- ============================================================
-- Задача 4: Документы (BLOB / bytea)
-- ============================================================
DROP TABLE IF EXISTS documents CASCADE;
CREATE TABLE documents (
    id          SERIAL PRIMARY KEY,
    file_name   VARCHAR(255) NOT NULL,
    content     BYTEA NOT NULL,
    uploaded_at TIMESTAMPTZ DEFAULT now()
);

-- 5 небольших документов (от 1 КБ до 1 МБ) для тестов
INSERT INTO documents (file_name, content)
SELECT
    'doc_' || i || '.bin',
    decode(repeat('AB', i * 1024), 'hex')  -- i * 1024 байт
FROM generate_series(1, 5) AS i;

-- Для теста большого BLOB (500 МБ) — раскомментировать при необходимости
-- INSERT INTO documents (file_name, content)
-- VALUES ('big.bin', decode(repeat('CD', 500 * 1024 * 1024), 'hex'));

-- ============================================================
-- Задача 5: Категории и продукты (батчинг вместо MARS)
-- ============================================================
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

CREATE TABLE categories (
    id   INTEGER PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE products (
    id          INTEGER PRIMARY KEY,
    category_id INTEGER NOT NULL REFERENCES categories(id),
    name        VARCHAR(200) NOT NULL,
    price       NUMERIC(10,2) NOT NULL
);

INSERT INTO categories (id, name) VALUES
    (1, 'Electronics'),
    (2, 'Books'),
    (3, 'Clothing');

-- 30 продуктов, по 10 в каждой категории
INSERT INTO products (id, category_id, name, price)
SELECT
    i,
    (i % 3) + 1,
    'Product ' || i,
    (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 30) AS i;

CREATE INDEX idx_products_category ON products(category_id);

-- ============================================================
-- Задачи 6, 9: Таблица для нагрузочных тестов пула
-- ============================================================
DROP TABLE IF EXISTS pool_test CASCADE;
CREATE TABLE pool_test (
    id         SERIAL PRIMARY KEY,
    payload    TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);

INSERT INTO pool_test (payload)
SELECT 'test_' || i
FROM generate_series(1, 100) AS i;

-- ============================================================
-- Задача 7: Health check (ничего дополнительного не нужно)
-- ============================================================
-- Используется SELECT 1 и statement_timeout на сессии

-- ============================================================
-- Задача 8: Пакетная вставка с ошибками (EnableErrorBarriers)
-- ============================================================
DROP TABLE IF EXISTS batch_items CASCADE;
CREATE TABLE batch_items (
    id    INTEGER PRIMARY KEY,
    name  VARCHAR(100) NOT NULL
);

-- Начальные данные отсутствуют — тест вставляет и проверяет дубликаты

-- ============================================================
-- Задача 10: Полный DAL (та же таблица orders, но с версией)
-- ============================================================
-- Для оптимистичной блокировки через xmin отдельная колонка не нужна.
-- Если хотите использовать явную version BIGINT:
-- ALTER TABLE orders ADD COLUMN version BIGINT DEFAULT 1;

-- ============================================================
-- Проверка: все таблицы созданы
-- ============================================================
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;
```

---

## Пояснения по таблицам и задачам

| Таблица | Задачи | Назначение |
|---------|--------|------------|
| `employees` | 1 | Пагинация, отмена, таймауты |
| `orders` | 2, 5, 10 | COPY, батчинг, полный DAL |
| `accounts` | 3 | Транзакции, переводы |
| `documents` | 4 | SequentialAccess, `bytea` |
| `categories` / `products` | 5 | `NpgsqlBatch` вместо MARS |
| `pool_test` | 6, 9 | Нагрузка на пул, параллельные запросы |
| `batch_items` | 8 | `EnableErrorBarriers` |

---

## Дополнительные настройки для задач

### Для задачи 3 (транзакции)

Убедитесь, что уровень изоляции поддерживает `SERIALIZABLE` (по умолчанию в PostgreSQL он есть). Проверка:

```sql
SHOW default_transaction_isolation;
-- Должно быть: read committed (можно менять на session level)
```

### Для задачи 4 (BLOB)

Если хотите протестировать действительно большой `bytea` (500 МБ), раскомментируйте `INSERT` с `repeat('CD', 500*1024*1024)`. Учтите, что это займёт много дискового пространства и времени. Для большинства тестов достаточно документов до 1 МБ.

### Для задачи 7 (health check)

`statement_timeout` можно установить на уровне роли:

```sql
-- Для пользователя, под которым работает приложение
ALTER ROLE postgres SET statement_timeout = '3000ms';

-- Или на уровне БД
ALTER DATABASE npgsql_tasks SET statement_timeout = '3000ms';
```

Но в коде health check лучше устанавливать `statement_timeout` на сессию после `OpenAsync`, чтобы не зависеть от глобальных настроек.

### Для задачи 8 (EnableErrorBarriers)

`batch_items` намеренно создана **без** начальных данных. В тесте вы вставите 100 строк, одна из которых будет дубликатом `id`, и проверите поведение с `EnableErrorBarriers` и без него.

### Для задачи 10 (оптимистичная блокировка)

В PostgreSQL системная колонка `xmin` доступна в любом `SELECT`. Для `UPDATE` с проверкой версии:

```sql
-- Чтение
SELECT id, customer_id, amount, xmin FROM orders WHERE id = 1;

-- Обновление с проверкой xmin
UPDATE orders
SET amount = 999.99
WHERE id = 1 AND xmin = 12345::xid;
```

Если `ExecuteNonQueryAsync` вернул `0` — версия устарела (конкурентное изменение).

---

## Быстрый старт

```bash
# 1. Подключиться к PostgreSQL
psql -U postgres -h localhost

# 2. Создать БД
CREATE DATABASE npgsql_tasks;

# 3. Подключиться к ней
\c npgsql_tasks

# 4. Выполнить весь скрипт выше
\i /path/to/init.sql

# 5. Проверить
\dt
```

После этого можно запускать C#-код из предыдущего ответа, заменив строку подключения на:

```
Host=localhost;Port=5432;Database=npgsql_tasks;Username=postgres;Password=ваш_пароль;
```




# Задачи
# 10 комплексных задач для отработки асинхронной работы с Npgsql

---

## Задача 1. Базовый асинхронный репозиторий с отменой и таймаутом

**Сценарий:** DAL-библиотека для ASP.NET Core и WPF. Нужен список сотрудников с пагинацией.

**Что проверяется:**
- `await using` для `NpgsqlConnection`, `NpgsqlCommand`, `NpgsqlDataReader`.
- `OpenAsync`, `ExecuteReaderAsync`, `ReadAsync` с `CancellationToken`.
- `CreateLinkedTokenSource` для связывания пользовательской отмены с внутренним таймаутом.
- `.ConfigureAwait(false)` в библиотечном коде.
- Синхронные геттеры после `ReadAsync` (в Npgsql данные строки уже в буфере при неsequential чтении).

**Требования:**
1. Метод `Task<List<Employee>> GetEmployeesAsync(int page, int pageSize, CancellationToken userToken)`.
2. Внутренний `CancelAfter(15s)`, `CommandTimeout = 10`. В комментариях объяснить: что произойдёт при срабатывании каждого, и почему `CommandTimeout` в Npgsql — это **клиентский socket timeout**, закрывающий соединение.
3. Параметризованный запрос (`@p0` или `$1` — синтаксис Npgsql).
4. `try/catch`, различающий `OperationCanceledException` (наш `CancelAfter` или пользовательская отмена), `NpgsqlException` с `PostgresException.SqlState == "57014"` (серверная отмена через `statement_timeout` или `Cancel()`) и `NpgsqlException` с таймаутом чтения потока.

**Подсказки:**
- Npgsql рекомендует устанавливать `statement_timeout` на уровне сервера или сессии **меньше**, чем клиентский `CommandTimeout`. Тогда сервер сам прервёт запрос и вернёт нормальное исключение `PostgresException` с `SqlState = "57014"`, сохранив соединение.
- `CommandTimeout = 0` означает «без лимита» в Npgsql, но это опасно.

**Критерии приёмки:**
- Код работает против реального PostgreSQL.
- В комментариях объяснена разница между клиентским socket timeout и серверным `statement_timeout`.
- Нет `.Result` / `.Wait()` / `async void`.

---

## Задача 2. Массовая вставка через `COPY` (бинарный импорт) с прогрессом

**Сценарий:** загрузить 1 000 000 строк в staging-таблицу. `INSERT` слишком медленный — нужен `COPY`.

**Что проверяется:**
- `BeginBinaryImportAsync("COPY ... FROM STDIN (FORMAT BINARY)")`.
- `StartRowAsync`, `WriteAsync`, `WriteNullAsync`, `CompleteAsync`.
- **Обязательный** вызов `Complete()` — иначе при `Dispose` произойдёт откат COPY.
- `IProgress<int>` для прогресса (поскольку `COPY` не имеет события `NotifyAfter`, как `SqlBulkCopy`).
- Отмена: `Dispose` без `Complete` прерывает импорт и откатывает уже вставленные строки.

**Требования:**
1. Метод `Task BulkCopyAsync(IEnumerable<Order> orders, IProgress<int> progress, CancellationToken token)`.
2. Использовать `NpgsqlDbType` в перегрузках `WriteAsync` для явного указания типов (защита от «silent data corruption»).
3. Каждые 10 000 строк вызывать `progress.Report(...)`.
4. При отмене: `await using` для importer без вызова `CompleteAsync()` — проверить, что таблица осталась пустой (или в состоянии до COPY).
5. В комментариях: почему `COPY` быстрее `INSERT`; почему нельзя вызвать `CompleteAsync` после исключения.

**Подсказки:**
- `NpgsqlBinaryImporter` реализует `IAsyncDisposable`. Если `CompleteAsync()` не был вызван, `DisposeAsync` откатывает операцию.
- Для NULL используйте `WriteNullAsync()`, а не `WriteAsync(null)`.
- Прогресс считайте вручную: `rowsWritten++` после каждой строки, `Report` каждые N строк.

**Критерии приёмки:**
- На 1 млн строк пиковая память не растёт линейно.
- Отмена на середине оставляет таблицу в исходном состоянии (проверить `COUNT(*)`).
- В логе — точное число записанных строк при отмене.

---

## Задача 3. Транзакция с корректным откатом при отмене (состояние «aborted»)

**Сценарий:** перевод средств между счетами. PostgreSQL при ошибке внутри транзакции переводит её в состояние `idle in transaction (aborted)`, но **не откатывает** автоматически. Нужно корректно обработать это.

**Что проверяется:**
- `BeginTransactionAsync`, `CommitAsync`, `RollbackAsync`.
- Понимание, что после любой ошибки (включая отмену) транзакция в PostgreSQL **уже aborted** — но `ROLLBACK` всё равно нужно вызвать для очистки состояния.
- Отдельный cleanup-токен для `RollbackAsync` (нельзя использовать отменённый `userToken`).
- Проверка `PostgresException` и `SqlState`.

**Требования:**
1. Метод `Task TransferAsync(int fromId, int toId, decimal amount, CancellationToken userToken)`.
2. Уровень изоляции `Serializable` (обосновать).
3. `SELECT ... FOR UPDATE` для блокировки строк.
4. В `catch`: `RollbackAsync(CancellationToken.None)` или отдельный `CancellationTokenSource(5s)`.
5. Если `RollbackAsync` тоже упал — `AggregateException`.
6. В комментариях: объяснить, почему в PostgreSQL после `PostgresException` внутри транзакции все последующие команды игнорируются, пока не выполнен `ROLLBACK`.

**Подсказки:**
- `NpgsqlTransaction` можно использовать в `await using` — `DisposeAsync` без `Commit` выполнит `ROLLBACK`.
- Но **не полагайтесь** на `Dispose` для обработки ошибок — явный `RollbackAsync` в `catch` даёт контроль над таймаутом и логированием.

**Критерии приёмки:**
- После отмены на любом этапе баланс обоих счетов неизменен.
- В тестах: отмена до `BeginTransaction`, после, после первого `UPDATE`, после второго, во время `Commit`.
- В каждом случае — консистентное состояние.

---

## Задача 4. SequentialAccess + потоковая обработка `bytea` (BLOB)

**Сценарий:** в таблице `documents` лежат PDF-файлы до 500 МБ в колонке `bytea`. Нужно скопировать их в файловую систему без загрузки в память.

**Что проверяется:**
- `CommandBehavior.SequentialAccess` в `ExecuteReaderAsync`.
- `GetFieldValueAsync<byte[]>` в sequential режиме — данные подгружаются по требованию.
- Понимание, что в Npgsql `GetStream` для `bytea` **не поддерживается** напрямую (в отличие от `SqlClient`); для потоковой обработки больших `bytea` нужно использовать `GetFieldValueAsync<byte[]>` в sequential режиме, либо `GetBytes`/`GetTextReader` (если применимо).
- Строгий порядок чтения колонок в sequential режиме.

**Требования:**
1. Метод `Task ExportDocumentsAsync(string outputDir, CancellationToken token)`.
2. Запрос: `SELECT id, file_name, content FROM documents ORDER BY id`.
3. `ExecuteReaderAsync(CommandBehavior.SequentialAccess, token)`.
4. Читать `id` и `file_name` синхронно, `content` — через `await reader.GetFieldValueAsync<byte[]>(2, token)`.
5. Записывать в файл через `FileStream` с `FileOptions.Asynchronous` и `CopyToAsync`.
6. В комментариях: что произойдёт, если поменять порядок чтения колонок; почему для маленьких `bytea` (до ~1 МБ) sequential не даёт выгоды.

**Подсказки:**
- `GetFieldValueAsync<byte[]>` в sequential режиме всё равно загрузит **весь** `bytea` в массив — Npgsql не предоставляет потоковый API для `bytea` на уровне одного поля. Для действительно больших BLOB в PostgreSQL используют `Large Objects` (`lo_read`/`lo_write`), но это отдельная тема.
- Для 500 МБ `bytea` это всё равно загрузит 500 МБ в память. Задача — показать **ограничения** подхода и обсудить альтернативы (`Large Objects`, внешнее хранение файлов).

**Критерии приёмки:**
- Пиковая память процесса ≤ размер самого большого документа (не всех сразу).
- Файлы побайтово совпадают с содержимым в БД.
- В комментариях обсуждены ограничения `bytea` для очень больших данных.

---

## Задача 5. Батчинг вместо MARS: несколько запросов за один round-trip

**Сценарий:** нужно выполнить 5 независимых `SELECT` и 3 `INSERT` в рамках одной логической операции. В SQL Server вы бы использовали MARS. В Npgsql MARS **нет** — нужно использовать `NpgsqlBatch`.

**Что проверяется:**
- `NpgsqlBatch` и `BatchCommands`.
- `ExecuteReaderAsync` на батче с несколькими результирующими наборами.
- `NextResultAsync` для перехода между результатами.
- Понимание, что батч **не даёт параллелизма** — команды выполняются последовательно на сервере, но **один round-trip** экономит сетевую латентность.
- Альтернатива: два отдельных соединения из пула для настоящего параллелизма.

**Требования:**
1. Метод `Task<ReportData> BuildReportAsync(int customerId, CancellationToken token)`.
2. Внутри — `NpgsqlBatch` с командами: (a) заказы клиента, (b) позиции заказов (JOIN), (c) сумма по клиенту.
3. Использовать `ExecuteReaderAsync` на батче, читать результаты через `NextResultAsync`.
4. Отдельный метод-демонстрация: сравнить время выполнения 3 запросов через `NpgsqlBatch` vs 3 отдельных `ExecuteReaderAsync` **на одном соединении последовательно** vs 3 отдельных на **двух соединениях параллельно**.
5. В комментариях: почему `NpgsqlBatch` — не замена MARS; когда нужен настоящий параллелизм (два соединения), а когда достаточно батча.

**Подсказки:**
- `NpgsqlBatch` требует `await using`.
- При ошибке в одной команде батча по умолчанию все последующие пропускаются, а предыдущие откатываются (если нет явной транзакции). `EnableErrorBarriers` позволяет изолировать ошибки.

**Критерии приёмки:**
- Отчёт строится без ошибки «connection is busy».
- Замеры времени показывают разницу между батчем и параллельными соединениями.
- В комментариях чётко: батч экономит round-trip'ы, но не даёт параллельного выполнения на сервере.

---

## Задача 6. Пул соединений Npgsql: исчерпание и диагностика

**Сценарий:** production падает с `NpgsqlException: The connection pool has been exhausted`. Нужно воспроизвести и диагностировать.

**Что проверяется:**
- Параметры пула: `Maximum Pool Size`, `Timeout`, `Connection Idle Lifetime`.
- Разница между `Maximum Pool Size` (клиентский пул Npgsql) и `max_connections` на сервере PostgreSQL.
- Поведение `OpenAsync` при исчерпании пула: асинхронное ожидание, затем `NpgsqlException` с timeout.
- `NpgsqlConnection.ClearPool` / `ClearAllPools`.

**Требования:**
1. «Убийца пула»: `Task ExhaustPoolAsync(int count)` — открыть `count` соединений и **не закрывать** (держать в списке). `Maximum Pool Size=5;Timeout=5` в строке подключения.
2. Диагностика: `Task DiagnosePoolAsync(token)` — попытаться открыть соединение, замерить время, поймать `NpgsqlException` с сообщением про pool exhausted.
3. «Лекарь»: `void HealPool()` — `NpgsqlConnection.ClearAllPools()`, проверить, что соединения снова открываются.
4. В комментариях: почему `Maximum Pool Size=800` в клиенте и `max_connections=3437` на сервере — это **два разных** лимита; как connection leak (незакрытое соединение) выглядит снаружи (пул исчерпан, но на сервере мало соединений).

**Подсказки:**
- `ClearAllPools()` помечает пулы как недействительные; новые `Open` создадут новые физические соединения.
- Для проверки реального количества соединений на сервере: `SELECT count(*) FROM pg_stat_activity WHERE datname = current_database()`.

**Критерии приёмки:**
- Воспроизведён `NpgsqlException` с текстом про pool exhausted.
- После `ClearAllPools` соединения открываются.
- В логе — разница во времени между «из пула» и «новое физическое».

---

## Задача 7. Health check с `statement_timeout` и `Cancel`

**Сценарий:** компонент `IHealthCheck` для ASP.NET Core, проверяющий PostgreSQL, но не «вешающий» приложение.

**Что проверяется:**
- `OpenAsync` с коротким `Timeout` (это **клиентский** таймаут пула).
- `ExecuteScalarAsync("SELECT 1")`.
- `CancellationTokenSource.CancelAfter(3s)` для «нашего» таймаута.
- Понимание: в Npgsql отмена через `CancellationToken` или `NpgsqlCommand.Cancel()` отправляет серверу сигнал отмены, но если сеть потеряна — клиент зависнет (socket timeout для async не работает).
- `statement_timeout` на уровне сервера/сессии как **дополнительная** защита.

**Требования:**
1. `DatabaseHealthCheck : IHealthCheck`.
2. Внутренний `CancelAfter(3s)`. `Timeout=5` в connection string (намеренно больше).
3. Перед проверкой: `SET statement_timeout = '2000ms'` (или через `Options` в connection string, если поддерживается). Обосновать в комментариях: зачем нужен серверный таймаут, если уже есть клиентский.
4. Возвращать `Healthy`, `Degraded` (медленно, но работает) или `Unhealthy(ex)` с временем в `data`.
5. В комментариях: почему `CommandTimeout` (клиентский socket timeout) **закрывает соединение**, а `statement_timeout` — нет; почему health check должен использовать `statement_timeout`, а не `CommandTimeout`.

**Подсказки:**
- `statement_timeout` можно установить в connection string Npgsql через `Options=-c statement_timeout=2000` (если версия Npgsql поддерживает), либо отдельной командой `SET statement_timeout = 2000` после `OpenAsync`, либо на уровне роли в PostgreSQL.
- Если `CancelAfter` сработает раньше `statement_timeout`, вы получите `OperationCanceledException`. Если `statement_timeout` — `PostgresException` с `SqlState = "57014"`.

**Критерии приёмки:**
- При выключенном PostgreSQL health check возвращает `Unhealthy` за ~3 секунды.
- При работающем — `Healthy` с временем.
- При «зависшем» запросе (эмуляция через `pg_sleep(10)`) — `Unhealthy` за ~2 секунды (сработал `statement_timeout`), соединение остаётся живым.

---

## Задача 8. `NpgsqlBatch` с `EnableErrorBarriers` и обработка частичных ошибок

**Сценарий:** пакетная обработка 100 `INSERT`. Если один падает (например, нарушение уникальности), остальные должны **выполниться**, а не быть пропущены. В SQL Server для этого используют `TRY/CATCH` в одном batch. В Npgsql есть `EnableErrorBarriers`.

**Что проверяется:**
- `NpgsqlBatch.EnableErrorBarriers = true`.
- Понимание, что по умолчанию первая ошибка откатывает предыдущие команды батча и пропускает последующие.
- С `EnableErrorBarriers` каждая команда изолирована (на уровне протокола — `Sync` между командами).
- Ограничение: если батч выполняется **внутри явной транзакции**, первая ошибка всё равно переводит транзакцию в aborted, и все последующие команды упадут. `EnableErrorBarriers` полезен только **без** явной транзакции.

**Требования:**
1. Метод `Task<BatchResult> InsertBatchWithBarriersAsync(IEnumerable<Order> orders, CancellationToken token)`.
2. `NpgsqlBatch` с `EnableErrorBarriers = true`, 100 команд `INSERT`.
3. Одна из команд намеренно нарушает уникальность (дубликат `id`).
4. Вернуть `BatchResult` с: количество успешных, количество ошибок, список `PostgresException` (по одной на ошибку).
5. Второй метод: то же самое, но **внутри** `BeginTransactionAsync` — показать, что `EnableErrorBarriers` не спасает от aborted transaction.
6. В комментариях: объяснить разницу между поведением с barriers и без; почему для частичных ошибок без транзакции barriers работают, а с транзакцией — нет.

**Подсказки:**
- `NpgsqlBatch.ExecuteNonQueryAsync` возвращает суммарное количество затронутых строк. Для получения ошибок по каждой команде нужно использовать `ExecuteReaderAsync` и читать результаты через `NextResultAsync`, перехватывая исключения.
- Альтернатива: выполнять команды по одной в цикле с `try/catch` — проще, но больше round-trip'ов.

**Критерии приёмки:**
- С `EnableErrorBarriers` и без транзакции: 99 успешных, 1 ошибка (дубликат).
- Без barriers: только команды до первой ошибки выполнены (или откатаны), остальные пропущены.
- С barriers **и** транзакцией: транзакция aborted после первой ошибки, все последующие команды падают.

---

## Задача 9. Параллельная обработка с ограничением конкурентности (и учёт пула)

**Сценарий:** 100 независимых запросов к PostgreSQL, но не более 10 одновременно (чтобы не исчерпать пул и не «положить» сервер).

**Что проверяется:**
- `SemaphoreSlim` для ограничения конкурентности.
- Разница между «100 задач» и «100 задач, но ≤10 соединений одновременно».
- `Task.WhenAll` + `AggregateException`.
- `CancellationToken` для отмены всех задач.
- Взаимодействие с пулом Npgsql: если `Maximum Pool Size < maxConcurrency`, задачи будут ждать в пуле **асинхронно**, но это всё равно расходует «слот» ожидания.

**Требования:**
1. Метод `Task<List<QueryResult>> RunParallelAsync(int queryCount, int maxConcurrency, CancellationToken token)`.
2. `SemaphoreSlim(maxConcurrency)` вокруг **каждой** операции (от `OpenAsync` до `DisposeAsync`).
3. `Task.WhenAll` + сбор всех исключений с контекстом (какой запрос упал).
4. Замерить пиковое количество соединений на сервере (`pg_stat_activity`) и сравнить с `maxConcurrency`.
5. В комментариях: что произойдёт, если `Maximum Pool Size = 5`, а `maxConcurrency = 10`; как это соотносится с поведением пула Npgsql (асинхронное ожидание, затем timeout).

**Подсказки:**
- `SemaphoreSlim.WaitAsync(token)` — асинхронный захват.
- `Task.WhenAll` дожидается всех задач; исключения собираются в `AggregateException`.
- Для замера пика соединений: отдельный «монитор» в фоновой задаче, опрашивающий `pg_stat_activity` каждые 100 мс.

**Критерии приёмки:**
- Пиковое количество соединений на сервере ≤ `maxConcurrency` **и** ≤ `Maximum Pool Size`.
- При отмене все задачи завершаются в течение ~1 секунды.
- В логе — какое исключение к какому запросу относится.

---

## Задача 10. Полноценный асинхронный DAL для PostgreSQL с тестами

**Сценарий:** production-ready репозиторий для `Order` с полным CRUD, COPY, батчингом и тестами на Testcontainers.

**Что проверяется:**
- Всё из задач 1–9.
- Архитектура: `IOrderRepository`, `NpgsqlOrderRepository`, DI.
- Логирование с корректным уровнем для каждого типа ошибки.
- Тесты с `xUnit` + `Testcontainers.PostgreSql` (реальный PostgreSQL в Docker).
- Тесты на отмену, конкуренцию (оптимистичная блокировка через `xmin` или отдельную колонку `version`), COPY с отменой.

**Требования:**

**10.1. Интерфейс:**
```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id, CancellationToken token);
    Task<List<Order>> GetByCustomerAsync(int customerId, int page, int pageSize, CancellationToken token);
    Task<int> CreateAsync(Order order, CancellationToken token);
    Task UpdateAsync(Order order, uint expectedVersion, CancellationToken token);
    Task DeleteAsync(int id, CancellationToken token);
    Task BulkCopyAsync(IEnumerable<Order> orders, IProgress<int> progress, CancellationToken token);
    Task<OrderReport> BuildReportAsync(int customerId, CancellationToken token);
}
```

**10.2. Требования к реализации:**
- Все методы — `async Task`, без `.Result`/`.Wait()`.
- Все `NpgsqlConnection`/`NpgsqlCommand`/`NpgsqlDataReader` — через `await using`.
- Все библиотечные `await` — с `.ConfigureAwait(false)`.
- `CancellationToken` пробрасывается везде, включая `BeginBinaryImportAsync`.
- `UpdateAsync` использует оптимистичную блокировку через системную колонку `xmin` (в PostgreSQL это `uint`/`xid`) или отдельную `version BIGINT`.
- `BulkCopyAsync` — `COPY ... FROM STDIN (FORMAT BINARY)` с `CompleteAsync` и `IProgress<int>`.
- `BuildReportAsync` — `NpgsqlBatch` (не MARS!), с чтением нескольких результирующих наборов через `NextResultAsync`.
- Транзакции — только `NpgsqlTransaction`, с cleanup-токеном для `RollbackAsync`.

