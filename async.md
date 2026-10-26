# Асинхронная работа в ADO.NET: полное руководство с примерами

Асинхронное программирование в ADO.NET — это не просто замена синхронных методов на их аналоги с суффиксом `Async`. Это фундаментальное изменение того, как приложение использует потоки при ожидании операций ввода-вывода (сеть, диск). Ниже разобраны все ключевые нюансы, подводные камни и приведены полностью рабочие примеры для типовых и не очень ситуаций.

---

## 1. Основы: Почему это работает и что нужно запомнить

Когда вы вызываете синхронный `SqlCommand.ExecuteReader()`, поток, выполняющий этот код, блокируется до тех пор, пока SQL Server не вернет первый результат (или не завершится таймаут). В веб-приложении или UI это означает, что поток из пула тратится впустую на ожидание.

Асинхронные методы (`ExecuteReaderAsync`, `OpenAsync`) возвращают `Task`. Пока задача не завершена, поток освобождается и может обслуживать другие запросы. **Важно:** асинхронность не ускоряет выполнение запроса на сервере. Она лишь освобождает клиентский поток .

### Главное правило преобразования
Если вы переводите метод в асинхронный режим, вы должны «протащить» асинхронность через всю цепочку вызовов: от `OpenAsync` до `ReadAsync` и `DisposeAsync`. Смешивать синхронные и асинхронные вызовы в одном потоке выполнения (например, вызывать `.Result` или `.Wait()` на задаче) **категорически нельзя** — это приведет к взаимной блокировке (deadlock) в контексте синхронизации (UI, ASP.NET Classic) или просто сведет на нет всю пользу .

---

## 2. Базовый шаблон: Чтение данных с отменой и таймаутом

Этот пример показывает «золотой стандарт» асинхронного запроса. Здесь объединены `CancellationToken` (для управления со стороны вызывающего кода) и `CommandTimeout` (для ограничения на стороне сервера).

```csharp
using System;
using System.Data;
using System.Data.SqlClient;
using System.Threading;
using System.Threading.Tasks;

public class AsyncAdoExample
{
    // Строка подключения. В реальном коде храните ее в конфигурации.
    private const string ConnectionString = 
        "Server=localhost;Database=AdventureWorks;Integrated Security=true;TrustServerCertificate=true;";

    /// <summary>
    /// Асинхронно читает данные, используя токен отмены и таймаут команды.
    /// </summary>
    public static async Task ReadDataAsync(CancellationToken cancellationToken)
    {
        // 1. Создаем источник отмены, который автоматически сработает через 10 секунд.
        // Это наш "внешний" предохранитель, не зависящий от CommandTimeout.
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        cts.CancelAfter(TimeSpan.FromSeconds(10));

        // 2. Открываем соединение асинхронно. 
        // Передаем токен, чтобы можно было прервать даже процесс установки соединения (включая ожидание в пуле).
        await using var connection = new SqlConnection(ConnectionString);
        await connection.OpenAsync(cts.Token);

        // 3. Создаем команду. 
        // CommandTimeout — это внутренний таймаут драйвера на выполнение запроса и чтение данных по сети.
        // 0 означает "без лимита", но мы ставим 15 секунд для примера.
        await using var command = new SqlCommand(
            "SELECT TOP 10 BusinessEntityID, FirstName, LastName FROM Person.Person ORDER BY BusinessEntityID", 
            connection)
        {
            CommandTimeout = 15 
        };

        // 4. Выполняем запрос асинхронно. 
        // ExecuteReaderAsync возвращается, как только сервер пришлет первый результат (метаданные), 
        // а не когда придут все строки.
        await using SqlDataReader reader = await command.ExecuteReaderAsync(CommandBehavior.CloseConnection, cts.Token);

        // 5. Читаем строки. Каждый ReadAsync освобождает поток на время ожидания данных по сети.
        while (await reader.ReadAsync(cts.Token))
        {
            // GetFieldValueAsync используется для полей большого размера (varbinary(max), varchar(max)).
            // Для обычных полей (int, datetime) можно использовать синхронные GetInt32, GetString, 
            // так как данные уже в буфере.
            int id = reader.GetInt32(0);
            string firstName = await reader.GetFieldValueAsync<string>(1, cts.Token);
            string lastName = await reader.GetFieldValueAsync<string>(2, cts.Token);
            
            Console.WriteLine($"{id}: {firstName} {lastName}");
        }
        // Выход из using автоматически вызовет DisposeAsync для reader и connection.
    }
}
```

### Нюансы, которые здесь важны:
1.  **`CancellationTokenSource` vs `CommandTimeout`**: `CancellationToken` позволяет вам (или пользователю) нажать «Отмена» и прервать ожидание. `CommandTimeout` — это страховка на случай, если сервер «завис». Они работают в связке. Если сработает `CancelAfter`, вы получите `OperationCanceledException` (или `TaskCanceledException`). Если сработает `CommandTimeout`, вы получите `SqlException` с текстом о таймауте .
2.  **`CommandBehavior.CloseConnection`**: В примере я использовал этот флаг. Он говорит: «Когда я закрою `SqlDataReader`, автоматически закрой и соединение». Это удобно, но требует осторожности: вы не сможете использовать соединение для других команд, пока reader не закрыт. Если вам нужно несколько активных reader'ов на одном соединении, используйте **MARS** (Multiple Active Result Sets) в строке подключения .
3.  **`GetFieldValueAsync`**: Используйте его для потенциально больших полей. Для полей фиксированного размера (int, bit, datetime) данные уже загружены в буфер при чтении строки, поэтому синхронный `GetInt32` не блокирует поток .

