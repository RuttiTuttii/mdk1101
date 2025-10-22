Для выполнения этого задания необходимо разработать модель данных, настроить контекст базы данных, создать соответствующие таблицы и хранимые процедуры, а также реализовать методы для выполнения указанных SQL-команд с помощью Entity Framework Core. Ниже я приведу примерный план и образцы реализации для каждого пункта.

---

### 1. Модели данных и контекст базы данных

Создадим модели для фильмов, сеансов, посетителей и билетов:

```csharp
public class Movie
{
    public int Id { get; set; }
    public string Title { get; set; }
    public int Year { get; set; }
    public string Genre { get; set; }
    public ICollection<Session> Sessions { get; set; }
}

public class Session
{
    public int Id { get; set; }
    public int MovieId { get; set; }
    public DateTime DateTime { get; set; }
    public decimal Price { get; set; }
    public int HallId { get; set; }
    public Hall Hall { get; set; }
    public ICollection<Ticket> Tickets { get; set; }
}

public class Hall
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Session> Sessions { get; set; }
}

public class Ticket
{
    public int Id { get; set; }
    public int SessionId { get; set; }
    public int SeatNumber { get; set; }
    public string CustomerPhone { get; set; }
    public Session Session { get; set; }
}

public class Visitor
{
    public int Id { get; set; }
    public string Phone { get; set; }
    public string Name { get; set; }
}
```

Контекст:

```csharp
public class CinemaContext : DbContext
{
    public DbSet<Movie> Movies { get; set; }
    public DbSet<Session> Sessions { get; set; }
    public DbSet<Hall> Halls { get; set; }
    public DbSet<Ticket> Tickets { get; set; }
    public DbSet<Visitor> Visitors { get; set; }

    // Настройка соединения и модели...
}
```

---

### 2. Реализация методов

#### 5.1 Получение списка фильмов, отсортированных по выбранному столбцу

```csharp
public List<Movie> GetMoviesOrdered(string columnName)
{
    string sql = $"SELECT * FROM Movies ORDER BY {columnName}";
    return _context.Movies.FromSqlRaw(sql).ToList();
}
```

#### 5.2 Получение фильмов по названию и минимальному году выхода

```csharp
public List<Movie> GetMoviesByTitleAndYear(string title, int minYear)
{
    string sql = "SELECT * FROM Movies WHERE Title = @title AND Year >= @minYear";
    return _context.Movies.FromSqlRaw(sql, new SqlParameter("@title", title), new SqlParameter("@minYear", minYear)).ToList();
}
```

#### 5.3 Увеличение цены сеансов в определённом зале

```csharp
public void IncreaseSessionPrice(int hallId, decimal increment)
{
    string sql = "UPDATE Sessions SET Price = Price + @increment WHERE HallId = @hallId";
    int affectedRows = _context.Database.ExecuteSqlRaw(sql, new SqlParameter("@increment", increment), new SqlParameter("@hallId", hallId));
    Console.WriteLine($"{affectedRows} строк(а) обновлено(ы).");
}
```

#### 5.4 Получение жанров фильма по ID (SqlQuery)

```csharp
public List<string> GetGenresByMovieId(int movieId)
{
    string sql = "SELECT Genre FROM Movies WHERE Id = @movieId";
    return _context.Database.SqlQuery<string>(sql, new SqlParameter("@movieId", movieId)).ToList();
}
```

*(Обратите внимание, что для `SqlQuery` может потребоваться расширение или использование Dapper или другого метода, так как EF Core не имеет метода `SqlQuery` как такового. Можно использовать `FromSqlRaw` с моделями или Dapper.)*

#### 5.5 Получение времени сеанса по номеру билета

```csharp
public DateTime? GetSessionTimeByTicketId(int ticketId)
{
    string sql = "SELECT s.DateTime FROM Tickets t JOIN Sessions s ON t.SessionId = s.Id WHERE t.Id = @ticketId";
    return _context.Sessions.FromSqlRaw(sql, new SqlParameter("@ticketId", ticketId)).Select(s => s.DateTime).FirstOrDefault();
}
```

#### 5.6 Стандартные SQL-функции

##### 5.6.1 Названия фильмов начиная с диапазона букв

```csharp
public List<Movie> GetMoviesStartingWithRange(char startChar, char endChar)
{
    string pattern = $"{startChar}%";
    return _context.Movies.Where(m => EF.Functions.Like(m.Title, $"{startChar}%") && m.Title[0] <= endChar).ToList();
    // Или более сложным способом
}
```

##### 5.6.2 Минимальная, максимальная и средняя цена сеанса по фильму

```csharp
public (decimal min, decimal max, decimal avg) GetSessionPriceStats(int movieId)
{
    var prices = _context.Sessions.Where(s => s.MovieId == movieId);
    return (
        min: prices.Min(s => s.Price),
        max: prices.Max(s => s.Price),
        avg: prices.Average(s => s.Price)
    );
}
```

#### 5.7 Вызов хранимой процедуры получения билетов по телефону

```csharp
public List<Ticket> GetTicketsByCustomerPhone(string phone)
{
    var param = new SqlParameter("@Phone", phone);
    return _context.Tickets.FromSqlRaw("EXEC GetTicketsByPhone @Phone", param).ToList();
}
```

*(Предполагается, что в базе есть такая хранимая процедура.)*

#### 5.8 Вызов хранимой процедуры добавления посетителя с выходным параметром

```csharp
public int AddVisitor(string phone)
{
    var outputIdParam = new SqlParameter("@VisitorId", SqlDbType.Int) { Direction = ParameterDirection.Output };
    _context.Database.ExecuteSqlRaw("EXEC AddVisitor @Phone, @VisitorId OUT", new SqlParameter("@Phone", phone), outputIdParam);
    return (int)outputIdParam.Value;
}
```

#### 5.9 Вызов табличной функции по идентификатору фильма

```csharp
public List<Session> GetSessionsByMovieId(int movieId)
{
    string sql = "SELECT * FROM dbo.GetSessionsByMovieId(@MovieId)";
    return _context.Sessions.FromSqlRaw(sql, new SqlParameter("@MovieId", movieId)).ToList();
}
```

---

### Итог

Это примерная структура и реализации методов. В реальной ситуации потребуется подготовить базу данных, создать таблицы, хранимые процедуры и функции, а также настроить подключение. Также следует учитывать обработку ошибок и безопасность запросов.

Если нужно, я могу помочь подготовить конкретный код для создания базы данных или дополнительных процедур.
