Архитектура решения для лабораторной работы №14

Структура решения (Solution Structure)

```
CinemaApiSolution/
├── CinemaApi/                          # Веб-API проект (ASP.NET Core)
│   ├── Controllers/
│   │   ├── FilmsController.cs
│   │   ├── GenresController.cs
│   │   ├── VisitorsController.cs
│   │   └── TicketsController.cs
│   ├── Properties/
│   │   └── launchSettings.json
│   ├── Program.cs
│   ├── appsettings.json
│   └── CinemaApi.csproj
├── DatabaseLibrary/                    # Библиотека данных
│   ├── Models/
│   │   ├── Film.cs
│   │   ├── Genre.cs
│   │   ├── Visitor.cs
│   │   └── Ticket.cs
│   ├── Data/
│   │   └── CinemaContext.cs
│   └── DatabaseLibrary.csproj
├── ApiServicesLibrary/                 # Библиотека сервисов API
│   ├── Services/
│   │   ├── IFilmService.cs
│   │   ├── FilmService.cs
│   │   ├── IGenreService.cs
│   │   ├── GenreService.cs
│   │   ├── IVisitorService.cs
│   │   ├── VisitorService.cs
│   │   ├── ITicketService.cs
│   │   └── TicketService.cs
│   ├── ResponseHandlers/
│   │   └── ApiResponseHandler.cs
│   └── ApiServicesLibrary.csproj
├── ConsoleClient/                      # Консольное приложение
│   ├── Program.cs
│   ├── DemoRunner.cs
│   └── ConsoleClient.csproj
└── CinemaApiSolution.sln
```

Подробная архитектура компонентов

1. CinemaApi (Веб-API проект)

Program.cs

```csharp
using DatabaseLibrary.Data;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Add Scalar for API testing
builder.Services.AddScalar();

// Add DbContext
builder.Services.AddDbContext<CinemaContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    app.MapScalarApiReference();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

// Ensure database is created
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<CinemaContext>();
    context.Database.EnsureCreated();
}

app.Run();
```

appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=cinema.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

2. DatabaseLibrary

Models/Film.cs

```csharp
namespace DatabaseLibrary.Models
{
    public class Film
    {
        public int Id { get; set; }
        public string Title { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
        public int Duration { get; set; }
        public int GenreId { get; set; }
        public Genre? Genre { get; set; }
        public List<Ticket> Tickets { get; set; } = new();
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    }
}
```

Data/CinemaContext.cs

```csharp
using DatabaseLibrary.Models;
using Microsoft.EntityFrameworkCore;

namespace DatabaseLibrary.Data
{
    public class CinemaContext : DbContext
    {
        public CinemaContext(DbContextOptions<CinemaContext> options) : base(options) { }

        public DbSet<Film> Films { get; set; }
        public DbSet<Genre> Genres { get; set; }
        public DbSet<Visitor> Visitors { get; set; }
        public DbSet<Ticket> Tickets { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // Configure relationships
            modelBuilder.Entity<Film>()
                .HasOne(f => f.Genre)
                .WithMany(g => g.Films)
                .HasForeignKey(f => f.GenreId);

            modelBuilder.Entity<Ticket>()
                .HasOne(t => t.Film)
                .WithMany(f => f.Tickets)
                .HasForeignKey(t => t.FilmId);

            modelBuilder.Entity<Ticket>()
                .HasOne(t => t.Visitor)
                .WithMany(v => v.Tickets)
                .HasForeignKey(t => t.VisitorId);

            // Seed initial data
            modelBuilder.Entity<Genre>().HasData(
                new Genre { Id = 1, Name = "Action" },
                new Genre { Id = 2, Name = "Comedy" },
                new Genre { Id = 3, Name = "Drama" },
                new Genre { Id = 4, Name = "Horror" }
            );
        }
    }
}
```

3. ApiServicesLibrary

Services/IFilmService.cs

```csharp
using DatabaseLibrary.Models;

namespace ApiServicesLibrary.Services
{
    public interface IFilmService
    {
        Task<List<Film>> GetAllFilmsAsync();
        Task<Film?> GetFilmByIdAsync(int id);
        Task<Film> CreateFilmAsync(Film film);
        Task<bool> UpdateFilmAsync(int id, Film film);
        Task<bool> DeleteFilmAsync(int id);
    }
}
```