---

## 3. Выполнение команд без возврата данных (INSERT/UPDATE/DELETE)

Здесь используется `ExecuteNonQueryAsync`. Ситуация проще, но есть нюанс с количеством затронутых строк.

```csharp
public static async Task<int> UpdateDataAsync(int businessEntityId, string newLastName, CancellationToken token)
{
    await using var connection = new SqlConnection(ConnectionString);
    await connection.OpenAsync(token);

    // Важно: не забывайте про параметризацию, чтобы избежать SQL-инъекций.
    await using var command = new SqlCommand(
        "UPDATE Person.Person SET LastName = @LastName WHERE BusinessEntityID = @Id", 
        connection);
    
    command.Parameters.Add("@LastName", SqlDbType.NVarChar, 50).Value = newLastName;
    command.Parameters.Add("@Id", SqlDbType.Int).Value = businessEntityId;

    // ExecuteNonQueryAsync возвращает количество затронутых строк.
    int rowsAffected = await command.ExecuteNonQueryAsync(token);
    
    return rowsAffected;
}
```

---
## Тема 5. Транзакции: Commit, Rollback, Savepoints, вложенность (MSSQL)

### 5.1. Базовая транзакция

```csharp
using (var conn = new SqlConnection(DbConfig.ConnectionString))
{
    conn.Open();
    using (var tran = conn.BeginTransaction())
    {
        try
        {
            // Каждая команда должна быть создана с этой транзакцией (3-й параметр)
            using (var cmd = new SqlCommand(
                "UPDATE Accounts SET Balance = Balance - @amt WHERE Id = @id", conn, tran))
            {
                cmd.Parameters.Add("@amt", SqlDbType.Decimal).Value = 100m;
                cmd.Parameters.Add("@id", SqlDbType.Int).Value = 1;
                if (cmd.ExecuteNonQuery() == 0)
                    throw new InvalidOperationException("Счёт не найден");
            }

            using (var cmd = new SqlCommand(
                "UPDATE Accounts SET Balance = Balance + @amt WHERE Id = @id", conn, tran))
            {
                cmd.Parameters.Add("@amt", SqlDbType.Decimal).Value = 100m;
                cmd.Parameters.Add("@id", SqlDbType.Int).Value = 2;
                cmd.ExecuteNonQuery();
            }

            tran.Commit(); // фиксируем обе операции
        }
        catch
        {
            tran.Rollback(); // откатываем всё
            throw;
        }
    }
}
```

**Нюанс:** если команда создана без транзакции (`new SqlCommand(sql, conn)`), она выполнится вне её, и `Rollback` её не затронет — данные останутся изменёнными.

### 5.2. Уровни изоляции

```csharp
// ReadCommitted — по умолчанию в SQL Server
using (var tran = conn.BeginTransaction(IsolationLevel.ReadCommitted)) { /* ... */ }

// Serializable — самый строгий, блокирует диапазоны
using (var tran = conn.BeginTransaction(IsolationLevel.Serializable)) { /* ... */ }

// Snapshot — требует включения на уровне базы; читатели не блокируют писателей
using (var tran = conn.BeginTransaction(IsolationLevel.Snapshot)) { /* ... */ }
```

### 5.3. Savepoints: частичный откат

```csharp
using (var tran = conn.BeginTransaction())
{
    try
    {
        // Шаг 1: создать клиента
        using (var cmd = new SqlCommand(
            "INSERT INTO Customers (Name) OUTPUT INSERTED.Id VALUES (@n)", conn, tran))
        {
            cmd.Parameters.Add("@n", SqlDbType.NVarChar, 100).Value = "Новый клиент";
            int customerId = (int)cmd.ExecuteScalar();

            // Точка сохранения после создания клиента
            tran.Save("AfterCustomer");

            try
            {
                // Шаг 2: заказ материалов (может упасть)
                using (var cmd2 = new SqlCommand(
                    "INSERT INTO MaterialRequests (CustomerId, PartId) VALUES (@c, @p)", conn, tran))
                {
                    cmd2.Parameters.Add("@c", SqlDbType.Int).Value = customerId;
                    cmd2.Parameters.Add("@p", SqlDbType.Int).Value = 999;
                    cmd2.ExecuteNonQuery();
                }
                tran.Commit(); // всё прошло
            }
            catch (SqlException)
            {
                // Откат только шага 2; клиент остаётся созданным
                tran.Rollback("AfterCustomer");

                // Транзакция ещё активна — нужно явно закоммитить
                tran.Commit();
            }
        }
    }
    catch
    {
        tran.Rollback(); // полный откат при других ошибках
        throw;
    }
}
```

