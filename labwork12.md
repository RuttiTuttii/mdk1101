Понял, нужно выполнить все задания лабораторной работы через сервисы. Вот полное решение:

Program.cs

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        // Database
        services.AddDbContext<FilmAppDbContext>(options =>
            options.UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=FilmAppDb;Trusted_Connection=true"));

        // Services
        services.AddScoped<IMainService, MainService>();
    })
    .Build();

using (var scope = host.Services.CreateScope())
{
    var mainService = scope.ServiceProvider.GetRequiredService<IMainService>();
    
    // Тестирование всех методов
    await TestAllMethods(mainService);
}

async Task TestAllMethods(IMainService mainService)
{
    Console.WriteLine("=== Лабораторная работа №12 ===");
    
    // 5.1
    Console.WriteLine("\n5.1 - Фильмы отсортированные по названию:");
    var films = await mainService.GetFilmsSortedByColumnAsync("Title");
    foreach (var film in films.Take(3))
        Console.WriteLine($"{film.Title} ({film.Year})");

    // 5.2
    Console.WriteLine("\n5.2 - Фильмы по названию и году:");
    var filmsByTitle = await mainService.GetFilmsByTitleAndYearAsync("Inception", 2010);
    foreach (var film in filmsByTitle)
        Console.WriteLine($"{film.Title} ({film.Year})");

    // 5.3
    Console.WriteLine("\n5.3 - Обновление цен:");
    var updatedRows = await mainService.IncreaseSessionPriceAsync(1, 50);
    Console.WriteLine($"Обновлено строк: {updatedRows}");

    // 5.4
    Console.WriteLine("\n5.4 - Жанры фильма:");
    var genres = await mainService.GetFilmGenresAsync(1);
    Console.WriteLine($"Жанры: {string.Join(", ", genres)}");

    // 5.5
    Console.WriteLine("\n5.5 - Дата сеанса по билету:");
    var sessionDate = await mainService.GetSessionDateTimeByTicketAsync(1);
    Console.WriteLine($"Дата сеанса: {sessionDate}");

    // 5.6.1
    Console.WriteLine("\n5.6.1 - Фильмы по диапазону названий:");
    var filmsByRange = await mainService.GetFilmsByTitleRangeAsync('A', 'D');
    foreach (var film in filmsByRange.Take(3))
        Console.WriteLine($"{film.Title}");

    // 5.6.2
    Console.WriteLine("\n5.6.2 - Статистика цен:");
    var priceStats = await mainService.GetSessionPriceStatsAsync(1);
    Console.WriteLine($"Min: {priceStats.MinPrice}, Max: {priceStats.MaxPrice}, Avg: {priceStats.AvgPrice}");

    // 5.7
    Console.WriteLine("\n5.7 - Билеты по телефону:");
    var tickets = await mainService.GetVisitorTicketsByPhoneAsync("+1234567890");
    foreach (var ticket in tickets.Take(2))
        Console.WriteLine($"Билет {ticket.Id}");

    // 5.8
    Console.WriteLine("\n5.8 - Добавление посетителя:");
    var visitorId = await mainService.AddVisitorAndGetIdAsync("+9876543210");
    Console.WriteLine($"ID нового посетителя: {visitorId}");

    // 5.9
    Console.WriteLine("\n5.9 - Сеансы по фильму:");
    var sessions = await mainService.GetSessionsByFilmIdAsync(1);
    foreach (var session in sessions.Take(2))
        Console.WriteLine($"Сеанс {session.Id} - {session.DateTime}");
}

await host.RunAsync();
```

Services/MainService.cs

```csharp
using Microsoft.EntityFrameworkCore;

public interface IMainService
{
    // 5.1 - Выполнение SQL-команды на выборку данных (FromSqlRaw)
    Task<List<Film>> GetFilmsSortedByColumnAsync(string columnName);
    
    // 5.2 - Выполнение SQL-команды на выборку данных с входными параметрами (FromSql)
    Task<List<Film>> GetFilmsByTitleAndYearAsync(string title, int minYear);
    
    // 5.3 - Выполнение SQL-команды на модификацию данных с входными параметрами (ExecuteSql)
    Task<int> IncreaseSessionPriceAsync(int hallId, decimal increaseAmount);
    
    // 5.4 - Получение списка значений на основе параметров (SqlQuery)
    Task<List<string>> GetFilmGenresAsync(int filmId);
    
    // 5.5 - Получение одного значения на основе параметров (SqlQuery)
    Task<DateTime?> GetSessionDateTimeByTicketAsync(int ticketId);
    
    // 5.6.1 - Выполнение стандартных SQL-функций (EF.Functions.Like)
    Task<List<Film>> GetFilmsByTitleRangeAsync(char startChar, char endChar);
    
    // 5.6.2 - Стандартные агрегатные функции LINQ
    Task<PriceStats> GetSessionPriceStatsAsync(int filmId);
    
    // 5.7 - Вызов хранимой процедуры со входными параметрами
    Task<List<Ticket>> GetVisitorTicketsByPhoneAsync(string phone);
    
    // 5.8 - Вызов хранимой процедуры с выходными параметрами
    Task<int> AddVisitorAndGetIdAsync(string phone);
    
    // 5.9 - Вызов табличной функции пользователя
    Task<List<Session>> GetSessionsByFilmIdAsync(int filmId);
}

public class PriceStats
{
    public decimal MinPrice { get; set; }
    public decimal MaxPrice { get; set; }
    public decimal AvgPrice { get; set; }
}