Services/FilmService.cs (с автоматической сериализацией)

```csharp
using DatabaseLibrary.Models;
using System.Net.Http.Json;

namespace ApiServicesLibrary.Services
{
    public class FilmService : IFilmService
    {
        private readonly HttpClient _httpClient;
        private readonly string _baseUrl = "api/films";

        public FilmService(HttpClient httpClient)
        {
            _httpClient = httpClient;
        }

        public async Task<List<Film>> GetAllFilmsAsync()
        {
            var response = await _httpClient.GetAsync(_baseUrl);
            response.EnsureSuccessStatusCode();
            
            return await response.Content.ReadFromJsonAsync<List<Film>>() 
                ?? new List<Film>();
        }

        public async Task<Film?> GetFilmByIdAsync(int id)
        {
            var response = await _httpClient.GetAsync($"{_baseUrl}/{id}");
            
            if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
                return null;
                
            response.EnsureSuccessStatusCode();
            
            return await response.Content.ReadFromJsonAsync<Film>();
        }

        public async Task<Film> CreateFilmAsync(Film film)
        {
            var response = await _httpClient.PostAsJsonAsync(_baseUrl, film);
            
            if (response.StatusCode == System.Net.HttpStatusCode.BadRequest)
                throw new ArgumentException("Invalid film data");
                
            response.EnsureSuccessStatusCode();
            
            return await response.Content.ReadFromJsonAsync<Film>() 
                ?? throw new Exception("Failed to create film");
        }

        public async Task<bool> UpdateFilmAsync(int id, Film film)
        {
            var response = await _httpClient.PutAsJsonAsync($"{_baseUrl}/{id}", film);
            
            if (response.StatusCode == System.Net.HttpStatusCode.BadRequest)
                throw new ArgumentException("Invalid film data");
                
            if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
                return false;
                
            response.EnsureSuccessStatusCode();
            return true;
        }

        public async Task<bool> DeleteFilmAsync(int id)
        {
            var response = await _httpClient.DeleteAsync($"{_baseUrl}/{id}");
            
            if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
                return false;
                
            response.EnsureSuccessStatusCode();
            return true;
        }
    }
}
```

Services/GenreService.cs (с явной сериализацией)

```csharp
using DatabaseLibrary.Models;
using System.Text;
using System.Text.Json;

namespace ApiServicesLibrary.Services
{
    public class GenreService
    {
        private readonly HttpClient _httpClient;
        private readonly string _baseUrl = "api/genres";
        private readonly JsonSerializerOptions _jsonOptions;

        public GenreService(HttpClient httpClient)
        {
            _httpClient = httpClient;
            _jsonOptions = new JsonSerializerOptions
            {
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
                PropertyNameCaseInsensitive = true
            };
        }

        public async Task<List<Genre>> GetAllGenresAsync()
        {
            var response = await _httpClient.GetAsync(_baseUrl);
            response.EnsureSuccessStatusCode();
            
            var content = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<List<Genre>>(content, _jsonOptions) 
                ?? new List<Genre>();
        }

        public async Task<Genre> CreateGenreAsync(Genre genre)
        {
            var json = JsonSerializer.Serialize(genre, _jsonOptions);
            var content = new StringContent(json, Encoding.UTF8, "application/json");
            
            var response = await _httpClient.PostAsync(_baseUrl, content);
            response.EnsureSuccessStatusCode();
            
            var responseContent = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<Genre>(responseContent, _jsonOptions) 
                ?? throw new Exception("Failed to create genre");
        }
    }
}
```

4. ConsoleClient

Program.cs

```csharp
using ApiServicesLibrary.Services;
using DatabaseLibrary.Models;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

// Create host builder
var builder = Host.CreateApplicationBuilder(args);

// Configure HttpClient
builder.Services.AddHttpClient<IFilmService, FilmService>(client =>
{
    client.BaseAddress = new Uri("https://localhost:7000/");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

builder.Services.AddHttpClient<GenreService>(client =>
{
    client.BaseAddress = new Uri("https://localhost:7000/");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

builder.Services.AddTransient<DemoRunner>();

var host = builder.Build();

// Run demo
var demoRunner = host.Services.GetRequiredService<DemoRunner>();
await demoRunner.RunAsync();

Console.WriteLine("Press any key to exit...");
Console.ReadKey();
```