**Нюанс:** после `Rollback("point")` транзакция остаётся активной. Если не вызвать `Commit()`, изменения до savepoint будут потеряны при закрытии соединения.

### 5.4. Вложенность и `TransactionScope`

`SqlTransaction` не поддерживает настоящую вложенность — второй `BeginTransaction()` на том же соединении выбросит исключение. Для распределённых транзакций используется `TransactionScope`, но в .NET Core на Linux он не поддерживается (нужен Windows + MSDTC).

```csharp
using System.Transactions;

// Ambient-транзакция: команды внутри scope автоматически в неё входят
using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    using (var conn = new SqlConnection(DbConfig.ConnectionString))
    {
        conn.Open();
        using (var cmd = new SqlCommand("INSERT INTO Logs (Message) VALUES (@m)", conn))
        {
            cmd.Parameters.Add("@m", SqlDbType.NVarChar, 200).Value = "test";
            cmd.ExecuteNonQuery();
        }
    }
    scope.Complete(); // без Complete транзакция откатится при Dispose
}
```

**Нюанс:** `TransactionScope` с несколькими соединениями/ресурсами повышает транзакцию до распределённой (MSDTC). В консольном приложении на Linux это не сработает — используйте `SqlTransaction` в рамках одного соединения.
---

## 4. Асинхронные транзакции: Коммит и Откат

Транзакции в ADO.NET также имеют асинхронные версии (`BeginTransactionAsync`, `CommitAsync`, `RollbackAsync`). Главный подводный камень здесь — **откат при отмене**.

Если вы передали `CancellationToken` в `CommitAsync`, и он был отменен, транзакция может остаться в неопределенном состоянии. Microsoft рекомендует использовать **отдельный токен** для очистки (отката), так как основной токен уже может быть «мертв» .

```csharp
public static async Task PerformTransactionAsync(CancellationToken userToken)
{
    // Создаем отдельный токен для очистки (Rollback), 
    // который не зависит от userToken и даст нам время завершить откат.
    using var cleanupCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
    
    await using var connection = new SqlConnection(ConnectionString);
    await connection.OpenAsync(userToken);

    // Начинаем транзакцию асинхронно.
    await using var transaction = (SqlTransaction)await connection.BeginTransactionAsync(userToken);

    try
    {
        await using var command = new SqlCommand(
            "INSERT INTO Person.Person (FirstName, LastName) VALUES ('Test', 'User')", 
            connection, transaction);

        // Выполняем команду в рамках транзакции.
        await command.ExecuteNonQueryAsync(userToken);

        // Если пользователь отменил операцию, токен выбросит исключение здесь или выше.
        // Коммитим.
        await transaction.CommitAsync(userToken);
    }
    catch (Exception ex)
    {
        // Если произошла ошибка (включая отмену), пытаемся откатить.
        try
        {
            // Используем cleanupCts.Token, чтобы откат не прервался из-за отмены userToken.
            await transaction.RollbackAsync(cleanupCts.Token);
        }
        catch (Exception rollbackEx)
        {
            // Если откат тоже упал, оборачиваем оба исключения в AggregateException.
            throw new AggregateException("Ошибка операции и ошибка отката", ex, rollbackEx);
        }
        
        // Пробрасываем исходное исключение дальше, если откат прошел успешно.
        throw;
    }
}
```

**Нюанс:** В некоторых реализациях (например, в старых версиях `Microsoft.Data.SqlClient` или при использовании провайдеров, отличных от SQL Server) методы `CommitAsync`/`RollbackAsync` могут быть просто синхронными обертками, возвращающими завершенную задачу. Это значит, что они не освобождают поток во время выполнения, а просто проверяют токен перед началом. Это нормально, но полезно знать .

---

## 5. Массовое копирование (SqlBulkCopy) асинхронно

`SqlBulkCopy` — критически важный класс для загрузки больших объемов данных. У него есть метод `WriteToServerAsync`, но есть важное ограничение: **вы не можете использовать один и тот же экземпляр `SqlBulkCopy` для параллельных операций**. Дождитесь завершения одной, прежде чем начинать другую .

```csharp
public static async Task BulkInsertAsync(DataTable dataTable, CancellationToken token)
{
    await using var destinationConnection = new SqlConnection(ConnectionString);
    await destinationConnection.OpenAsync(token);

    // SqlBulkCopy не имеет асинхронного конструктора, только методы.
    using var bulkCopy = new SqlBulkCopy(destinationConnection)
    {
        DestinationTableName = "dbo.MyTargetTable",
        // Размер батча влияет на потребление памяти и количество round-trip'ов к серверу.
        BatchSize = 5000,
        BulkCopyTimeout = 60
    };

    // Настраиваем маппинг колонок, если имена в DataTable отличаются от имен в БД.
    // bulkCopy.ColumnMappings.Add("SourceColumn", "DestinationColumn");

    try
    {
        // WriteToServerAsync копирует данные асинхронно.
        await bulkCopy.WriteToServerAsync(dataTable, token);
    }
    catch (Exception ex)
    {
        // Обработка ошибок. Если токен отменен, будет OperationCanceledException.
        Console.WriteLine($"Ошибка массового копирования: {ex.Message}");
        throw;
    }
}
```

---

