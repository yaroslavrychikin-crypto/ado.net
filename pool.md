### 1. Базовая настройка пула в строке подключения

Пул в Npgsql управляется параметрами строки подключения. Каждая уникальная строка подключения создаёт **отдельный пул**. Это значит, что если вы меняете `Maximum Pool Size` или другие параметры, вы создаёте новый пул, а не изменяете старый.

**Основные параметры пула:**

| Параметр | Значение по умолчанию | Описание |
| :--- | :--- | :--- |
| `Pooling` | `true` | Включает или выключает пулирование. |
| `Minimum Pool Size` | `1` (или `0`) | Минимальное количество соединений, которые поддерживаются открытыми. Npgsql предварительно выделяет указанное число подключений к серверу . |
| `Maximum Pool Size` | `100` | Максимальное количество соединений в пуле. При превышении запросы встают в очередь ожидания. Допустимый диапазон: 0–1024 . |
| `Connection Idle Lifetime` | `300` (сек) | Через сколько секунд простоя соединение может быть закрыто. |
| `Connection Lifetime` | `0` (бесконечно) | Максимальное время жизни соединения (полезно для балансировки). |

**Пример строки подключения с настройками пула:**

```csharp
// Строка подключения с явными настройками пула.
// Обратите внимание: если вы измените хотя бы один параметр (например, Maximum Pool Size),
// Npgsql создаст НОВЫЙ пул для этого набора параметров.
var connectionString = "Host=localhost;Port=5432;Database=mydb;Username=postgres;Password=secret;" +
                       "Pooling=true;" +
                       "Minimum Pool Size=2;" +   // Держим 2 соединения "горячими"
                       "Maximum Pool Size=20;" +  // Не более 20 одновременных соединений
                       "Connection Idle Lifetime=60;"; // Закрывать простаивающие соединения через минуту
```

### 2. Современный подход: `NpgsqlDataSource`

Начиная с Npgsql 7.0, рекомендуется использовать `NpgsqlDataSource` вместо прямого создания `NpgsqlConnection` со строкой подключения. `NpgsqlDataSource` инкапсулирует пул и является потокобезопасным объектом, который следует создавать один раз на всё приложение.

**Пример с `NpgsqlDataSource`:**

```csharp
using Npgsql;

// 1. Создаём DataSource ОДИН РАЗ при старте приложения (например, в Program.cs или Startup).
//    Он управляет пулом соединений.
var dataSourceBuilder = new NpgsqlDataSourceBuilder(
    "Host=localhost;Database=mydb;Username=postgres;Password=secret;Maximum Pool Size=50");
await using var dataSource = dataSourceBuilder.Build();

// 2. В любом месте приложения (в сервисе, репозитории) открываем соединение из этого пула.
//    Когда соединение закрывается (dispose), оно возвращается в пул, а не закрывается физически.
await using (var connection = await dataSource.OpenConnectionAsync())
{
    await using (var cmd = new NpgsqlCommand("SELECT current_database()", connection))
    {
        var dbName = await cmd.ExecuteScalarAsync();
        Console.WriteLine($"Connected to: {dbName}");
    }
} // Здесь соединение возвращается в пул.

// 3. Повторное открытие происходит очень быстро, так как используется уже существующее физическое соединение.
await using (var connection = await dataSource.OpenConnectionAsync())
{
    await using (var cmd = new NpgsqlCommand("SELECT 1", connection))
    {
        await cmd.ExecuteNonQueryAsync();
    }
}
```

**Важно:** Не создавайте новый `NpgsqlDataSource` на каждый запрос. Это приведёт к созданию множества мелких пулов и быстрому исчерпанию лимита соединений на сервере PostgreSQL.

### 3. Работа с PgBouncer (внешний пулер) — три режима и их последствия

Если вы используете PgBouncer (например, в Azure Database for PostgreSQL или на своём сервере), нужно учитывать взаимодействие двух пулов: внутреннего (Npgsql) и внешнего (PgBouncer). Официальная документация Npgsql прямо указывает на необходимость настройки в зависимости от режима PgBouncer .

**Режимы PgBouncer и их влияние:**

| Режим PgBouncer | Что происходит | Рекомендация для Npgsql |
| :--- | :--- | :--- |
| **Session** | Клиент удерживает серверное соединение на всю сессию. | Можно оставить `Pooling=true`, но двойное пулирование обычно избыточно. |
| **Transaction** | Соединение возвращается в пул после завершения транзакции. | **Обязательно** `No Reset On Close=true`, иначе `DISCARD ALL` сломает состояние между транзакциями . |
| **Statement** | Соединение берётся на один запрос. | Аналогично Transaction: `No Reset On Close=true`. Мультизапросные транзакции не поддерживаются. |

**Пример строки подключения для работы с PgBouncer (режим transaction):**