DemoRunner.cs

```csharp
using ApiServicesLibrary.Services;
using DatabaseLibrary.Models;

namespace ConsoleClient
{
    public class DemoRunner
    {
        private readonly IFilmService _filmService;
        private readonly GenreService _genreService;

        public DemoRunner(IFilmService filmService, GenreService genreService)
        {
            _filmService = filmService;
            _genreService = genreService;
        }

        public async Task RunAsync()
        {
            try
            {
                Console.WriteLine("=== Testing Film Service (Automatic Serialization) ===");
                await TestFilmService();
                
                Console.WriteLine("\n=== Testing Genre Service (Explicit Serialization) ===");
                await TestGenreService();
                
                Console.WriteLine("\n=== Testing Error Handling ===");
                await TestErrorHandling();
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
            }
        }

        private async Task TestFilmService()
        {
            // Get all films
            var films = await _filmService.GetAllFilmsAsync();
            Console.WriteLine($"Found {films.Count} films");

            // Create new film
            var newFilm = new Film
            {
                Title = "The Matrix",
                Description = "Sci-fi action film",
                Duration = 136,
                GenreId = 1
            };

            var createdFilm = await _filmService.CreateFilmAsync(newFilm);
            Console.WriteLine($"Created film: {createdFilm.Title}");

            // Get film by ID
            var film = await _filmService.GetFilmByIdAsync(createdFilm.Id);
            if (film != null)
            {
                Console.WriteLine($"Retrieved film: {film.Title}");
            }

            // Update film
            film!.Description = "Updated description";
            var updated = await _filmService.UpdateFilmAsync(film.Id, film);
            Console.WriteLine($"Film updated: {updated}");
        }

        private async Task TestGenreService()
        {
            var genres = await _genreService.GetAllGenresAsync();
            Console.WriteLine($"Found {genres.Count} genres");

            foreach (var genre in genres)
            {
                Console.WriteLine($"- {genre.Name}");
            }
        }

        private async Task TestErrorHandling()
        {
            // Test non-existent film
            var film = await _filmService.GetFilmByIdAsync(9999);
            if (film == null)
            {
                Console.WriteLine("Correctly returned null for non-existent film");
            }

            // Test invalid data
            try
            {
                var invalidFilm = new Film { Title = "" };
                await _filmService.CreateFilmAsync(invalidFilm);
            }
            catch (ArgumentException ex)
            {
                Console.WriteLine($"Correctly caught validation error: {ex.Message}");
            }
        }
    }
}
```

Зависимости проектов (Project References)

CinemaApi.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="9.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="9.0.0" />
    <PackageReference Include="Scalar" Version="2.0.13" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\DatabaseLibrary\DatabaseLibrary.csproj" />
  </ItemGroup>
</Project>
```

ApiServicesLibrary.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\DatabaseLibrary\DatabaseLibrary.csproj" />
  </ItemGroup>
</Project>
```

Настройка запуска

.vscode/launch.json (для VS Code)

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Cinema API",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "build",
      "program": "${workspaceFolder}/CinemaApi/bin/Debug/net9.0/CinemaApi.dll",
      "args": [],
      "cwd": "${workspaceFolder}/CinemaApi",
      "serverReadyAction": {
        "action": "openExternally",
        "pattern": "https://localhost:([0-9]+)",
        "uriFormat": "https://localhost:%s/scalar"
      },
      "env": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    {
      "name": "Console Client",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "build",
      "program": "${workspaceFolder}/ConsoleClient/bin/Debug/net9.0/ConsoleClient.dll",
      "args": [],
      "cwd": "${workspaceFolder}/ConsoleClient"
    }
  ],
  "compounds": [
    {
      "name": "Run All",
      "configurations": ["Cinema API", "Console Client"],
      "stopAll": true
    }
  ]
}
```

Эта архитектура обеспечивает:

· Четкое разделение ответственности между проектами
· Возможность тестирования каждого компонента независимо
· Гибкость в выборе способа сериализации
· Обработку ошибок на разных уровнях
· Легкость расширения для добавления новых сущностей и сервисов