## 6. Ключевые подводные камни (Checklist)

1.  **Не используйте `using` (синхронный) для `SqlConnection` в асинхронном коде.** Используйте `await using` (или `IAsyncDisposable`). Это гарантирует, что ресурсы (сетевое соединение) будут освобождены асинхронно, не блокируя поток .
2.  **Один читатель на соединение.** Если вы не включили `MultipleActiveResultSets=True` в строке подключения, вы не можете выполнить вторую команду на том же соединении, пока открыт `SqlDataReader`. Попытка сделать это вызовет исключение.
3.  **Пул соединений.** Асинхронное открытие (`OpenAsync`) также учитывает пул. Если пул исчерпан, `OpenAsync` будет ждать освобождения соединения асинхронно (не блокируя поток), пока не истечет `Connect Timeout` .
4.  **ConfigureAwait(false).** В библиотеках (DAL) и серверном коде (ASP.NET Core) рекомендуется использовать `.ConfigureAwait(false)` после каждого `await` в методах доступа к данным. Это предотвращает захват контекста синхронизации (хотя в ASP.NET Core его нет, привычка полезна для переносимости кода в другие среды) .
5.  **Не смешивайте `async void`.** Единственное допустимое место для `async void` — обработчики событий (например, `button_Click`). В остальных случаях используйте `async Task`, чтобы иметь возможность дождаться завершения и перехватить исключения.

---

## 7. Итоговый пример: Сценарий с MARS и параллельным чтением

Если вам действительно нужно выполнять несколько запросов одновременно на одном соединении, используйте MARS. Это не делает код быстрее, но позволяет эффективнее использовать одно соединение.

```csharp
public static async Task ReadMultipleSetsAsync(CancellationToken token)
{
    // В строке подключения обязательно должно быть MultipleActiveResultSets=True
    const string marsConnectionString = ConnectionString + ";MultipleActiveResultSets=True;";
    
    await using var connection = new SqlConnection(marsConnectionString);
    await connection.OpenAsync(token);

    // Запускаем две команды. Благодаря MARS они могут выполняться на одном соединении.
    await using var command1 = new SqlCommand("SELECT TOP 5 * FROM Person.Person", connection);
    await using var command2 = new SqlCommand("SELECT TOP 5 * FROM HumanResources.Employee", connection);

    // Получаем два reader'а одновременно.
    await using SqlDataReader reader1 = await command1.ExecuteReaderAsync(token);
    await using SqlDataReader reader2 = await command2.ExecuteReaderAsync(token);

    // Читаем из первого, затем из второго. 
    // В реальном сценарии вы могли бы чередовать чтение, если бы это имело смысл.
    while (await reader1.ReadAsync(token))
    {
        Console.WriteLine($"Person: {reader1.GetString(1)}");
    }

    while (await reader2.ReadAsync(token))
    {
        Console.WriteLine($"Employee ID: {reader2.GetInt32(0)}");
    }
}
```

**Важное предупреждение по MARS:** Хотя MARS позволяет иметь несколько активных reader'ов, он не является серебряной пулей. Он может снижать производительность из-за дополнительных накладных расходов на сервере и клиенте. Используйте его только тогда, когда это действительно необходимо (например, при потоковой передаче данных из двух источников в одном запросе), а не для простого распараллеливания запросов — для этого лучше открыть два отдельных соединения из пула.

---


# 2 Часть Более подробный разбор
# Асинхронная работа в ADO.NET: углублённое руководство (расширенная версия)

Продолжая предыдущий материал, углубимся в темы, которые чаще всего остаются «за кадром»: как именно ADO.NET достигает асинхронности на уровне операционной системы, почему `ReadAsync` и `GetFieldValueAsync` ведут себя по-разному в зависимости от режима доступа, что происходит с пулом соединений при отмене, и какие ловушки подстерегают при работе с транзакциями и MARS.


## 1. Как ADO.NET достигает «настоящей» асинхронности (а не просто блокировки другого потока)

Ключевое отличие асинхронных методов ADO.NET от наивной эмуляции через `Task.Run` заключается в том, что они **не занимают поток пула на время ожидания ввода-вывода**.

До появления встроенной асинхронности разработчики часто «симулировали» неблокирующее выполнение с помощью асинхронных делегатов или `ThreadPool`. Однако такие решения просто блокировали **другой** фоновый поток, что сводило на нет смысл асинхронности при высокой конкуренции .

ADO.NET (начиная с версии 2.0 и далее в Task-based API) использует **настоящий асинхронный сетевой ввод-вывод** под капотом. На Windows это реализуется через механизм **I/O Completion Ports (IOCP)** и overlapped I/O. Это означает, что при выполнении `ExecuteReaderAsync` или `ReadAsync` **ни один поток не блокируется в ожидании ответа от сервера**. Один или несколько потоков обрабатывают завершения всех выдающихся запросов процесса .

**Практическое следствие:** если вы выполняете 1000 параллельных запросов к базе данных через `ExecuteReaderAsync`, вы не исчерпаете пул потоков (thread pool), даже если сервер отвечает медленно. Потоки освобождаются сразу после того, как запрос отправлен, и возвращаются в пул для обработки других задач. Именно поэтому асинхронный ADO.NET критически важен для масштабируемости веб-приложений и сервисов.

