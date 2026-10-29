# Полный практический гайд по Dapper + PostgreSQL

Dapper — это микро-ORM: он не пишет SQL за вас и не отслеживает изменения объектов, а только выполняет ваш SQL и маппит результат в C#-объекты. В связке с PostgreSQL вы получаете скорость, близкую к чистому ADO.NET, плюс полный контроль над запросами. PostgreSQL добавляет свои особенности — массивы, JSONB, `RETURNING`, схемы, snake_case — и именно на них мы сделаем упор.

---

## 1. Установка и подключение

```bash
dotnet add package Dapper
dotnet add package Npgsql
```

Dapper не содержит драйверов БД. Работа с PostgreSQL целиком обеспечивается **Npgsql** — официальным ADO.NET-провайдером.

### Строка подключения

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Postgres": "Host=localhost;Port=5432;Database=myapp;Username=postgres;Password=secret"
  }
}
```

### Фабрика соединений (правильный способ)

Никогда не храните `NpgsqlConnection` в `static` поле и не переиспользуйте один экземпляр между запросами. Пул соединений уже решает задачу переиспользования — вам достаточно создавать новый объект на каждый запрос .

```csharp
public interface IDbConnectionFactory
{
    NpgsqlConnection Create();
}

public class PostgresConnectionFactory : IDbConnectionFactory
{
    private readonly string _connectionString;

    public PostgresConnectionFactory(IConfiguration config)
    {
        _connectionString = config.GetConnectionString("Postgres")!;
    }

    public NpgsqlConnection Create() => new(_connectionString);
}