public class MainService : IMainService
{
    private readonly FilmAppDbContext _context;

    public MainService(FilmAppDbContext context)
    {
        _context = context;
    }

    // 5.1 - Сортировка фильмов по указанному столбцу
    public async Task<List<Film>> GetFilmsSortedByColumnAsync(string columnName)
    {
        // Защита от SQL-инъекций - проверяем допустимые имена столбцов
        var allowedColumns = new[] { "Title", "Year", "Director", "Rating" };
        if (!allowedColumns.Contains(columnName))
            throw new ArgumentException("Недопустимое имя столбца");

        return await _context.Films
            .FromSqlRaw($"SELECT * FROM Films ORDER BY {columnName}")
            .ToListAsync();
    }

    // 5.2 - Фильмы по названию и минимальному году
    public async Task<List<Film>> GetFilmsByTitleAndYearAsync(string title, int minYear)
    {
        return await _context.Films
            .FromSqlInterpolated($"SELECT * FROM Films WHERE Title = {title} AND Year >= {minYear}")
            .ToListAsync();
    }

    // 5.3 - Увеличение цены сеансов в указанном зале
    public async Task<int> IncreaseSessionPriceAsync(int hallId, decimal increaseAmount)
    {
        return await _context.Database
            .ExecuteSqlRawAsync("UPDATE Sessions SET Price = Price + {0} WHERE HallId = {1}", 
                increaseAmount, hallId);
    }

    // 5.4 - Список жанров фильма по ID
    public async Task<List<string>> GetFilmGenresAsync(int filmId)
    {
        return await _context.Database
            .SqlQueryRaw<string>("SELECT Genre FROM FilmGenres WHERE FilmId = {0}", filmId)
            .ToListAsync();
    }

    // 5.5 - Дата и время сеанса по номеру билета
    public async Task<DateTime?> GetSessionDateTimeByTicketAsync(int ticketId)
    {
        return await _context.Database
            .SqlQueryRaw<DateTime>("SELECT s.DateTime FROM Sessions s JOIN Tickets t ON s.Id = t.SessionId WHERE t.Id = {0}", ticketId)
            .FirstOrDefaultAsync();
    }

    // 5.6.1 - Поиск фильмов по диапазону первых букв названия
    public async Task<List<Film>> GetFilmsByTitleRangeAsync(char startChar, char endChar)
    {
        return await _context.Films
            .Where(f => EF.Functions.Like(f.Title, $"[{startChar}-{endChar}]%"))
            .ToListAsync();
    }

    // 5.6.2 - Статистика цен на сеансы фильма
    public async Task<PriceStats> GetSessionPriceStatsAsync(int filmId)
    {
        var sessions = _context.Sessions.Where(s => s.FilmId == filmId);
        
        return new PriceStats
        {
            MinPrice = await sessions.MinAsync(s => s.Price),
            MaxPrice = await sessions.MaxAsync(s => s.Price),
            AvgPrice = await sessions.AverageAsync(s => s.Price)
        };
    }

    // 5.7 - Билеты посетителя по номеру телефона (хранимая процедура)
    public async Task<List<Ticket>> GetVisitorTicketsByPhoneAsync(string phone)
    {
        return await _context.Tickets
            .FromSqlRaw("EXEC GetVisitorTicketsByPhone @Phone = {0}", phone)
            .ToListAsync();
    }

    // 5.8 - Добавление посетителя и возврат ID (хранимая процедура с выходным параметром)
    public async Task<int> AddVisitorAndGetIdAsync(string phone)
    {
        var outputParam = new Microsoft.Data.SqlClient.SqlParameter
        {
            ParameterName = "@NewVisitorId",
            SqlDbType = System.Data.SqlDbType.Int,
            Direction = System.Data.ParameterDirection.Output
        };

        await _context.Database
            .ExecuteSqlRawAsync("EXEC AddVisitor @Phone = {0}, @NewVisitorId = {1} OUTPUT", 
                phone, outputParam);

        return (int)outputParam.Value;
    }

    // 5.9 - Сеансы фильма по ID (табличная функция)
    public async Task<List<Session>> GetSessionsByFilmIdAsync(int filmId)
    {
        return await _context.Sessions
            .FromSqlRaw("SELECT * FROM dbo.GetSessionsByFilmId({0})", filmId)
            .ToListAsync();
    }
}
```

SQL для создания процедур и функций:

```sql
-- Хранимая процедура для задания 5.7
CREATE PROCEDURE GetVisitorTicketsByPhone
    @Phone NVARCHAR(20)
AS
BEGIN
    SELECT t.* 
    FROM Tickets t
    INNER JOIN Visitors v ON t.VisitorId = v.Id
    WHERE v.Phone = @Phone
END

-- Хранимая процедура для задания 5.8
CREATE PROCEDURE AddVisitor
    @Phone NVARCHAR(20),
    @NewVisitorId INT OUTPUT
AS
BEGIN
    INSERT INTO Visitors (Phone, RegistrationDate)
    VALUES (@Phone, GETDATE())
    
    SET @NewVisitorId = SCOPE_IDENTITY()
END

-- Табличная функция для задания 5.9
CREATE FUNCTION GetSessionsByFilmId(@FilmId INT)
RETURNS TABLE
AS
RETURN
    SELECT * FROM Sessions WHERE FilmId = @FilmId
```

Это полное решение лабораторной работы, покрывающее все 9 заданий через сервисный слой с использованием всех требуемых технологий EF Core.