**Важное ограничение:** асинхронность не ускоряет сам запрос. Если SQL Server выполняет тяжёлый `JOIN` 10 секунд, клиентское приложение всё равно будет ждать 10 секунд. Разница лишь в том, что клиентский поток в это время свободен и может обслуживать другие запросы.


## 2. Эволюция API: почему `BeginExecuteReader` больше не нужен

В старых версиях .NET Framework (до 4.5) асинхронные операции в ADO.NET реализовывались через паттерн **APM** (Asynchronous Programming Model): методы `BeginExecuteReader` / `EndExecuteReader`, `BeginExecuteNonQuery` / `EndExecuteNonQuery` и т.д. .

Этот паттерн требовал передачи `AsyncCallback` и ручной обработки `IAsyncResult`. Он был громоздким и плохо сочетался с современными `async/await`.

**Начиная с .NET Framework 4.5 и во всех современных версиях (.NET Core, .NET 5+)** предпочтительным (и единственно доступным в .NET Core) является **Task-based Asynchronous Pattern (TAP)**: методы `ExecuteReaderAsync`, `ExecuteNonQueryAsync`, `OpenAsync` и т.д., возвращающие `Task` или `Task<T>` .

**Если вы поддерживаете устаревший код** с `BeginExecuteReader`, помните: методы `Begin*` в некоторых сценариях **не отправляли команду асинхронно**. В частности, согласно документации по реализации SqlClient, текст команды и параметры **отправлялись на сервер синхронно**, и только **чтение ответа** было асинхронным. Это означало, что при отправке большого пакета параметров или длинного текста команды метод `BeginExecute...` мог заблокировать поток на время записи в сеть . Современные `*Async` методы лишены этого недостатка.


## 3. `ReadAsync` и `GetFieldValueAsync`: когда они действительно нужны, а когда вредны

Это одна из самых неочевидных тем. Многие разработчики механически заменяют `Read()` на `ReadAsync()` и `GetString()` на `GetFieldValueAsync<string>()`, не понимая, что производительность может **ухудшиться**.

### 3.1 Режимы доступа: неsequential vs SequentialAccess

`SqlDataReader` может работать в двух режимах:

- **По умолчанию (неsequential):** при вызове `ReadAsync` (или `Read`) драйвер **считывает всю строку целиком** в буфер, даже если вы не собираетесь читать все колонки. Это удобно, но может привести к лишнему расходу памяти и сетевому трафику, если в строке есть большие поля (`varbinary(max)`, `varchar(max)`), которые вам не нужны.

- **`CommandBehavior.SequentialAccess`:** драйвер **не кэширует строку целиком**. Вы обязаны читать колонки **строго по порядку**, и данные подгружаются по мере продвижения. Это позволяет обрабатывать огромные строки с минимальным потреблением памяти .

### 3.2 Рекомендации по использованию асинхронных методов чтения

Согласно официальным рекомендациям Microsoft (в интерпретации Daniel Paoliello и сообщества):

1. **`ReadAsync` — используйте всегда.** В неsequential режиме он асинхронно считывает все данные строки, которые могут занимать несколько сетевых пакетов. В sequential режиме он асинхронно дочитывает текущую строку и токены TDS между строками .

2. **`IsDBNullAsync` и `GetFieldValueAsync` в sequential режиме — НЕ используйте.** В sequential режиме данные уже находятся в буфере после `ReadAsync`. Создание `Task` для обёртки значения только добавляет накладные расходы без какой-либо выгоды .

3. **`IsDBNullAsync` и `GetFieldValueAsync` в неsequential режиме — ситуативно.** Если вы уже вызвали `ReadAsync` (неsequential), данные всей строки уже в буфере. Синхронные `GetInt32`, `GetString` и `IsDBNull` будут **быстрее**, потому что не создают `Task` . Асинхронные версии имеют смысл только если вы **не** использовали `ReadAsync` или если работаете в sequential режиме с очень большими полями, которые ещё не полностью загружены.

4. **Для огромных полей (`varbinary(max)`, `varchar(max)`, XML) — используйте потоковые методы:** `GetStream()`, `GetTextReader()`, `GetXmlReader()`. Они позволяют обрабатывать данные по частям, не загружая мегабайты в управляемую память и не создавая давления на Large Object Heap (LOH) .

### 3.3 Пример: sequential access для обработки большого BLOB

```csharp
public static async Task ReadLargeBlobAsync(int id, CancellationToken token)
{
    await using var connection = new SqlConnection(ConnectionString);
    await connection.OpenAsync(token);

    await using var command = new SqlCommand(
        "SELECT Id, LargeData, Description FROM dbo.Documents WHERE Id = @Id", 
        connection);
    command.Parameters.AddWithValue("@Id", id);

    // Включаем SequentialAccess — данные не кэшируются целиком
    await using var reader = await command.ExecuteReaderAsync(
        CommandBehavior.SequentialAccess, token);

    if (await reader.ReadAsync(token))
    {
        // Колонки нужно читать строго по порядку!
        int docId = reader.GetInt32(0); // синхронно — данные уже в буфере

        // Для LargeData используем поток, чтобы не загружать всё в память
        await using var stream = reader.GetStream(1);
        
        // Копируем в файл или обрабатываем по частям
        await using var file = File.Create($"doc_{docId}.bin");
        await stream.CopyToAsync(file, token);
        
        // Description читаем после LargeData, потому что SequentialAccess требует порядок
        string description = await reader.GetFieldValueAsync<string>(2, token);
        Console.WriteLine($"Description: {description}");
    }
}
```