// Program.cs
builder.Services.AddSingleton<IDbConnectionFactory, PostgresConnectionFactory>();
```

Стандартное использование:

```csharp
using var connection = _factory.Create();
var users = await connection.QueryAsync<User>("SELECT * FROM users");
// using закрывает соединение → оно возвращается в пул
```

> **Важно:** Dapper не открывает и не закрывает соединение сам, если вы передаёте уже открытое. При передаче закрытого соединения Dapper откроет его, выполнит запрос и закроет обратно — но лучше управлять этим явно через `using`.

---

## 2. Критичная настройка: snake_case → PascalCase

PostgreSQL приводит незакавыченные идентификаторы к нижнему регистру: `UserName` превращается в `user_name`. Dapper по умолчанию ищет точное совпадение имён, поэтому свойство `UserName` не заполнится из колонки `user_name`.

**Решение — выполнить один раз при старте приложения:**

```csharp
// Program.cs
Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true;
```

После этого `user_name` → `UserName`, `created_at` → `CreatedAt`, `project_id` → `ProjectId` работают автоматически.

Если настройку включить нельзя (например, код в библиотеке), используйте алиасы:

```sql
SELECT user_name AS "UserName", created_at AS "CreatedAt" FROM users
```

> Кавычки обязательны, иначе PostgreSQL снова приведёт алиас к нижнему регистру.

---

## 3. Чтение данных

### 3.1. Список записей

```csharp
public class User
{
    public int Id { get; set; }
    public string UserName { get; set; } = "";
    public string Email { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}

// Простой список
var users = (await connection.QueryAsync<User>(
    "SELECT id, user_name, email, created_at FROM users ORDER BY id"
)).ToList();

// С фильтром через параметр (никогда не конкатенируйте строки!)
var activeUsers = await connection.QueryAsync<User>(
    "SELECT * FROM users WHERE is_active = @IsActive",
    new { IsActive = true }
);
```

### 3.2. Одна запись

```csharp
// Вернёт null, если ничего не найдено — самый частый вариант
var user = await connection.QueryFirstOrDefaultAsync<User>(
    "SELECT * FROM users WHERE id = @Id",
    new { Id = 5 }
);

if (user is null)
{
    // обрабатываем отсутствие
}

// Упадёт с исключением, если записи нет — когда отсутствие является ошибкой
var required = await connection.QuerySingleAsync<User>(
    "SELECT * FROM users WHERE id = @Id",
    new { Id = 5 }
);

// Упадёт, если вернулось больше одной строки — защита от дублей
var unique = await connection.QuerySingleOrDefaultAsync<User>(
    "SELECT * FROM users WHERE email = @Email",
    new { Email = "a@b.com" }
);
```

**Разница методов:**

| Метод | Нет строк | Больше одной строки |
|---|---|---|
| `QueryFirst` | исключение | берёт первую |
| `QueryFirstOrDefault` | `null` | берёт первую |
| `QuerySingle` | исключение | исключение |
| `QuerySingleOrDefault` | `null` | исключение |

### 3.3. Скалярные значения

```csharp
// Количество
int count = await connection.ExecuteScalarAsync<int>(
    "SELECT COUNT(*) FROM users WHERE is_active = @Active",
    new { Active = true }
);

// Одно значение из строки
string? email = await connection.QueryFirstOrDefaultAsync<string>(
    "SELECT email FROM users WHERE id = @Id",
    new { Id = 5 }
);
```

### 3.4. Динамические результаты

Когда структура результата заранее неизвестна (например, произвольный `SELECT` из админки):

```csharp
var rows = await connection.QueryAsync(
    "SELECT id, user_name FROM users LIMIT 10"
);

foreach (IDictionary<string, object> row in rows)
{
    Console.WriteLine($"{row["id"]} — {row["user_name"]}");
}
```

> Динамика медленнее типизированных запросов и не даёт автодополнения — используйте только там, где без неё нельзя.

### 3.5. Чтение нескольких результатов одним запросом

`QueryMultiple` позволяет выполнить несколько `SELECT` за один round-trip:

```csharp
using var multi = await connection.QueryMultipleAsync(@"
    SELECT COUNT(*) FROM users;
    SELECT * FROM users ORDER BY id LIMIT 10;
    SELECT * FROM orders WHERE user_id = @UserId;
", new { UserId = 5 });

var total = await multi.ReadSingleAsync<int>();
var users = (await multi.ReadAsync<User>()).ToList();
var orders = (await multi.ReadAsync<Order>()).ToList();
```

Результаты читаются строго в том порядке, в котором идут `SELECT`.


## 4. Изменение данных

Все операции `INSERT` / `UPDATE` / `DELETE` выполняются через `ExecuteAsync`, который возвращает число затронутых строк.

### 4.1. INSERT

```csharp
var rows = await connection.ExecuteAsync(
    @"INSERT INTO users (user_name, email, is_active)
      VALUES (@UserName, @Email, @IsActive)",
    new { UserName = "bob", Email = "bob@test.com", IsActive = true }
);
```

### 4.2. INSERT с возвратом сгенерированного ID (`RETURNING`)

Это одно из главных преимуществ PostgreSQL: `RETURNING` позволяет получить вставленную строку в том же запросе, без второго обращения к БД.

```csharp
// Возврат одного поля
int newId = await connection.ExecuteScalarAsync<int>(
    @"INSERT INTO users (user_name, email)
      VALUES (@UserName, @Email)
      RETURNING id",
    new { UserName = "alice", Email = "alice@test.com" }
);

// Возврат всей вставленной строки
var created = await connection.QuerySingleAsync<User>(
    @"INSERT INTO users (user_name, email)
      VALUES (@UserName, @Email)
      RETURNING id, user_name, email, created_at",
    new { UserName = "carol", Email = "carol@test.com" }
);
```

### 4.3. UPDATE с RETURNING

```csharp
var updated = await connection.QuerySingleOrDefaultAsync<User>(
    @"UPDATE users
      SET email = @Email, user_name = @UserName
      WHERE id = @Id
      RETURNING *",
    new { Id = 5, Email = "new@test.com", UserName = "newname" }
);

if (updated is null)
{
    // записи с таким id не было
}
```

### 4.4. DELETE с RETURNING

```csharp
var deleted = await connection.QuerySingleOrDefaultAsync<User>(
    "DELETE FROM users WHERE id = @Id RETURNING *",
    new { Id = 5 }
);
```

### 4.5. UPDATE ... FROM (UPDATE с join)

PostgreSQL поддерживает обновление на основе другой таблицы:

```csharp
await connection.ExecuteAsync(@"
    UPDATE orders o
    SET status = 'shipped'
    FROM shipments s
    WHERE s.order_id = o.id
      AND s.shipped_at IS NOT NULL
      AND o.status = 'pending'
");
```

### 4.6. UPSERT (INSERT ... ON CONFLICT)

```csharp
var result = await connection.QuerySingleAsync<User>(@"
    INSERT INTO users (id, user_name, email)
    VALUES (@Id, @UserName, @Email)
    ON CONFLICT (id) DO UPDATE
    SET user_name = EXCLUDED.user_name,
        email     = EXCLUDED.email
    RETURNING *",
    new { Id = 5, UserName = "bob", Email = "bob@test.com" }
);
```

`EXCLUDED` — это псевдотаблица с данными, которые пытались вставить.

---

## 5. Параметры: все ситуации

Dapper передаёт параметры через Npgsql, поэтому синтаксис — стандартный `@Name`.

### 5.1. Анонимный объект

```csharp
await connection.ExecuteAsync(
    "UPDATE users SET email = @Email WHERE id = @Id",
    new { Id = 5, Email = "x@y.com" }
);
```

### 5.2. Класс

```csharp
var user = new User { Id = 5, Email = "x@y.com" };
await connection.ExecuteAsync(
    "UPDATE users SET email = @Email WHERE id = @Id",
    user  // Dapper возьмёт свойства объекта
);
```

### 5.3. `DynamicParameters` — для output-параметров и точного контроля

```csharp
var p = new DynamicParameters();
p.Add("@UserName", "bob");
p.Add("@Email", "bob@test.com");
p.Add("@NewId", dbType: DbType.Int32, direction: ParameterDirection.Output);

await connection.ExecuteAsync(
    "sp_create_user", p, commandType: CommandType.StoredProcedure);

int id = p.Get<int>("@NewId");
```

### 5.4. IN-список (`= ANY(@Ids)`)

В PostgreSQL правильный способ передать список — массив + `ANY`, а не генерация `IN (@p1, @p2, ...)`:

```csharp
var ids = new[] { 1, 2, 3, 4 };

var users = await connection.QueryAsync<User>(
    "SELECT * FROM users WHERE id = ANY(@Ids)",
    new { Ids = ids }
);
```

Для `string[]`:

```csharp
var emails = new[] { "a@b.com", "c@d.com" };
var users = await connection.QueryAsync<User>(
    "SELECT * FROM users WHERE email = ANY(@Emails)",
    new { Emails = emails }
);
```

> Npgsql сам превращает `int[]` → `integer[]`, `string[]` → `text[]`. Никаких ручных преобразований не нужно.

---

## 6. Работа с массивами PostgreSQL

Массивы — одна из ключевых фич PostgreSQL, и Npgsql маппит их напрямую в .NET-массивы.

### 6.1. Чтение массива

```csharp
public class Article
{
    public int ArticleId { get; set; }
    public string Title { get; set; } = "";
    public string[] Tags { get; set; } = Array.Empty<string>();
    public int[] ViewCounts { get; set; } = Array.Empty<int>();
}

var articles = await connection.QueryAsync<Article>(
    "SELECT article_id, title, tags, view_counts FROM articles"
);
// tags → string[], view_counts → int[]
```

### 6.2. Запись массива

```csharp
await connection.ExecuteAsync(
    @"INSERT INTO articles (title, tags, view_counts)
      VALUES (@Title, @Tags, @ViewCounts)",
    new
    {
        Title = "Dapper + PostgreSQL",
        Tags = new[] { "csharp", "dapper", "postgres" },
        ViewCounts = new[] { 0, 0, 0 }
    }
);
```

### 6.3. Поиск по массиву

```csharp
// Есть ли тег в массиве
var tagged = await connection.QueryAsync<Article>(
    "SELECT * FROM articles WHERE @Tag = ANY(tags)",
    new { Tag = "dotnet" }
);

// Массив содержит все переданные значения (оператор @>)
var allTags = await connection.QueryAsync<Article>(
    "SELECT * FROM articles WHERE tags @> @Tags",
    new { Tags = new[] { "csharp", "postgres" } }
);

// Пересечение массивов (оператор &&)
var anyTags = await connection.QueryAsync<Article>(
    "SELECT * FROM articles WHERE tags && @Tags",
    new { Tags = new[] { "csharp", "python" } }
);
```

### 6.4. `List<T>` вместо массива

Npgsql также принимает `List<T>`, но при чтении в свойство лучше объявлять именно `T[]` — маппинг массивов в коллекции надёжнее.

---

## 7. Работа с JSONB

JSONB — бинарный JSON с индексацией. Npgsql умеет маппить JSONB в строку, `JsonDocument`, `JsonElement` и пользовательские типы через `System.Text.Json`.

### 7.1. Чтение JSONB как строки

```csharp
public class Event
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Payload { get; set; } = ""; // JSONB как строка
}

var events = await connection.QueryAsync<Event>(
    "SELECT id, name, payload FROM events"
);
```

### 7.2. Чтение JSONB как `JsonDocument`

```csharp
public class Event
{
    public int Id { get; set; }
    public JsonDocument Payload { get; set; } = null!;
}

var ev = await connection.QuerySingleAsync<Event>(
    "SELECT id, payload FROM events WHERE id = @Id",
    new { Id = 1 }
);

string name = ev.Payload.RootElement.GetProperty("name").GetString()!;
```

### 7.3. Запись JSONB

```csharp
var payload = JsonSerializer.Serialize(new { name = "click", x = 100, y = 200 });

await connection.ExecuteAsync(
    "INSERT INTO events (name, payload) VALUES (@Name, @Payload::jsonb)",
    new { Name = "user_click", Payload = payload }
);
```

> Приведение `@Payload::jsonb` обязательно, если Npgsql передаёт параметр как `text`. Иначе PostgreSQL не сможет неявно преобразовать тип при вставке в jsonb-колонку.

### 7.4. Запросы внутри JSONB

```csharp
// Фильтр по полю JSONB
var clicks = await connection.QueryAsync<Event>(
    "SELECT * FROM events WHERE payload->>'name' = @Name",
    new { Name = "click" }
);

// Числовое сравнение (->> возвращает text, приводим)
var bigClicks = await connection.QueryAsync<Event>(
    "SELECT * FROM events WHERE (payload->>'x')::int > @MinX",
    new { MinX = 50 }
);

// Проверка наличия ключа (оператор ?)
var withX = await connection.QueryAsync<Event>(
    "SELECT * FROM events WHERE payload ? 'x'"
);
```

### 7.5. Обновление JSONB

```csharp
// Добавить/перезаписать ключ
await connection.ExecuteAsync(@"
    UPDATE events
    SET payload = payload || @Patch::jsonb
    WHERE id = @Id",
    new { Id = 1, Patch = "{\"processed\": true}" }
);

// Удалить ключ
await connection.ExecuteAsync(@"
    UPDATE events
    SET payload = payload - 'x'
    WHERE id = @Id",
    new { Id = 1 }
);
```

---

## 8. `DateOnly` и `TimeOnly` (.NET 6+)

Npgsql 6+ напрямую маппит `DateOnly` ↔ `date` и `TimeOnly` ↔ `time`:

```csharp
public class Booking
{
    public int Id { get; set; }
    public DateOnly BookingDate { get; set; }
    public TimeOnly StartTime { get; set; }
}

var bookings = await connection.QueryAsync<Booking>(
    "SELECT id, booking_date, start_time FROM bookings WHERE booking_date >= @From",
    new { From = DateOnly.FromDateTime(DateTime.Today) }
);
```

`DateTime` с `Kind=Utc` ↔ `timestamptz`, `DateTime` с `Kind=Unspecified` ↔ `timestamp`. Npgsql 6+ строго различает эти типы, и несовпадение приводит к исключению. Это сделано намеренно — чтобы не терять информацию о часовом поясе.

---

## 9. NULL и nullable-типы

```csharp
public class User
{
    public int Id { get; set; }
    public string? MiddleName { get; set; }  // NULL → null
    public DateTime? LastLoginAt { get; set; } // NULL → null
}
```

Для value-типов, которые могут быть NULL, используйте `T?`:

```csharp
var age = await connection.QueryFirstOrDefaultAsync<int?>(
    "SELECT age FROM users WHERE id = @Id",
    new { Id = 5 }
);

if (age is null) { /* возраст не указан */ }
```

### Передача NULL в параметр

```csharp
await connection.ExecuteAsync(
    "UPDATE users SET middle_name = @MiddleName WHERE id = @Id",
    new { Id = 5, MiddleName = (string?)null }
);
```

> Без приведения `(string?)null` компилятор может не понять, какой тип у анонимного объекта, и Npgsql попытается передать `unknown`.

---

## 10. Связи: один-к-одному и много-к-одному

Когда запрос возвращает колонки двух таблиц, используйте **Multi-Mapping**:

```csharp
public class User
{
    public int Id { get; set; }
    public string UserName { get; set; } = "";
    public Address? Address { get; set; }
}

public class Address
{
    public int UserId { get; set; }
    public string City { get; set; } = "";
    public string Street { get; set; } = "";
}

string sql = @"
    SELECT u.id, u.user_name,
           a.user_id, a.city, a.street
    FROM users u
    LEFT JOIN addresses a ON a.user_id = u.id
    WHERE u.id = @Id";

var user = await connection.QueryAsync<User, Address, User>(
    sql,
    (u, a) =>
    {
        u.Address = a;   // если a == null, адрес останется null
        return u;
    },
    new { Id = 5 },
    splitOn: "user_id"   // с этой колонки начинается второй объект
).FirstOrDefault();
```

**Как работает `splitOn`:** Dapper читает колонки по порядку. Всё до `user_id` идёт в первый тип (`User`), начиная с `user_id` — во второй (`Address`). Значение `splitOn` — **имя первой колонки второго объекта** в результирующем наборе (то, как оно пришло из SQL, то есть `user_id`, не `UserId`).

> С `LEFT JOIN` второй объект может быть `null` — Dapper вернёт `null` для ссылочного типа, и это надо учесть в лямбде.

---

## 11. Связи: один-ко-многим

Dapper не собирает коллекции автоматически. Для `Project` → `Tasks` нужна ручная группировка через словарь:

```csharp
public class Project
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<TaskItem> Tasks { get; set; } = new();
}

public class TaskItem
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public int ProjectId { get; set; }
}