```csharp
// Вариант 1: Полностью отключаем пул Npgsql, доверяя PgBouncer.
// Это самый простой и часто рекомендуемый способ.
var connStrWithPgBouncer = "Host=pgbouncer-host;Port=6432;Database=mydb;Username=user;Password=pass;" +
                           "Pooling=false;";

// Вариант 2: Оставляем пул Npgsql, но отключаем DISCARD ALL.
// Используйте это, только если понимаете, зачем вам внутренний пул.
var connStrWithPgBouncerAndInternalPool = "Host=pgbouncer-host;Port=6432;Database=mydb;Username=user;Password=pass;" +
                                          "Pooling=true;" +
                                          "No Reset On Close=true;" + // Отключает DISCARD ALL
                                          "Maximum Pool Size=10;";
```

**Дополнительный нюанс:** В режиме Transaction именованные подготовленные выражения (prepared statements) могут не работать, если они привязаны к конкретному соединению. Npgsql по умолчанию использует автоматическую подготовку; при работе с PgBouncer в transaction mode может потребоваться отключить её: `Max Auto Prepare=0` .

### 4. Обработка «битых» соединений из пула

Это критический нюанс. Npgsql **не проверяет** живость соединения, когда выдаёт его из пула, потому что такая проверка (round-trip к серверу) убила бы всю производительность пула. В исходном коде видно, что при извлечении из пула выполняется только проверка `IsBroken` и времени жизни, но не ping сервера . Это значит, что вы можете получить из пула соединение, которое уже разорвано (например, из-за сетевого сбоя, таймаута на сервере или idle-таймаута firewall).

**Решение:** Ваш код должен быть готов к обработке `NpgsqlException` (или `IOException`) при первом использовании соединения и уметь повторить операцию.

**Пример с повторной попыткой (Retry):**

```csharp
using Npgsql;

public async Task<string> GetDataWithRetryAsync(NpgsqlDataSource dataSource)
{
    // Количество попыток
    const int maxRetries = 3;
    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        try
        {
            // Пытаемся открыть соединение и выполнить запрос
            await using var connection = await dataSource.OpenConnectionAsync();
            await using var cmd = new NpgsqlCommand("SELECT 'Hello, Pool!'", connection);
            return (string)await cmd.ExecuteScalarAsync();
        }
        catch (NpgsqlException ex) when (attempt < maxRetries)
        {
            // Логируем ошибку (например, "Connection is broken")
            Console.WriteLine($"Attempt {attempt} failed: {ex.Message}. Retrying...");
            
            // Небольшая задержка перед повторной попыткой (экспоненциальная)
            await Task.Delay(TimeSpan.FromMilliseconds(100 * attempt));
        }
    }
    throw new InvalidOperationException("Failed to get data after multiple retries.");
}
```

### 5. Ситуация: пул исчерпан, но PostgreSQL показывает мало соединений

Это одна из самых коварных ситуаций. Вы получаете ошибку `The connection pool has been exhausted`, но при проверке `pg_stat_activity` на сервере видите всего 5–20 активных соединений. Как такое возможно?

**Причины:**
1. **Утечка соединений в клиентском коде.** Соединение взято из пула, но не возвращено (не вызван `Dispose`/`Close`). Npgsql считает его занятым, хотя на сервере оно может простаивать.
2. **Долгие транзакции или запросы.** Соединение занято, пока выполняется запрос. Если запрос «висит» (например, из-за блокировки), пул исчерпывается, а сервер показывает соединение в состоянии `idle in transaction`.
3. **`Maximum Pool Size` на клиенте больше, чем реально нужно, но соединения «застряли» в ожидании.**

**Диагностика через метрики:** Npgsql предоставляет счётчики через `System.Diagnostics.Metrics`. Вы можете использовать `dotnet-counters` для быстрого просмотра состояния пула :

```bash
dotnet counters monitor --counters Npgsql -p <PID>
```

Вы увидите метрики вида `db.client.connection.count` (с разбивкой по состояниям idle/used) и `db.client.connection.max`. Если `count` в состоянии `used` близок к `max`, а на сервере соединений мало — ищите утечку в коде.

**Как дать пулу имя для удобной диагностики:**

```csharp
var builder = new NpgsqlDataSourceBuilder("Host=localhost;Username=test;Password=test")
{
    Name = "CustomersDB"  // Это имя будет отображаться в метриках
};
await using var dataSource = builder.Build();
```

### 6. `DISCARD ALL` и роли по умолчанию

По умолчанию Npgsql при возврате соединения в пул выполняет `DISCARD ALL` для сброса состояния сессии. Это может вызвать неожиданные проблемы, если вы используете **роли по умолчанию** (`ALTER ROLE ... SET ROLE ...`). `DISCARD ALL` сбрасывает текущую роль, и следующее использование этого соединения из пула будет выполняться уже от имени другой роли, что может привести к ошибкам прав доступа.