## 4. Отмена (CancellationToken): что происходит «под капотом» и где подстерегают deadlock'и

`CancellationToken` в ADO.NET — это не просто флаг, который проверяется между операциями. Он передаётся в драйвер и может **прервать сетевое ожидание**.

Однако здесь есть несколько важных нюансов, которые проявляются в реальных production-сценариях.

### 4.1 Отмена `OpenAsync` и ambient transactions — известный deadlock

Существует задокументированный баг в `Microsoft.Data.SqlClient` (воспроизводится на версиях 6.1.6, 7.0.3 и main), связанный с отменой `OpenAsync` при использовании `TransactionScope` и **отключённом пуле соединений** (`Pooling=false`).

Сценарий:
1. Приложение открывает соединение внутри `TransactionScope`.
2. `OpenAsync` получает `CancellationToken` и начинается процесс логина.
3. Токен отменяется во время логина.
4. Продолжение (continuation) отменённого `OpenAsync` пытается выполнить `Enlist` в ambient транзакцию, в то время как путь отката транзакции захватывает блокировку `lock(connection)`.
5. Возникает **взаимная блокировка** между parser lock соединения и блокировкой объекта транзакции .

**Практический вывод:** если вы используете `TransactionScope` и отменяемые операции открытия соединения, **включайте пулинг** (оставьте `Pooling=true`, значение по умолчанию). При пулинге путь `CompleteLogin` с `enlistOK` и `ConnectionOptions.Enlist` не выполняется для уже существующего соединения из пула, что снижает риск этого конкретного deadlock'а.

### 4.2 Отмена транзакций: нужен отдельный «cleanup» токен

Как упоминалось в предыдущем ответе, при откате транзакции после отмены основной операции **нельзя использовать тот же `CancellationToken`**, потому что он уже находится в состоянии «отменён».

Официальная рекомендация Microsoft для `Microsoft.Data.SqlClient`:

```csharp
await using SqlTransaction transaction = 
    (SqlTransaction)await connection.BeginTransactionAsync(cancellationToken);

using var command = new SqlCommand(sql, connection, transaction);

try
{
    await command.ExecuteNonQueryAsync(cancellationToken);
    await transaction.CommitAsync(cancellationToken);
}
catch (Exception operationException)
{
    try
    {
        // ВАЖНО: CancellationToken.None или отдельный cleanup-токен
        await transaction.RollbackAsync(CancellationToken.None);
    }
    catch (Exception rollbackException)
    {
        throw new AggregateException(
            "The operation and transaction rollback both failed.",
            operationException, rollbackException);
    }
}
```

Обратите внимание: методы `CommitAsync` и `RollbackAsync` в `Microsoft.Data.SqlClient` фактически являются **обёртками над синхронными методами**, которые возвращают уже завершённый `Task`. Они проверяют токен **перед** началом операции, но **не прерывают** уже выполняющуюся транзакционную операцию . Это важно понимать: отмена `CommitAsync` не «откатит» коммит на полпути.


## 5. Пул соединений: скрытый источник проблем в асинхронном коде

Пул соединений (connection pool) — это невидимый посредник между вашим `SqlConnection` и реальным физическим соединением с SQL Server. В асинхронном контексте пул ведёт себя особым образом.

### 5.1 Как пул влияет на `OpenAsync`

Когда вы вызываете `OpenAsync`:

1. Если в пуле есть свободное соединение, оно возвращается **немедленно** (без реального сетевого вызова). `Task` завершается синхронно.
2. Если свободных соединений нет, но не достигнут `Max Pool Size`, создаётся новое физическое соединение — это включает реальный сетевой ввод-вывод, и `OpenAsync` действительно асинхронен.
3. Если достигнут `Max Pool Size` и все соединения заняты, `OpenAsync` **асинхронно ждёт** освобождения соединения. Если ожидание превышает `Connect Timeout` (по умолчанию 15 секунд), выбрасывается исключение .

**Важно для асинхронных приложений:** параметр `Connect Timeout` в строке подключения — это **не** таймаут на установку TCP-соединения. Это **общий** таймаут на получение соединения из пула, включая время ожидания в очереди.

### 5.2 Ключи пула и фрагментация

Каждая уникальная строка подключения создаёт **отдельный пул**. Даже незначительные различия (например, `Application Name=MyApp-v1` vs `Application Name=MyApp-v2`) приводят к созданию второго пула. В асинхронном приложении с высокой конкуренцией фрагментация пулов может привести к исчерпанию соединений на сервере задолго до достижения `Max Pool Size` в каждом отдельном пуле.