string sql = @"
    SELECT p.id, p.name,
           t.id, t.title, t.project_id
    FROM projects p
    LEFT JOIN tasks t ON t.project_id = p.id
    WHERE p.id = @Id
    ORDER BY p.id, t.id";

var dict = new Dictionary<int, Project>();

await connection.QueryAsync<Project, TaskItem, Project>(
    sql,
    (project, task) =>
    {
        if (!dict.TryGetValue(project.Id, out var current))
        {
            current = project;
            current.Tasks = new List<TaskItem>();
            dict.Add(current.Id, current);
        }

        // При LEFT JOIN, если задач нет, task придёт с Id = 0
        if (task is not null && task.Id != 0)
            current.Tasks.Add(task);

        return current;
    },
    new { Id = 1 },
    splitOn: "id"  // вторая колонка t.id — начало TaskItem
);

var result = dict.Values.FirstOrDefault();
```

> `splitOn: "id"` здесь работает, потому что первая колонка второго объекта — `t.id`. Порядок колонок в `SELECT` критичен: сначала все колонки первого типа, потом второго.

---

## 12. Хранимые процедуры и функции

PostgreSQL использует `CALL` для процедур и `SELECT * FROM function()` для функций.

### Процедура

```csharp
await connection.ExecuteAsync(
    "CALL sp_create_user(@UserName, @Email)",
    new { UserName = "bob", Email = "bob@test.com" },
    commandType: CommandType.StoredProcedure
);
```

### Функция, возвращающая таблицу

```csharp
var users = await connection.QueryAsync<User>(
    "SELECT * FROM get_active_users(@MinAge)",
    new { MinAge = 18 }
);
```

### Функция с OUT-параметром (через `DynamicParameters`)

```csharp
var p = new DynamicParameters();
p.Add("@Input", 10);
p.Add("@Result", dbType: DbType.Int32, direction: ParameterDirection.Output);