Если вы сталкиваетесь с такой ситуацией:
1. Используйте `SET ROLE` внутри каждой транзакции или сессии явно.
2. Настройте пул так, чтобы он не выполнял `DISCARD ALL` (`No Reset On Close=true`), если вы уверены, что состояние сессии не мешает.

### 7. Multiplexing (экспериментальная функция)

Npgsql 5.0 представил экспериментальную функцию **мультиплексирования**, которая меняет модель работы: пользовательский код больше не получает эксклюзивное физическое соединение из пула, а отправляет команды, которые выполняются на любом доступном соединении . Это может значительно снизить требуемое количество физических соединений и повысить пропускную способность.

**Ограничения и риски:**
- Функция считается экспериментальной.
- Не все сценарии поддерживаются (например, транзакции требуют эксклюзивного соединения на время транзакции).
- Включается явно: `Multiplexing=true` в строке подключения.

```csharp
// Включение мультиплексирования (экспериментально!)
var connStr = "Host=localhost;Database=mydb;Username=postgres;Password=secret;" +
              "Multiplexing=true;";
```

**Когда стоит рассмотреть:** Высоконагруженные сценарии с множеством коротких запросов, где стоимость выделения эксклюзивного соединения на каждый запрос становится заметной. Для типичных OLTP-приложений с транзакциями лучше остаться на классическом пуле.

### 8. Мониторинг состояния пула

Помимо `dotnet-counters`, вы можете программно подписаться на метрики Npgsql через `MeterListener`. Это полезно для интеграции с собственной системой мониторинга или для отладки в тестах.

```csharp
using System.Diagnostics.Metrics;

// Создаём MeterListener для прослушивания метрик Npgsql.
// В реальном приложении это обычно не требуется — используйте OpenTelemetry/Prometheus.
var listener = new MeterListener();
listener.InstrumentPublished = (instrument, l) =>
{
    // Фильтруем только метрики соединений из Npgsql
    if (instrument.Meter.Name == "Npgsql" && instrument.Name.Contains("connection"))
    {
        l.EnableMeasurementEvents(instrument);
    }
};
listener.SetMeasurementEventCallback<long>((inst, val, tags, state) =>
{
    // Выводим значение метрики и теги (например, pool.name, state)
    Console.WriteLine($"{inst.Name}: {val} [{string.Join(", ", tags.ToArray().Select(t => $"{t.Key}={t.Value}"))}]");
});
listener.Start();

// Ваш код работы с БД...
```

**Важно:** Метрики с именами `db.client.connection.*` появились в Npgsql 10.0. В более старых версиях используются Event Counters .

### 9. Тонкости таймаутов и `CommandTimeout`

При работе с пулом важно различать два таймаута:
- **Connection Timeout** (в строке подключения, по умолчанию 15 сек) — сколько ждать получения соединения из пула или установки физического соединения.
- **Command Timeout** (на `NpgsqlCommand`, по умолчанию 30 сек) — сколько ждать выполнения запроса.

Если пул исчерпан, вы получите `NpgsqlException` с сообщением о таймауте пула (а не `TimeoutException` от команды). Это важно для правильной обработки ошибок.

**Пример обработки таймаута пула:**

```csharp
try
{
    await using var conn = await dataSource.OpenConnectionAsync();
    // ...
}
catch (NpgsqlException ex) when (ex.InnerException is TimeoutException)
{
    // Это таймаут пула, а не таймаут команды
    Console.WriteLine("Connection pool exhausted. Consider raising Max Pool Size or investigating leaks.");
}
```

### Итоговые рекомендации

1. **Для стандартного приложения** используйте один экземпляр `NpgsqlDataSource`, созданный при старте, и настройте `Maximum Pool Size` исходя из лимитов PostgreSQL (`max_connections`) и количества экземпляров вашего приложения. Не создавайте DataSource на каждый запрос.
2. **Всегда обрабатывайте ошибки соединения** при работе с пулом. Получение «мёртвого» соединения — это нормальная ситуация в сетевой среде, а не баг.
3. **При использовании PgBouncer** отключите пул Npgsql (`Pooling=false`) или настройте `No Reset On Close=true` для режима транзакций. В transaction mode также рассмотрите `Max Auto Prepare=0`, если возникают проблемы с подготовленными выражениями .
4. **Если видите «пул исчерпан», а на сервере мало соединений** — ищите утечки в коде (недиспознутые соединения) или долгие транзакции. Используйте метрики `db.client.connection.count` для диагностики .
5. **Не создавайте строку подключения с уникальными параметрами** на каждый запрос, если в этом нет острой необходимости, чтобы не плодить лишние пулы.