Рекомендация: создайте **одну каноническую строку подключения** и используйте её везде. Избегайте динамических значений в `Application Name`, `Workstation ID` и подобных ключах .

### 5.3 Период блокировки аутентификации

Если аутентификация не удалась (неверный пароль, проблема с firewall, временная недоступность), пул переходит в **период блокировки** (blocking period). В течение этого периода все попытки открыть соединение с теми же параметрами **немедленно выбрасывают закэшированное исключение** без повторной попытки аутентификации.

Первый период блокировки — 5 секунд. При повторной неудаче он удваивается до максимума в 1 минуту. Это поведение управляется параметром `Pool Blocking Period` (`Auto` по умолчанию) .

**Асинхронный нюанс:** в асинхронном приложении с высокой конкуренцией, если пул вошёл в период блокировки, сотни одновременных `OpenAsync` немедленно завершатся ошибкой, даже если проблема уже устранена. Это может выглядеть как «каскадный отказ». Если вы уверены, что проблема устранена, можно вызвать `SqlConnection.ClearPool()` для сброса состояния пула.


## 6. MARS (Multiple Active Result Sets): когда он нужен и почему он не «серебряная пуля»

MARS позволяет иметь **несколько активных `SqlDataReader` на одном соединении**. Без MARS попытка выполнить вторую команду, пока первый reader не закрыт, вызывает исключение: `There is already an open DataReader associated with this Connection`.

### 6.1 Как работает MARS

При включении `MultipleActiveResultSets=True` каждый объект `SqlCommand`, выполняемый на соединении, создаёт **логическую сессию** внутри одного физического соединения. Каждая сессия имеет собственную копию окружения выполнения (SET-опции, контекст безопасности, контекст базы данных и т.д.) .

`SqlClient` кэширует до **10 MARS-сессий** на соединение. Если нужно больше — создаётся новая сессия (без ошибки). Кэш очищается только при закрытии соединения .

### 6.2 Критические ограничения MARS

1. **MARS не обеспечивает параллельного выполнения на сервере.** Операторы выполняются **синхронно** на сервере. `SELECT` и `BULK INSERT` могут чередоваться. Но DML (INSERT/UPDATE/DELETE) и DDL выполняются **атомарно**: если один batch выполняет DML, другие batch'и **блокируются** до его завершения .

2. **`WAITFOR` внутри `SELECT` блокирует всё соединение.** Пока `WAITFOR` ждёт, никакие другие batch'и на этом соединении выполняться не могут .

3. **MARS не является thread-safe.** Вы не можете одновременно обращаться к одному и тому же MARS-соединению из разных потоков .

4. **Соединения с MARS и без него находятся в разных пулах.** Даже если строка подключения отличается только `MultipleActiveResultSets=True/False`, это два разных пула .

### 6.3 Когда MARS действительно оправдан

MARS стоит использовать, когда вам **действительно нужно** чередовать чтение из двух источников в рамках одной логической операции (например, потоковая передача данных из двух таблиц в одном отчёте). Для простого распараллеливания запросов **лучше открыть два отдельных соединения из пула** — это даст настоящее параллельное выполнение на сервере.

Пример с осторожным использованием MARS:

```csharp
public static async Task StreamTwoTablesAsync(CancellationToken token)
{
    // MARS включён в строке подключения
    const string marsConnStr = ConnectionString + ";MultipleActiveResultSets=True;";
    
    await using var connection = new SqlConnection(marsConnStr);
    await connection.OpenAsync(token);

    await using var cmd1 = new SqlCommand(
        "SELECT Id, Name FROM dbo.Categories ORDER BY Id", connection);
    await using var cmd2 = new SqlCommand(
        "SELECT Id, CategoryId, ProductName FROM dbo.Products ORDER BY CategoryId, Id", 
        connection);

    // Два reader'а на одном соединении — возможно только с MARS
    await using var catReader = await cmd1.ExecuteReaderAsync(token);
    await using var prodReader = await cmd2.ExecuteReaderAsync(token);

    // Чередуем чтение (в реальности порядок может быть сложнее)
    while (await catReader.ReadAsync(token))
    {
        int catId = catReader.GetInt32(0);
        string catName = catReader.GetString(1);
        Console.WriteLine($"Category: {catName}");

        // Для каждой категории читаем соответствующие продукты из второго reader'а
        // ВНИМАНИЕ: это работает, потому что оба reader'а на одном MARS-соединении
        while (await prodReader.ReadAsync(token) && prodReader.GetInt32(1) == catId)
        {
            Console.WriteLine($"  Product: {prodReader.GetString(2)}");
        }
    }
}
```

**Предупреждение:** приведённый выше код имеет логическую проблему — второй reader не «перематывается» назад, поэтому для следующей категории продукты могут быть уже прочитаны. В реальном коде для такого сценария лучше использовать один запрос с `JOIN` или буферизовать данные. Пример лишь демонстрирует **механику** MARS.


## 7. `ConfigureAwait(false)` в библиотеках доступа к данным: зачем и когда

В библиотеках (DAL, репозиториях), которые не являются UI-кодом, после каждого `await` следует использовать `.ConfigureAwait(false)`.