await connection.ExecuteAsync(
    "CALL calculate_something(@Input, @Result)",
    p,
    commandType: CommandType.StoredProcedure);

int result = p.Get<int>("@Result");
```

> Npgsql поддерживает `ParameterDirection.Output` только для процедур (`CALL`), которые объявляют `OUT`-параметры. Для функций чаще возвращают `RETURNS TABLE` и читают через `Query`.

---

## 13. Транзакции

Несколько операций «всё или ничего» выполняются через `NpgsqlTransaction`:

```csharp
using var connection = _factory.Create();
await connection.OpenAsync();

await using var tx = await connection.BeginTransactionAsync();

try
{
    await connection.ExecuteAsync(
        "UPDATE accounts SET balance = balance - @Amount WHERE id = @From",
        new { Amount = 100, From = 1 }, transaction: tx);

    await connection.ExecuteAsync(
        "UPDATE accounts SET balance = balance + @Amount WHERE id = @To",
        new { Amount = 100, To = 2 }, transaction: tx);

    await tx.CommitAsync();
}
catch
{
    await tx.RollbackAsync();
    throw;
}
```

> **Обязательно** передавайте `transaction: tx` в каждый вызов. Если забыть — операция выполнится вне транзакции, и откат её не затронет.

### Уровень изоляции

```csharp
await using var tx = await connection.BeginTransactionAsync(
    IsolationLevel.Serializable);
```

PostgreSQL поддерживает `ReadCommitted` (по умолчанию), `RepeatableRead`, `Serializable`, `ReadUncommitted` (ведёт себя как `ReadCommitted`).

### Savepoint (частичный откат)

```csharp
await using var tx = await connection.BeginTransactionAsync();

await connection.ExecuteAsync("INSERT INTO logs(msg) VALUES ('start')", transaction: tx);

await tx.SaveAsync("before_risky");

try
{
    await connection.ExecuteAsync("...рискованная операция...", transaction: tx);
}
catch
{
    await tx.RollbackAsync("before_risky"); // откат только до savepoint
}

await tx.CommitAsync();
```

---

## 14. Массовые операции

Стандартный Dapper делает один `INSERT` за вызов. Для больших объёмов это медленно.

### 14.1. `unnest` — вставка массива за один запрос

Самый эффективный способ без дополнительных библиотек:

```csharp
var users = new[]
{
    new { UserName = "a", Email = "a@x.com" },
    new { UserName = "b", Email = "b@x.com" },
    new { UserName = "c", Email = "c@x.com" }
};

await connection.ExecuteAsync(@"
    INSERT INTO users (user_name, email)
    SELECT * FROM unnest(@UserNames, @Emails)",
    new
    {
        UserNames = users.Select(u => u.UserName).ToArray(),
        Emails = users.Select(u => u.Email).ToArray()
    }
);
```

### 14.2. `COPY` через Npgsql (максимальная скорость)

Для десятков тысяч строк используйте бинарный `COPY`:

```csharp
await using var writer = await connection.BeginBinaryImportAsync(
    "COPY users (user_name, email) FROM STDIN (FORMAT BINARY)");

