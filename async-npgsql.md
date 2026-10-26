# Асинхронная работа в ADO.NET с Npgsql: полное руководство

Асинхронность в ADO.NET — это не просто «добавить `async/await` везде». Это фундаментальное изменение того, как приложение использует потоки и ресурсы базы данных. При работе с PostgreSQL через Npgsql есть свои важные нюансы, которые напрямую влияют на производительность и стабильность. Ниже — подробная теория и рабочие примеры, которые можно запустить сразу.

---

## 1. Теоретическая основа: что на самом деле делает асинхронность в ADO.NET

### 1.1. Почему синхронный вызов — это проблема

Когда вы вызываете `command.ExecuteReader()` синхронно, текущий поток блокируется на всё время выполнения запроса. В ASP.NET Core это означает, что поток из пула блокируется в ожидании ввода-вывода (сети, диска). При высокой нагрузке пул потоков истощается, и приложение перестаёт отвечать, даже если процессор свободен.

Асинхронный вызов (`ExecuteReaderAsync`) освобождает поток на время ожидания. Поток возвращается в пул и может обслуживать другие запросы. Когда база данных вернёт данные, продолжение выполнится на доступном потоке.

**Ключевой принцип:** Асинхронность в ADO.NET — это асинхронность ввода-вывода (I/O-bound), а не асинхронность вычислений (CPU-bound). Она не ускоряет сам запрос, но позволяет эффективнее использовать потоки.

### 1.2. Два поколения асинхронных API в ADO.NET

В ADO.NET исторически существовало два подхода:

**Старый подход (APM — Asynchronous Programming Model):** методы `BeginExecuteNonQuery` / `EndExecuteNonQuery`, основанные на `IAsyncResult` и `WaitHandle`. Вы увидите их в старых примерах Microsoft, но для нового кода они не рекомендуются. Они громоздки, плохо сочетаются с `async/await` и не являются истинно асинхронными на уровне драйвера в некоторых реализациях .

**Современный подход (TAP — Task-based Asynchronous Pattern):** методы с суффиксом `Async`, возвращающие `Task` или `ValueTask`. Это стандарт для .NET сегодня. Npgsql полностью поддерживает TAP, и именно его нужно использовать.

> **Важно:** В отличие от `System.Data.SqlClient` (SQL Server), для Npgsql **не требуется** добавлять `Asynchronous Processing=true` в строку подключения. Npgsql асинхронен по своей архитектуре на уровне протокола PostgreSQL.

### 1.3. Асинхронность на уровне протокола PostgreSQL

Npgsql реализует асинхронность через неблокирующие сокеты. Когда вы вызываете `ExecuteReaderAsync`, Npgsql отправляет запрос и возвращает управление, не блокируя поток. Чтение ответа происходит асинхронно по мере поступления данных от сервера.

Это означает, что асинхронные методы Npgsql являются **истинно асинхронными** (true async), а не просто обёрткой синхронного вызова в `Task.Run`.

---

## 2. Ключевые классы и их асинхронные методы

| Класс | Асинхронные методы | Когда использовать |
|---|---|---|
| `NpgsqlConnection` | `OpenAsync`, `CloseAsync`, `DisposeAsync` | Всегда, если приложение асинхронное |
| `NpgsqlCommand` | `ExecuteNonQueryAsync`, `ExecuteScalarAsync`, `ExecuteReaderAsync`, `PrepareAsync` | Любое выполнение SQL |
| `NpgsqlDataReader` | `ReadAsync`, `NextResultAsync`, `CloseAsync`, `DisposeAsync` | Чтение результатов построчно |
| `NpgsqlTransaction` | `CommitAsync`, `RollbackAsync`, `DisposeAsync` | Управление транзакциями |
| `NpgsqlConnection` (уведомления) | `WaitAsync` | Ожидание NOTIFY/LISTEN |

---

## 3. Правильные паттерны подключения и освобождения ресурсов

### 3.1. `await using` вместо `using`