**Причина:** без `ConfigureAwait(false)` продолжение после `await` будет запланировано в **исходном контексте синхронизации** (например, в UI-потоке или в контексте ASP.NET Classic). Это означает, что поток UI будет вынужден обрабатывать продолжение, даже если оно не имеет отношения к UI. В ASP.NET Core `SynchronizationContext` отсутствует, поэтому эффекта нет, но для переносимости кода (например, если библиотека будет использоваться в WPF или ASP.NET Classic) `ConfigureAwait(false)` обязателен.

```csharp
// Правильно для библиотечного кода
public static async Task<List<Person>> GetPeopleAsync(CancellationToken token)
{
    var result = new List<Person>();
    
    await using var connection = new SqlConnection(ConnectionString);
    await connection.OpenAsync(token).ConfigureAwait(false);
    
    await using var command = new SqlCommand("SELECT ...", connection);
    await using var reader = await command.ExecuteReaderAsync(token).ConfigureAwait(false);
    
    while (await reader.ReadAsync(token).ConfigureAwait(false))
    {
        result.Add(new Person
        {
            Id = reader.GetInt32(0),
            Name = await reader.GetFieldValueAsync<string>(1, token).ConfigureAwait(false)
        });
    }
    
    return result;
}
```

**Исключение:** в UI-коде (обработчики событий, код, который обновляет элементы управления после `await`), **не** используйте `ConfigureAwait(false)`, иначе вы попытаетесь обновить UI из фонового потока и получите исключение.


## 8. Итоговый чеклист для production-кода

| Ситуация | Правило |
|---|---|
| **Соединение** | Всегда `await using` (не `using`) для `SqlConnection`, `SqlDataReader`, `SqlCommand`, `SqlTransaction` |
| **Открытие** | `OpenAsync(token)` всегда. Не забывайте про `Connect Timeout` — это таймаут пула, не TCP |
| **Чтение строк** | `ReadAsync` — всегда. В sequential режиме не используйте `GetFieldValueAsync` для маленьких полей |
| **Большие поля** | `GetStream` / `GetTextReader` / `GetXmlReader` вместо `GetFieldValueAsync<byte[]>` |
| **Транзакции** | `RollbackAsync(CancellationToken.None)` или отдельный cleanup-токен. Не используйте отменённый токен для отката |
| **MARS** | Только если действительно нужно чередование. Для параллелизма — отдельные соединения |
| **Пул** | Одна каноническая строка подключения. Не меняйте `Application Name` без необходимости |
| **Библиотечный код** | `.ConfigureAwait(false)` после каждого `await` |
| **Отмена** | `CancellationTokenSource.CreateLinkedTokenSource` для связывания пользовательской отмены с внутренним таймаутом |


## 9. Сводный пример: асинхронный DAL с обработкой всех нюансов

```csharp
using System;
using System.Collections.Generic;
using System.Data;
using Microsoft.Data.SqlClient;
using System.Threading;
using System.Threading.Tasks;

public sealed class AsyncPersonRepository
{
    private readonly string _connectionString;

    public AsyncPersonRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    /// <summary>
    /// Асинхронно получает список персон с поддержкой отмены, таймаута и потокового чтения.
    /// </summary>
    public async Task<List<Person>> GetPeopleAsync(
        int maxRows, 
        CancellationToken userToken)
    {
        var people = new List<Person>(maxRows);
        
        // Связываем пользовательский токен с внутренним таймаутом на 30 секунд
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(userToken);
        cts.CancelAfter(TimeSpan.FromSeconds(30));
        var token = cts.Token;

        // await using для асинхронного освобождения ресурсов
        await using var connection = new SqlConnection(_connectionString);
        
        // ConfigureAwait(false) — библиотечный код не должен захватывать контекст
        await connection.OpenAsync(token).ConfigureAwait(false);

        await using var command = new SqlCommand(
            @"SELECT TOP (@MaxRows) 
                BusinessEntityID, FirstName, LastName, ModifiedDate 
              FROM Person.Person 
              ORDER BY BusinessEntityID",
            connection);
        
        command.Parameters.Add("@MaxRows", SqlDbType.Int).Value = maxRows;
        command.CommandTimeout = 20; // Внутренний таймаут драйвера

        // SequentialAccess для потенциально больших полей (здесь не нужен, но показан для примера)
        // Обычно для простых SELECT лучше default (неsequential)
        await using var reader = await command
            .ExecuteReaderAsync(CommandBehavior.Default, token)
            .ConfigureAwait(false);

        while (await reader.ReadAsync(token).ConfigureAwait(false))
        {
            // После ReadAsync в неsequential режиме данные уже в буфере — синхронные геттеры быстрее
            people.Add(new Person
            {
                Id = reader.GetInt32(0),
                FirstName = reader.GetString(1),
                LastName = reader.GetString(2),
                ModifiedDate = reader.GetDateTime(3)
            });
        }

        return people;
    }
}

public class Person
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime ModifiedDate { get; set; }
}
```

Этот пример демонстрирует: связывание токенов, `await using`, `ConfigureAwait(false)`, синхронные геттеры после `ReadAsync` (оптимально для маленьких полей) и разделение `CommandTimeout` (драйвер) и `CancelAfter` (приложение).