foreach (var u in users)
{
    await writer.StartRowAsync();
    await writer.WriteAsync(u.UserName);
    await writer.WriteAsync(u.Email);
}

await writer.CompleteAsync();
```

`COPY` в разы быстрее `INSERT` для больших объёмов, но не поддерживает `ON CONFLICT` и `RETURNING`.

### 14.3. Dapper Plus (коммерческая библиотека)

Если нужен именно `BulkInsert` из коробки — есть Dapper Plus (платная с ограничениями). Для большинства задач `unnest` или `COPY` достаточно.

---

## 15. Пагинация

PostgreSQL: `LIMIT` / `OFFSET`.

```csharp
public record Page<T>(IReadOnlyList<T> Items, int Total);

public async Task<Page<User>> GetPageAsync(int page, int pageSize)
{
    var offset = (page - 1) * pageSize;

    var sql = @"
        SELECT COUNT(*) FROM users;
        SELECT * FROM users ORDER BY id LIMIT @Limit OFFSET @Offset;";

    using var multi = await connection.QueryMultipleAsync(sql,
        new { Limit = pageSize, Offset = offset });

    var total = await multi.ReadSingleAsync<int>();
    var items = (await multi.ReadAsync<User>()).ToList();

    return new Page<User>(items, total);
}
```

> `OFFSET` работает плохо на больших значениях (PostgreSQL всё равно читает и отбрасывает строки). Для больших таблиц используйте keyset-пагинацию:
> ```sql
> SELECT * FROM users WHERE id > @LastId ORDER BY id LIMIT @Limit
> ```

---

## 16. Схемы (schemas)

Если таблица лежит не в `public`, указывайте схему явно:

```csharp
var users = await connection.QueryAsync<User>(
    "SELECT * FROM app.users WHERE id = @Id",
    new { Id = 5 }
);
```

Проверьте `search_path` при подключении, если хотите опускать схему:

```csharp
// В строке подключения
"Host=...;Search Path=app,public"
```

---

## 17. `RETURNING` и optimistic concurrency

`RETURNING` удобен для проверки, что `UPDATE` действительно затронул строку:

```csharp
// Возвращает обновлённую строку или null, если WHERE не нашёл запись
var updated = await connection.QuerySingleOrDefaultAsync<User>(
    @"UPDATE users
      SET email = @Email, version = version + 1
      WHERE id = @Id AND version = @Version
      RETURNING *",
    new { Id = 5, Email = "x@y.com", Version = 3 });

if (updated is null)
    throw new DbUpdateConcurrencyException("Запись изменена другим пользователем");
```

---

## 18. Асинхронность

Всегда используйте `*Async`-версии методов в веб-приложениях:

```csharp
var users = await connection.QueryAsync<User>("SELECT * FROM users");
var affected = await connection.ExecuteAsync("...", new { ... });
var user = await connection.QueryFirstOrDefaultAsync<User>("...", new { ... });
```

> Метод `Query`/`Execute` без `Async` блокирует поток на время сетевого round-trip. В ASP.NET Core это приводит к исчерпанию пула потоков под нагрузкой.

---

## 19. Отладка и типичные ошибки

| Симптом | Причина | Решение |
|---|---|---|
| Свойство всегда `null` | Имена колонок snake_case, а свойства PascalCase | `MatchNamesWithUnderscores = true` |
| Ошибка типа при вставке в JSONB | Npgsql передаёт `text` | Приведение `@Param::jsonb` |
| `NpgsqlException: Column does not exist` | Незакавыченный алиас с заглавной буквой | `AS "UserName"` в кавычках |
| `The connection is already in a transaction` | Забыли передать `transaction` в вызов | Передавать `transaction: tx` везде |
| Медленный `OFFSET` на больших данных | PostgreSQL читает все отброшенные строки | Keyset-пагинация по `id` |
| `42804: column ... is of type ... but expression is of type ...` | Несовпадение типов параметра | Явное приведение `@P::type` |
| `DateTime` с `Kind=Utc` падает на `timestamp` | Npgsql 6+ различает `timestamp`/`timestamptz` | Использовать `timestamptz` или `DateTime.SpecifyKind` |


## 20. Практические советы

1. **Пишите SQL руками** — это главное преимущество Dapper. Не пытайтесь генерировать запросы «как EF».
2. **Всегда параметризуйте** — никогда не конкатенируйте значения в SQL.
3. **`RETURNING` вместо второго `SELECT`** — один round-trip вместо двух.
4. **`unnest`/`COPY` для массовых вставок** — не делайте `foreach` + `ExecuteAsync`.
5. **`QueryMultiple` для связанных запросов** — экономит round-trip.
6. **`using var connection`** — обязательно. Dapper не управляет жизненным циклом соединения.
7. **`MatchNamesWithUnderscores = true`** — один раз при старте, иначе будете писать алиасы везде.
8. **`transaction: tx` в каждом вызове внутри транзакции** — иначе операция уйдёт вне транзакции.
9. **Keyset-пагинация для больших таблиц** — `OFFSET` масштабируется плохо.
10. **`COPY` для миграций и импорта** — в разы быстрее `INSERT`.


## 21. Проект для отработки

Ниже — полноценный учебный проект: **мини-трекер задач** (проекты → задачи → теги) с консольным меню, который покрывает все темы гайда.

### 21.1. Структура

```
TaskTracker/
├── TaskTracker.csproj
├── Program.cs
├── appsettings.json
├── Data/
│   ├── IDbConnectionFactory.cs
│   ├── PostgresConnectionFactory.cs
│   └── Schema.sql
├── Models/
│   ├── Project.cs
│   ├── TaskItem.cs
│   └── User.cs
└── Repositories/
    ├── ProjectRepository.cs
    ├── TaskRepository.cs
    └── UserRepository.cs
```

### 21.2. `TaskTracker.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Dapper" Version="2.1.35" />
    <PackageReference Include="Npgsql" Version="8.0.3" />
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="8.0.0" />
  </ItemGroup>
  <ItemGroup>
    <None Update="appsettings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
  </ItemGroup>