`NpgsqlConnection`, `NpgsqlCommand`, `NpgsqlDataReader` и `NpgsqlTransaction` реализуют `IAsyncDisposable`. Использование `await using` гарантирует, что освобождение ресурсов (закрытие сокета, возврат соединения в пул) произойдёт асинхронно, не блокируя поток.

```csharp
// ПРАВИЛЬНО
await using var conn = new NpgsqlConnection(connectionString);
await conn.OpenAsync();

// НЕПРАВИЛЬНО (блокирует поток при Dispose)
using var conn = new NpgsqlConnection(connectionString);
```

### 3.2. Открытие соединения: `OpenAsync` vs пул

Npgsql имеет встроенный пул соединений. `OpenAsync` не всегда создаёт новое физическое соединение — если в пуле есть свободное, оно возвращается немедленно. Поэтому `OpenAsync` может завершиться синхронно (что нормально и эффективно).

**Паттерн из документации Npgsql:** если вы используете `NpgsqlDataSource`, открывайте соединение через `dataSource.OpenConnectionAsync()`. Это гарантирует корректную обработку ошибок и освобождение ресурсов при сбое открытия .

```csharp
await using var dataSource = NpgsqlDataSource.Create(connectionString);
await using var conn = await dataSource.OpenConnectionAsync();
```

---

## 4. Примеры для всех основных сценариев (запускаемые)

Все примеры предполагают наличие PostgreSQL и строку подключения. Замените `YOUR_CONNECTION_STRING` на реальную.

### 4.1. ExecuteNonQueryAsync — INSERT/UPDATE/DELETE

```csharp
using Npgsql;

var connectionString = "YOUR_CONNECTION_STRING";

// Создаём тестовую таблицу
await using (var conn = new NpgsqlConnection(connectionString))
{
    await conn.OpenAsync();
    
    // DDL через ExecuteNonQueryAsync
    await using var createCmd = new NpgsqlCommand(
        "CREATE TABLE IF NOT EXISTS async_demo (id SERIAL PRIMARY KEY, name TEXT NOT NULL)", 
        conn);
    await createCmd.ExecuteNonQueryAsync();
}

// Вставка с параметрами — критически важно для безопасности и производительности
await using (var conn = new NpgsqlConnection(connectionString))
{
    await conn.OpenAsync();
    
    await using var insertCmd = new NpgsqlCommand(
        "INSERT INTO async_demo (name) VALUES (@name)", conn);
    
    // Всегда используйте параметры, а не конкатенацию строк
    insertCmd.Parameters.AddWithValue("name", "Тестовая запись");
    
    // Возвращает количество затронутых строк
    int rowsAffected = await insertCmd.ExecuteNonQueryAsync();
    Console.WriteLine($"Вставлено строк: {rowsAffected}");
}
```

**Нюанс:** `AddWithValue` удобен, но для строк переменной длины может привести к неоптимальным планам запросов в PostgreSQL. В production лучше указывать тип явно: `NpgsqlDbType.Text`.

### 4.2. ExecuteScalarAsync — получение одного значения

```csharp
await using (var conn = new NpgsqlConnection(connectionString))
{
    await conn.OpenAsync();
    
    await using var cmd = new NpgsqlCommand(
        "SELECT COUNT(*) FROM async_demo", conn);
    
    // ExecuteScalarAsync возвращает первый столбец первой строки
    // Возвращаемый тип — object, требуется приведение
    long count = (long)(await cmd.ExecuteScalarAsync())!;
    Console.WriteLine($"Всего записей: {count}");
}
```

**Нюанс:** Если запрос не вернул строк, `ExecuteScalarAsync` вернёт `null`. Приведение `(long)null` выбросит исключение. Используйте `as long?` или проверку на `null`.

### 4.3. ExecuteReaderAsync — чтение набора данных

Это самый важный сценарий, где асинхронность даёт максимальный эффект.