</Project>
```

### 21.3. `appsettings.json`

```json
{
  "ConnectionStrings": {
    "Postgres": "Host=localhost;Port=5432;Database=tasktracker;Username=postgres;Password=secret"
  }
}
```

### 21.4. `Data/Schema.sql` — выполните один раз в psql или DBeaver

```sql
CREATE TABLE IF NOT EXISTS users (
    id          SERIAL PRIMARY KEY,
    user_name   TEXT NOT NULL UNIQUE,
    email       TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS projects (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    owner_id    INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tags        TEXT[] NOT NULL DEFAULT '{}',
    settings    JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS tasks (
    id          SERIAL PRIMARY KEY,
    project_id  INT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    title       TEXT NOT NULL,
    is_done     BOOLEAN NOT NULL DEFAULT false,
    due_date    DATE,
    metadata    JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_tasks_project ON tasks(project_id);
CREATE INDEX IF NOT EXISTS idx_projects_tags ON projects USING GIN(tags);
CREATE INDEX IF NOT EXISTS idx_projects_settings ON projects USING GIN(settings);
```

### 21.5. `Data/IDbConnectionFactory.cs` и `PostgresConnectionFactory.cs`

```csharp
using Npgsql;

namespace TaskTracker.Data;

public interface IDbConnectionFactory
{
    NpgsqlConnection Create();
}

public class PostgresConnectionFactory : IDbConnectionFactory
{
    private readonly string _connectionString;

    public PostgresConnectionFactory(string connectionString)
    {
        _connectionString = connectionString;
    }

    public NpgsqlConnection Create() => new(_connectionString);
}
```

### 21.6. Модели

```csharp
namespace TaskTracker.Models;

public class User
{
    public int Id { get; set; }
    public string UserName { get; set; } = "";
    public string Email { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}

public class Project
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public int OwnerId { get; set; }
    public string[] Tags { get; set; } = Array.Empty<string>();
    public string Settings { get; set; } = "{}";  // JSONB как строка
    public DateTime CreatedAt { get; set; }
    public List<TaskItem> Tasks { get; set; } = new();
}

public class TaskItem
{
    public int Id { get; set; }
    public int ProjectId { get; set; }
    public string Title { get; set; } = "";
    public bool IsDone { get; set; }
    public DateOnly? DueDate { get; set; }
    public string Metadata { get; set; } = "{}";
    public DateTime CreatedAt { get; set; }
}
```

### 21.7. `Repositories/UserRepository.cs`

```csharp
using Dapper;
using TaskTracker.Data;
using TaskTracker.Models;

namespace TaskTracker.Repositories;

public class UserRepository
{
    private readonly IDbConnectionFactory _factory;

    public UserRepository(IDbConnectionFactory factory) => _factory = factory;

    // INSERT ... RETURNING — один round-trip
    public async Task<User> CreateAsync(string userName, string email)
    {
        using var conn = _factory.Create();

        return await conn.QuerySingleAsync<User>(@"
            INSERT INTO users (user_name, email)
            VALUES (@UserName, @Email)
            RETURNING id, user_name, email, created_at",
            new { UserName = userName, Email = email });
    }

    public async Task<User?> GetByIdAsync(int id)
    {
        using var conn = _factory.Create();

        return await conn.QueryFirstOrDefaultAsync<User>(
            "SELECT * FROM users WHERE id = @Id",
            new { Id = id });
    }

    public async Task<List<User>> GetAllAsync()
    {
        using var conn = _factory.Create();

        return (await conn.QueryAsync<User>(
            "SELECT * FROM users ORDER BY id")).ToList();
    }

    // UPSERT через ON CONFLICT
    public async Task<User> UpsertAsync(User user)
    {
        using var conn = _factory.Create();

        return await conn.QuerySingleAsync<User>(@"
            INSERT INTO users (id, user_name, email)
            VALUES (@Id, @UserName, @Email)
            ON CONFLICT (id) DO UPDATE
            SET user_name = EXCLUDED.user_name,
                email     = EXCLUDED.email
            RETURNING *",
            user);
    }
}
```

### 21.8. `Repositories/ProjectRepository.cs`

```csharp
using Dapper;
using TaskTracker.Data;
using TaskTracker.Models;

namespace TaskTracker.Repositories;

public class ProjectRepository
{
    private readonly IDbConnectionFactory _factory;

    public ProjectRepository(IDbConnectionFactory factory) => _factory = factory;

    public async Task<Project> CreateAsync(
        string name, int ownerId, string[] tags, string settingsJson)
    {
        using var conn = _factory.Create();

        return await conn.QuerySingleAsync<Project>(@"
            INSERT INTO projects (name, owner_id, tags, settings)
            VALUES (@Name, @OwnerId, @Tags, @Settings::jsonb)
            RETURNING *",
            new { Name = name, OwnerId = ownerId, Tags = tags, Settings = settingsJson });
    }

    public async Task<List<Project>> GetByTagAsync(string tag)
    {
        using var conn = _factory.Create();

        // Поиск по массиву через ANY
        return (await conn.QueryAsync<Project>(
            "SELECT * FROM projects WHERE @Tag = ANY(tags) ORDER BY id",
            new { Tag = tag })).ToList();
    }

    // JOIN проекта с задачами — один-ко-многим с ручной группировкой
    public async Task<Project?> GetWithTasksAsync(int projectId)
    {
        using var conn = _factory.Create();

        var dict = new Dictionary<int, Project>();

        await conn.QueryAsync<Project, TaskItem, Project>(@"
            SELECT p.id, p.name, p.owner_id, p.tags, p.settings, p.created_at,
                   t.id, t.project_id, t.title, t.is_done, t.due_date, t.metadata, t.created_at
            FROM projects p
            LEFT JOIN tasks t ON t.project_id = p.id
            WHERE p.id = @Id
            ORDER BY p.id, t.id",
            (project, task) =>
            {
                if (!dict.TryGetValue(project.Id, out var current))
                {
                    current = project;
                    current.Tasks = new List<TaskItem>();
                    dict.Add(current.Id, current);
                }

                if (task is not null && task.Id != 0)
                    current.Tasks.Add(task);

                return current;
            },
            new { Id = projectId },
            splitOn: "id");

        return dict.Values.FirstOrDefault();
    }

    // Пагинация через QueryMultiple
    public async Task<(List<Project> Items, int Total)> GetPageAsync(int page, int pageSize)
    {
        using var conn = _factory.Create();
        var offset = (page - 1) * pageSize;

        using var multi = await conn.QueryMultipleAsync(@"
            SELECT COUNT(*) FROM projects;
            SELECT * FROM projects ORDER BY id LIMIT @Limit OFFSET @Offset;",
            new { Limit = pageSize, Offset = offset });

        var total = await multi.ReadSingleAsync<int>();
        var items = (await multi.ReadAsync<Project>()).ToList();

        return (items, total);
    }
}
```

### 21.9. `Repositories/TaskRepository.cs`

```csharp
using Dapper;
using TaskTracker.Data;
using TaskTracker.Models;

namespace TaskTracker.Repositories;

public class TaskRepository
{
    private readonly IDbConnectionFactory _factory;

    public TaskRepository(IDbConnectionFactory factory) => _factory = factory;

    public async Task<TaskItem> CreateAsync(
        int projectId, string title, DateOnly? dueDate, string metadataJson)
    {
        using var conn = _factory.Create();

        return await conn.QuerySingleAsync<TaskItem>(@"
            INSERT INTO tasks (project_id, title, due_date, metadata)
            VALUES (@ProjectId, @Title, @DueDate, @Metadata::jsonb)
            RETURNING *",
            new { ProjectId = projectId, Title = title, DueDate = dueDate, Metadata = metadataJson });
    }

    // Обновление с проверкой, что задача была найдена
    public async Task<TaskItem?> MarkDoneAsync(int taskId)
    {
        using var conn = _factory.Create();

        return await conn.QuerySingleOrDefaultAsync<TaskItem>(@"
            UPDATE tasks
            SET is_done = true
            WHERE id = @Id AND is_done = false
            RETURNING *",
            new { Id = taskId });
    }

    // Массовая вставка через unnest — один запрос вместо N
    public async Task BulkInsertAsync(int projectId, IEnumerable<string> titles)
    {
        using var conn = _factory.Create();
        var arr = titles.ToArray();

        await conn.ExecuteAsync(@"
            INSERT INTO tasks (project_id, title)
            SELECT @ProjectId, t FROM unnest(@Titles) AS t",
            new { ProjectId = projectId, Titles = arr });
    }

    // Удаление с RETURNING
    public async Task<TaskItem?> DeleteAsync(int taskId)
    {
        using var conn = _factory.Create();

        return await conn.QuerySingleOrDefaultAsync<TaskItem>(
            "DELETE FROM tasks WHERE id = @Id RETURNING *",
            new { Id = taskId });
    }

    // Транзакция: отметить несколько задач атомарно
    public async Task MarkManyDoneAsync(int projectId)
    {
        using var conn = _factory.Create();
        await conn.OpenAsync();
        await using var tx = await conn.BeginTransactionAsync();

        try
        {
            await conn.ExecuteAsync(@"
                UPDATE tasks
                SET is_done = true
                WHERE project_id = @ProjectId AND is_done = false",
                new { ProjectId = projectId },
                transaction: tx);

            await conn.ExecuteAsync(@"
                INSERT INTO projects (name, owner_id, settings)  -- фиктивная операция для демонстрации
                SELECT name || ' (archived)', owner_id, settings
                FROM projects WHERE id = @ProjectId",
                new { ProjectId = projectId },
                transaction: tx);

            await tx.CommitAsync();
        }
        catch
        {
            await tx.RollbackAsync();
            throw;
        }
    }
}
```

> В `MarkManyDoneAsync` вторая операция (`INSERT` с `SELECT`) — это демонстрация того, что транзакция может содержать несколько операций; в реальном коде её не было бы.

### 21.10. `Program.cs` — консольное меню

```csharp
using Microsoft.Extensions.Configuration;
using TaskTracker.Data;
using TaskTracker.Repositories;

// 1. Загрузка конфигурации
var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false)
    .Build();

// 2. Глобальная настройка маппинга snake_case
Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true;

// 3. Фабрика соединений
var factory = new PostgresConnectionFactory(
    config.GetConnectionString("Postgres")!);

var users = new UserRepository(factory);
var projects = new ProjectRepository(factory);
var tasks = new TaskRepository(factory);

// 4. Проверка подключения
try
{
    using var conn = factory.Create();
    await conn.QueryAsync("SELECT 1");
    Console.WriteLine("Подключение к PostgreSQL успешно.");
}
catch (Exception ex)
{
    Console.WriteLine($"Ошибка подключения: {ex.Message}");
    return;
}

// 5. Меню
while (true)
{
    Console.WriteLine("""
        1. Создать пользователя
        2. Показать всех пользователей
        3. Создать проект
        4. Найти проект по тегу
        5. Показать проект с задачами
        6. Создать задачу
        7. Массово создать задачи
        8. Отметить задачу выполненной
        9. Показать страницу проектов (пагинация)
        0. Выход
        """);

    Console.Write("> ");
    var choice = Console.ReadLine();

    try
    {
        switch (choice)
        {
            case "1":
                Console.Write("Имя пользователя: ");
                var userName = Console.ReadLine()!;
                Console.Write("Email: ");
                var email = Console.ReadLine()!;
                var created = await users.CreateAsync(userName, email);
                Console.WriteLine($"Создан пользователь {created.Id}: {created.UserName}");
                break;

            case "2":
                var allUsers = await users.GetAllAsync();
                foreach (var u in allUsers)
                    Console.WriteLine($"{u.Id}. {u.UserName} <{u.Email}>");
                break;

            case "3":
                Console.Write("Название проекта: ");
                var projectName = Console.ReadLine()!;
                Console.Write("ID владельца: ");
                var ownerId = int.Parse(Console.ReadLine()!);
                Console.Write("Теги через запятую: ");
                var tags = Console.ReadLine()!.Split(',', StringSplitOptions.RemoveEmptyEntries);
                var project = await projects.CreateAsync(
                    projectName, ownerId, tags, "{\"theme\":\"dark\"}");
                Console.WriteLine($"Создан проект {project.Id}: {project.Name} [{string.Join(", ", project.Tags)}]");
                break;

            case "4":
                Console.Write("Тег: ");
                var tag = Console.ReadLine()!;
                var found = await projects.GetByTagAsync(tag);
                foreach (var p in found)
                    Console.WriteLine($"{p.Id}. {p.Name}");
                break;

            case "5":
                Console.Write("ID проекта: ");
                var pid = int.Parse(Console.ReadLine()!);
                var withTasks = await projects.GetWithTasksAsync(pid);
                if (withTasks is null)
                {
                    Console.WriteLine("Проект не найден.");
                }
                else
                {
                    Console.WriteLine($"Проект: {withTasks.Name}");
                    foreach (var t in withTasks.Tasks)
                        Console.WriteLine($"  {(t.IsDone ? "✓" : "·")} {t.Id}. {t.Title}");
                }
                break;

            case "6":
                Console.Write("ID проекта: ");
                var pid2 = int.Parse(Console.ReadLine()!);
                Console.Write("Заголовок: ");
                var title = Console.ReadLine()!;
                var task = await tasks.CreateAsync(pid2, title, null, "{\"priority\":\"normal\"}");
                Console.WriteLine($"Создана задача {task.Id}");
                break;

            case "7":
                Console.Write("ID проекта: ");
                var pid3 = int.Parse(Console.ReadLine()!);
                Console.Write("Заголовки через запятую: ");
                var titles = Console.ReadLine()!.Split(',', StringSplitOptions.RemoveEmptyEntries);
                await tasks.BulkInsertAsync(pid3, titles);
                Console.WriteLine($"Вставлено {titles.Length} задач.");
                break;

            case "8":
                Console.Write("ID задачи: ");
                var tid = int.Parse(Console.ReadLine()!);
                var done = await tasks.MarkDoneAsync(tid);
                Console.WriteLine(done is null
                    ? "Задача не найдена или уже выполнена."
                    : $"Задача {done.Id} отмечена выполненной.");
                break;

            case "9":
                Console.Write("Страница: ");
                var page = int.Parse(Console.ReadLine()!);
                var (items, total) = await projects.GetPageAsync(page, 5);
                Console.WriteLine($"Всего: {total}");
                foreach (var p in items)
                    Console.WriteLine($"{p.Id}. {p.Name}");
                break;

            case "0":
                return;

            default:
                Console.WriteLine("Неизвестная команда.");
                break;
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Ошибка: {ex.Message}");
    }

    Console.WriteLine();
}
```

### 21.11. Что отрабатывает проект

| Тема гайда | Где в проекте |
|---|---|
| `MatchNamesWithUnderscores` | `Program.cs`, шаг 2 |
| `INSERT ... RETURNING` | `UserRepository.CreateAsync`, `ProjectRepository.CreateAsync`, `TaskRepository.CreateAsync` |
| `QueryFirstOrDefault` / `QuerySingle` | `GetByIdAsync`, `CreateAsync` |
| `IN`-список через `ANY` | `ProjectRepository.GetByTagAsync` |
| Массивы (`TEXT[]`) | `tags` в `projects`, `BulkInsertAsync` |
| JSONB + `::jsonb` | `settings`, `metadata` |
| `ON CONFLICT` (UPSERT) | `UserRepository.UpsertAsync` |
| `DELETE ... RETURNING` | `TaskRepository.DeleteAsync` |
| Multi-mapping (один-ко-многим) | `ProjectRepository.GetWithTasksAsync` |
| Транзакция + rollback | `TaskRepository.MarkManyDoneAsync` |
| `unnest` для массовой вставки | `TaskRepository.BulkInsertAsync` |
| `QueryMultiple` + пагинация | `ProjectRepository.GetPageAsync` |
| `DateOnly?` | `tasks.due_date` |
| NULL и nullable | `due_date`, `metadata` (всегда not null) |

### 21.12. Как запустить

1. Создайте базу: `createdb tasktracker` (или через `psql -c "CREATE DATABASE tasktracker"`).
2. Выполните `Data/Schema.sql`.
3. Поправьте строку подключения в `appsettings.json`.
4. `dotnet run`.
5. Пройдите по пунктам меню: сначала создайте пользователя, затем проект, затем задачи, затем проверьте `GetWithTasksAsync`, `GetByTagAsync`, `GetPageAsync`, `MarkManyDoneAsync`.

### 21.13. Задания для самостоятельной отработки

1. Добавьте в `TaskRepository` метод `GetOverdueAsync()` — задачи с `due_date < CURRENT_DATE` и `is_done = false`. Используйте `QueryAsync<TaskItem>`.
2. Добавьте в `ProjectRepository` метод `UpdateSettingsAsync(int id, string jsonPatch)` — обновление JSONB через `settings || @Patch::jsonb`.
3. Реализуйте `GetPageKeysetAsync(int lastId, int limit)` — пагинацию по `id > @LastId`.
4. Замените `BulkInsertAsync` на `BeginBinaryImportAsync` (COPY) и сравните скорость на 100 000 задач.
5. Добавьте `GetWithTasksAsync` для нескольких проектов сразу (один запрос, группировка в словарь) — попрактикуйтесь с `splitOn` и порядком колонок.
6. Оберните `CreateAsync` проекта в транзакцию, которая также создаёт первую задачу «Kickoff».