```csharp
await using (var conn = new NpgsqlConnection(connectionString))
{
    await conn.OpenAsync();
    
    await using var cmd = new NpgsqlCommand(
        "SELECT id, name FROM async_demo ORDER BY id", conn);
    
    // ExecuteReaderAsync возвращает NpgsqlDataReader
    await using var reader = await cmd.ExecuteReaderAsync();
    
    // ReadAsync возвращает Task<bool>: true — есть строка, false — конец
    while (await reader.ReadAsync())
    {
        int id = reader.GetInt32(0);        // по индексу
        string name = reader.GetString(1);  // по индексу
        
        // Альтернатива: по имени столбца (медленнее из-за поиска)
        // int id = reader.GetInt32(reader.GetOrdinal("id"));
        
        Console.WriteLine($"{id}: {name}");
    }
}
```

**Нюанс:** `ReadAsync` не загружает весь результат в память. Данные читаются с сервера по мере необходимости. Это критически важно для больших выборок. Но помните: пока reader открыт, соединение занято. Не выполняйте другие команды на том же соединении внутри цикла чтения.

### 4.4. Обработка NULL в DataReader

```csharp
await using var cmd = new NpgsqlCommand(
    "SELECT id, name FROM async_demo WHERE id = @id", conn);
cmd.Parameters.AddWithValue("id", 999999); // заведомо несуществующий

await using var reader = await cmd.ExecuteReaderAsync();

if (await reader.ReadAsync())
{
    // IsDBNull проверяет NULL перед чтением
    string? name = await reader.IsDBNullAsync(1) ? null : reader.GetString(1);
}
else
{
    Console.WriteLine("Строка не найдена");
}
```

### 4.5. Асинхронные транзакции

Транзакции в Npgsql полностью поддерживают асинхронность. Критически важно использовать `await using` для транзакции, чтобы избежать утечки незавершённых транзакций.

```csharp
await using (var conn = new NpgsqlConnection(connectionString))
{
    await conn.OpenAsync();
    
    // BeginTransactionAsync возвращает NpgsqlTransaction
    await using var tx = await conn.BeginTransactionAsync();
    
    try
    {
        await using var cmd1 = new NpgsqlCommand(
            "INSERT INTO async_demo (name) VALUES (@name)", conn, tx);
        cmd1.Parameters.AddWithValue("name", "Транзакция 1");
        await cmd1.ExecuteNonQueryAsync();
        
        await using var cmd2 = new NpgsqlCommand(
            "INSERT INTO async_demo (name) VALUES (@name)", conn, tx);
        cmd2.Parameters.AddWithValue("name", "Транзакция 2");
        await cmd2.ExecuteNonQueryAsync();
        
        // Фиксация
        await tx.CommitAsync();
        Console.WriteLine("Транзакция зафиксирована");
    }
    catch
    {
        // Откат при ошибке
        await tx.RollbackAsync();
        throw;
    }
}
```

**Важный нюанс Npgsql:** В блоке `finally` или при обработке исключений можно проверить свойство `NpgsqlTransaction.IsCompleted`. Если транзакция уже была зафиксирована или откачена, повторный вызов `RollbackAsync` выбросит исключение. `IsCompleted` позволяет избежать этого .

```csharp
finally
{
    if (!tx.IsCompleted)
    {
        await tx.RollbackAsync();
    }
}
```

### 4.6. Асинхронная подготовка команды (PrepareAsync)

Подготовленные команды (prepared statements) в PostgreSQL выполняются быстрее при повторном использовании. `PrepareAsync` подготавливает команду на сервере асинхронно.

```csharp
await using (var conn = new NpgsqlConnection(connectionString))
{
    await conn.OpenAsync();
    
    await using var cmd = new NpgsqlCommand(
        "SELECT name FROM async_demo WHERE id = @id", conn);
    cmd.Parameters.Add(new NpgsqlParameter("id", NpgsqlTypes.NpgsqlDbType.Integer));
    
    // Подготавливаем на сервере
    await cmd.PrepareAsync();
    
    // Теперь можно многократно выполнять с разными параметрами
    for (int i = 1; i <= 3; i++)
    {
        cmd.Parameters["id"].Value = i;
        var result = await cmd.ExecuteScalarAsync();
        Console.WriteLine($"id={i}: {result ?? "не найдено"}");
    }
}
```

**Нюанс:** `PrepareAsync` имеет смысл только если команда будет выполняться многократно. Для одноразовых запросов подготовка создаёт дополнительную нагрузку на сервер.

### 4.7. Асинхронная отмена операций (CancellationToken)

Все асинхронные методы Npgsql принимают `CancellationToken`. Это позволяет отменить длительный запрос, например, по таймауту или запросу пользователя.

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

try
{
    await using var conn = new NpgsqlConnection(connectionString);
    await conn.OpenAsync(cts.Token);
    
    await using var cmd = new NpgsqlCommand(
        "SELECT pg_sleep(10)", conn); // заведомо долгий запрос
    
    // Отмена произойдёт через 5 секунд
    await cmd.ExecuteNonQueryAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Запрос отменён по таймауту");
}
```

**Нюанс:** При отмене Npgsql отправляет PostgreSQL запрос на отмену (`CancelRequest`). Это не всегда мгновенно — сервер должен обработать отмену. Если отмена не удалась, соединение может быть помечено как сломанное и удалено из пула.

### 4.8. Асинхронное ожидание уведомлений PostgreSQL (LISTEN/NOTIFY)

Это специфичная для Npgsql функция, которая часто вызывает путаницу. Уведомления PostgreSQL (`NOTIFY`) — это **асинхронные сообщения на уровне протокола**, но они **не имеют отношения к `async/await` ADO.NET** .

Npgsql обрабатывает уведомления только во время выполнения обычных команд. Если вы хотите ждать уведомление, не выполняя запросов, используйте `WaitAsync()`.

```csharp
await using var conn = new NpgsqlConnection(connectionString);
await conn.OpenAsync();

// Подписываемся на событие
conn.Notification += (o, e) =>
{
    Console.WriteLine($"Получено уведомление: канал={e.Channel}, payload={e.Payload}");
};

// Начинаем слушать канал
await using (var listenCmd = new NpgsqlCommand("LISTEN my_channel", conn))
{
    await listenCmd.ExecuteNonQueryAsync();
}

// Теперь ждём уведомление. WaitAsync освобождает поток.
// Без таймаута будет ждать вечно — в реальном коде используйте CancellationToken
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

try
{
    await conn.WaitAsync(cts.Token);
    Console.WriteLine("Уведомление получено и обработано");
}
catch (OperationCanceledException)
{
    Console.WriteLine("Таймаут ожидания уведомления");
}
```

**Критический нюанс:** `WaitAsync` в Npgsql — это **не** метод расширения ADO.NET. Это специфичный для Npgsql метод, который блокирует выполнение до получения одного уведомления. Не путайте его с `Task.WaitAsync` из .NET 6+.

### 4.9. Пакетная обработка и несколько результатов (NextResultAsync)

Npgsql поддерживает выполнение нескольких SQL-запросов в одной команде (батчинг). `NextResultAsync` переключается между результирующими наборами.

```csharp
await using var conn = new NpgsqlConnection(connectionString);
await conn.OpenAsync();

await using var cmd = new NpgsqlCommand(
    "SELECT 1 AS a; SELECT 2 AS b, 3 AS c;", conn);

await using var reader = await cmd.ExecuteReaderAsync();

int resultSetIndex = 0;
do
{
    Console.WriteLine($"--- Набор {resultSetIndex} ---");
    while (await reader.ReadAsync())
    {
        for (int i = 0; i < reader.FieldCount; i++)
        {
            Console.Write($"{reader.GetName(i)}={reader.GetValue(i)} ");
        }
        Console.WriteLine();
    }
    resultSetIndex++;
} while (await reader.NextResultAsync());
```

**Нюанс:** Батчинг в PostgreSQL имеет ограничения. Некоторые типы команд (например, `COPY`) нельзя смешивать с обычными запросами в одном батче. Также батчинг не работает с подготовленными командами в некоторых режимах.

### 4.10. Асинхронное освобождение и пул соединений

```csharp
// Правильный паттерн: await using гарантирует асинхронное закрытие
// и возврат соединения в пул
await using (var conn = new NpgsqlConnection(connectionString))
{
    await conn.OpenAsync();
    // ... работа ...
} // здесь вызывается DisposeAsync, соединение возвращается в пул

// Проверка статистики пула
var dataSource = NpgsqlDataSource.Create(connectionString);
Console.WriteLine($"Соединений в пуле: {dataSource.Statistics.Total}");
```

**Нюанс:** Если вы используете `using` (синхронный) вместо `await using`, при Dispose будет вызван синхронный `Close()`, который блокирует поток. В асинхронном приложении это может привести к снижению пропускной способности под нагрузкой.

---

## 5. Распространённые ошибки и подводные камни

### 5.1. Смешивание синхронных и асинхронных вызовов

Никогда не вызывайте `.Result` или `.Wait()` на асинхронных методах Npgsql в контексте с `SynchronizationContext` (например, в ASP.NET Classic или UI-потоке). Это приведёт к взаимной блокировке (deadlock). В ASP.NET Core `SynchronizationContext` отсутствует, но проблема может возникнуть при использовании `Task.Run` с блокирующими вызовами внутри.

### 5.2. Забытый `await` у `OpenAsync`

```csharp
// ОПАСНО: соединение не открыто, следующий запрос упадёт
var conn = new NpgsqlConnection(connectionString);
conn.OpenAsync(); // Task проигнорирован

// ПРАВИЛЬНО
await conn.OpenAsync();
```

### 5.3. Чтение данных без `await` в цикле

```csharp
// НЕПРАВИЛЬНО: блокирует поток
while (reader.Read()) { ... }

// ПРАВИЛЬНО
while (await reader.ReadAsync()) { ... }
```

### 5.4. Использование `AddWithValue` с `null`

```csharp
// Может привести к неожиданному поведению: Npgsql попытается вывести тип из null
cmd.Parameters.AddWithValue("name", null); // ошибка или неверный тип

// ПРАВИЛЬНО
cmd.Parameters.Add(new NpgsqlParameter("name", NpgsqlDbType.Text) { Value = DBNull.Value });
```

### 5.5. Ожидание уведомлений без `WaitAsync`

Если вы выполнили `LISTEN` и просто ждёте, ничего не делая, уведомление не будет обработано, пока вы не выполните следующую команду. Npgsql обрабатывает уведомления только в рамках цикла запрос-ответ .

---

## 6. Производительность: что реально даёт асинхронность

| Сценарий | Синхронный код | Асинхронный код |
|---|---|---|
| Веб-API, 100 одновременных запросов к БД | Пул потоков истощается, запросы встают в очередь | Потоки освобождаются во время I/O, пропускная способность выше |
| Консольная утилита с одним запросом | Работает нормально | Не даёт выигрыша (оверхед на state machine) |
| Чтение 1 млн строк | Поток заблокирован на всё время | Поток свободен, но данные всё равно читаются последовательно |
| Пакет из 1000 INSERT | Блокирует поток | Можно выполнять параллельно (с осторожностью) |

**Вывод:** Асинхронность критически важна для серверных приложений с высокой конкурентностью. Для скриптов и утилит с одним-двумя запросами она не даёт преимуществ, но и не вредит.

---

## 7. Итоговый чек-лист для production-кода с Npgsql

1. **Всегда** используйте `await using` для `NpgsqlConnection`, `NpgsqlCommand`, `NpgsqlDataReader`, `NpgsqlTransaction`.
2. **Всегда** передавайте `CancellationToken` в асинхронные методы (особенно в веб-приложениях).
3. **Никогда** не используйте `.Result` / `.Wait()` на асинхронных вызовах Npgsql.
4. **Проверяйте `IsCompleted`** у транзакции перед `RollbackAsync` в обработчике ошибок.
5. **Используйте параметры** (`AddWithValue` или `Add` с явным типом) вместо конкатенации SQL.
6. **Для уведомлений** используйте `WaitAsync` с `CancellationToken` — иначе поток заблокируется навсегда.
7. **Не открывайте** несколько reader'ов на одном соединении одновременно (если не используете MARS, которого в PostgreSQL нет).
8. **Для больших выборок** читайте построчно через `ReadAsync`, не загружайте всё в `List<T>` без необходимости.
