# ASP.NET Core Web API

> Данный обучающий материал предназначен для .NET 9/10. Важно: Swagger больше не часть стандартной
> поставки ASP.NET и не работает по умолчанию. Заменён на Microsoft.AspNetCore.OpenApi.

## Базовые термины ASP.NET

ASP.NET Core - кроссплатформенный фреймворк для создания веб-приложений и API. Создаётся Microsoft для платформы .Net и ориентирован на упрощение жизни разработчков, реализуя MVC паттерн на уровне архитектуры.

Есть 2 подхода к API:

- Controller-based - классический вариант задания API с контроллерами и атрибутами
- Minimal API - лёгкий и минималистичный, появился в .Net 6

## Структура проекта

Весь проект ASP.NET API разделён на файлы и каталоги исходя из назначения каждого конкретного файла.

```bash
MyApiProject/
├── Program.cs                           # Точка входа: создание хоста, настройка сервисов и middleware
├── appsettings.json                     # Основная конфигурация (строки подключения, настройки)
├── appsettings.Development.json         # Переопределение конфигурации для режима разработки
├── MyApiProject.csproj                  # Файл проекта (TargetFramework, пакеты, версии)
├── Properties/
│   └── launchSettings.json              # Профили запуска (IIS, Kestrel, переменные окружения)
├── Controllers/                         # (API с контроллерами) Классы контроллеров, наследующие ControllerBase
├── Models/                              # Классы данных: Entities, DTO, ViewModels
├── Services/                            # (или Infrastructure) Бизнес-логика, внешние сервисы
├── Migrations/                          # (EF Core) Автоматически генерируемые файлы миграций
├── Middleware/                          # (опционально) Собственные middleware-компоненты
├── Filters/                             # (опционально) Фильтры действий, исключений, авторизации
└── wwwroot/                             # Статические файлы (CSS, JS, изображения)
```

## Middleware - конвейер обработки запросов

Middleware - компоненты приложения на ASP.NET образующие конвейер обработки HTTP запроса и формирования ответа. Каждый Middleware либо обрабатывает запрос, либо передаёт его следующему компоненту.

```c#
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Middleware 1: логирование запросов
app.Use(async (context, next) =>
{
    Console.WriteLine($"Запрос: {context.Request.Method} {context.Request.Path}");
    await next(); // передаём дальше
    Console.WriteLine($"Ответ: {context.Response.StatusCode}");
});

// Middleware 2: обработка ошибок
app.UseExceptionHandler("/error");

// Middleware 3: маршрутизация
app.UseRouting();

// Middleware 4: авторизация
app.UseAuthorization();

// Middleware 5: конечные точки (endpoints)
app.MapControllers();

app.Run();
```

Middleware выполняются строго в том порядке, в котором они зарегистрированы. 

 ## Dependency Injection (DI) - встроенный контейнер

 ASP.NET Core имеет встроенный DI-контейнер. Все сервисы регистрируются в контейнере и внедряются через конструкторы.

 Сервисы имеют время жизни, зависящее от их назначения

 - Transient - создаются при каждом запросе. Лёгкие сервисы, не имеют состояния.
 - Scoped - сервисы создающиеся один раз на каждый запрос. Должны существовать на протяжении обработки запроса
 - Singleton - сервисы, создающиеся в единственном экземпляре на время жизни самого приложения. Это тяжелые сервисы, где много данных и/или логики, а так же кеши приложения.

```c#
var builder = WebApplication.CreateBuilder(args);

// Регистрация сервисов в DI-контейнере
builder.Services.AddControllers(); // регистрация контроллеров

// Регистрация собственных сервисов
builder.Services.AddScoped<IUserService, UserService>();      // на запрос
builder.Services.AddTransient<IEmailService, EmailService>(); // каждый раз новый
builder.Services.AddSingleton<ICacheService, CacheService>(); // один на всё приложение

// Регистрация DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Регистрация OpenAPI (встроенная поддержка в .NET 9+)
builder.Services.AddOpenApi(); // замена Swashbuckle[reference:5][reference:6]

var app = builder.Build();
```

Важно: определение времени жизни происходит самим разработчиком для своих сервисов, встроенные и библиотечные сервисы сами определяют (например AddOpenAPI) когда и как они живут и реализация жизни скрыта от разработчика.

Далее происходит внедрение через конструктор

```c#
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly ILogger<UsersController> _logger;

    // Зависимости внедряются через конструктор
    public UsersController(IUserService userService, ILogger<UsersController> logger)
    {
        _userService = userService;
        _logger = logger;
    }

    [HttpGet]
    public async Task<IActionResult> GetUsers()
    {
        _logger.LogInformation("Получение списка пользователей");
        var users = await _userService.GetAllAsync();
        return Ok(users);
    }
}
```

Т.е. контейнер инъекции зависимостей смотрит на контроллер и по сигнатуре методом рефлексии понимает, что он кушает `<IUserService, ILogger<UsersController>>` и передаёт ему экземпляры этих сервисов (которые создаются исходя из времени жизни сервиса), когда пользователь приложения обращается к контроллеру. 

## Настройка приложения

Настройки производятся в 2-х основных местах: 
- `appsettings.json` - Здесь осуществляется определение параметров приложения через json-конфигурацию
- `Program.cs` - точка входа в приложение, здесь настраиваются сервисы приложения, порядок вызова Middleware

`appsettings.json`

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyDb;User Id=sa;Password=MyPass;"
  },
  "AllowedHosts": "*"
}
```

`Program.cs`

```c#
using Microsoft.AspNetCore.OpenApi; // для OpenAPI

var builder = WebApplication.CreateBuilder(args);

// 1. Добавляем сервисы в DI-контейнер
builder.Services.AddControllers();
builder.Services.AddOpenApi(); // встроенная поддержка OpenAPI (.NET 9+)[reference:7]

// 2. Настройка логирования
builder.Logging.ClearProviders();
builder.Logging.AddConsole();

// 3. Сборка приложения
var app = builder.Build();

// 4. Middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi(); // эндпоинт для OpenAPI документа (.NET 9+)[reference:8]
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

Для доступа к конфигурации:

```c#
public class MyService
{
    private readonly IConfiguration _configuration;

    public MyService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public string GetConnectionString()
    {
        // Для строковых значений
        string customValue = _configuration["MyCustomKey"];

        // Для типизированных значений (с указанием типа)
        int timeout = _configuration.GetValue<int>("Timeout", 30); // 30 — значение по умолчанию

        // Для секций
        var section = _configuration.GetSection("MySection");
        string subKey = section["SubKey"];
        return _configuration.GetConnectionString("DefaultConnection");
    }
}
```

## OpenAPI

Ключевое изменение в .NET 9+ - внедрение OpenAPI вместо Swagger для отладки API приложений.

Пакет: `Microsoft.AspNetCore.OpenApi`
Добавление сервиса: `builder.Services.AddOpenApi()`
Добавление Middleware: `app.MapOpenApi()`
UI сервиса подключается теперь отдельно, рекомендованный Scalar UI. Схема API доступна по `endpoint` `/openapi/{documentName}.json. При регистрации сервисов ты можешь задать имя документа:

```c#
// Регистрируем документ с именем "v1" (это имя по умолчанию, если не указать другое)
builder.Services.AddOpenApi();

// Регистрируем документ с именем "internal"
builder.Services.AddOpenApi("internal");
```

Если ничего не передавать, то будет имя `v1.json`


Ты можешь изменить шаблон маршрута, чтобы, например, убрать {documentName} из пути или использовать другой формат:

```c#
// Изменяем маршрут для OpenAPI JSON
app.MapOpenApi("/openapi/{documentName}/openapi.json");[reference:5]

// Или для YAML формата
app.MapOpenApi("/openapi/{documentName}.yaml");[reference:6]
```

## Scalar UI - интерфейс отладки API

Установка: `dotnet add package Scalar.AspNetCore`

Применение в `Program.cs`:

```c#
var builder = WebApplication.CreateBuilder(args);

// 1. Добавляем встроенную генерацию OpenAPI
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    // 2. Маппим эндпоинт для OpenAPI JSON /openapi/v1.json
    app.MapOpenApi();

    // 3. Маппим Scalar UI
    app.MapScalarApiReference(); // Обычно доступно по /scalar/v1
}

app.Run();
```

Для Scalar UI и OpenAPI можно так же зарегистрировать несколько документов:

```c#
var builder = WebApplication.CreateBuilder(args);

// Регистрируем два документа с разными именами
builder.Services.AddOpenApi("v1");
builder.Services.AddOpenApi("internal");

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    // Эндпоинты для JSON-документов:
    // /openapi/v1.json
    // /openapi/internal.json
    app.MapOpenApi();

    // Scalar UI для каждого документа:
    // /scalar/v1
    // /scalar/internal
    app.MapScalarApiReference();
}

app.Run();
```

## Minimal API - малые и новые проекты, особенно с микросервисной архитектурой, рекомендуется

Минимальные API были представлены в ASP.NET Core 6 как способ создания HTTP-API с минимальным количеством кода и конфигурации. Microsoft рекомендует использовать Minimal API для новых проектов.

Минимально возможный пример

```c#
var app = WebApplication.Create(args);

app.MapGet("/", () => "Hello World!"); // [11†L26]

app.Run();
```

С параметрыми маршрутизации:

```c#
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/users/{userId}/books/{bookId}", (int userId, int bookId) => 
    $"The user id is {userId} and book id is {bookId}"); // [12†L31-L32]

app.Run();
```

Внедрение зависимостей:

```c#
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IUserService, UserService>();
var app = builder.Build();

app.MapGet("/users", async (IUserService userService) => 
{
    var users = await userService.GetAllAsync();
    return Results.Ok(users);
});

app.Run();
```

Где размещать код? Вся логика Minimal API обычно пишется непосредственно в Program.cs. Однако для поддержания порядка, можно выносить обработчики (handler) в отдельные статические методы или даже в отдельные классы, если проект разрастается.

## Управление жизненным циклом. Окружения Development vs Production

`Development` - среда исполнения для локальной разработки. Включает отладочные функции, активируется автоматически через `launchSettings.json` при локальном запуске средствами `dotnet`. 

Приложение смотрит во время запуска на переменную среды `ASPNETCORE_ENVIRONMENT`, если она не задана, то `DOTNET_ENVIRONMENT`, если и она не задана, то окружение считается `Production`.

В Windows среде имена переменных не чувствительны к регистру, для Linux среды чувствительны.

Способы задания окружения:

- `launchSettings.json` - в visual studio и VS Code файл `Properties/launchSettings.json` задаёт `ASPNETCORE_ENVIRONMENT = Development` по умолчанию и запускается из пакета расширения C# Dev Kit по кнопке вверхней части интерфейса. Данный способ - рекомендуемый. 
- Переменная окружения - для сессии в терминале или в постоянные переменные среды можно внести значения переменных ASPNETCORE_ENVIRONMENT и DOTNET_ENVIRONMENT, может быть полезно для Staging сервера.
- Командная строка: во время запуска можно указать окружение `dotnet run --environment Production/Staging/Development`
- Можно переопределить в коде, но не надо. 

Определить в приложении, где работа осуществляется можно внедрив зависимость от `IWebHostEnvironment`:

```c#
public class MyService
{
    private readonly IWebHostEnvironment _env;

    public MyService(IWebHostEnvironment env)
    {
        _env = env;
    }

    public void DoWork()
    {
        if (_env.IsDevelopment())
        {
            // Код, который выполняется только в Development
        }
        else if (_env.IsProduction())
        {
            // Код для Production
        }
    }
}
```

`Production` - среда для релизной версии приложения, а как задать production уже понятно из описаний выше.

## Настройки веб-сервиса

По умолчанию `ASP.NET` в новых приложениях генерирует и сохраняет случайные порты для HTTP и HTTPS в файле `Properties/launchSettings.json` и доступны они для локальной разработки.

HTTP порт случайный в диапазоне 5000-5300
HTTPS порт случайный в диапазоне 7000-7300

Для настройки для `Production` есть несколько вариантов:

1. Задать переменную окружения 

    ```bash
    # Для Windows (CMD)
    set ASPNETCORE_URLS=https://localhost:5001;http://localhost:5000
    dotnet run

    # Для Linux/macOS
    export ASPNETCORE_URLS="https://localhost:5001;http://localhost:5000"
    dotnet run
    ```

2. Аргумент командной строки --urls

    ```bash
    dotnet run --urls "https://localhost:5001;http://localhost:5000"
    ```

3. `appsettings.json` можно задать через конфигурационный файл конечные точки (лучший способ для прода):

    ```json
    {
    "Kestrel": {
        "Endpoints": {
        "Http": {
            "Url": "http://localhost:5000"
        },
        "Https": {
            "Url": "https://localhost:5001"
        }
        }
    }
    }
    ```

4. для  локальной разработки можно задать профили с `applicationUrl`:

    ```json
    {
    "profiles": {
        "MyApi": {
        "applicationUrl": "https://localhost:5001;http://localhost:5000"
        }
    }
    }
    ```

5. Можно задать через код:

    ```c#
    var builder = WebApplication.CreateBuilder(args);
    builder.WebHost.UseUrls("http://localhost:5000", "https://localhost:5001");
    // или
    builder.WebHost.ConfigureKestrel(options => { 
        options.ListenLocalhost(5000); // Слушаем только localhost на 5000 порту[reference:1]
        options.ListenAnyIP(7000, configure => configure.UseHttps()); // Слушаем все IP на 7000 порту с HTTPS[reference:2]
     });
    ```

    Вариант с ConfigureKestrel полезен для создания множества endpoint на разных протоколах.

## Маршрутизация

Маршрутизация - механизм, позволяющий связывать Url эндпоинта с конкретным методом в контроллере.

Контроллеры это обработчики твоих Endpoint'ов

Атрибутная маршрутизация - атрибуты применяются прямо на контроллере или его методах

```c#
[ApiController]
[Route("api/[controller]")]      // [controller] подставит имя контроллера без суффикса
public class UsersController : ControllerBase
{
    // GET: api/users
    [HttpGet]
    public IActionResult GetAll() { ... }

    // GET: api/users/5
    [HttpGet("{id}")]
    public IActionResult GetById(int id) { ... }

    // GET: api/users/5/orders
    [HttpGet("{id}/orders")]
    public IActionResult GetOrders(int id) { ... }

    // POST: api/users
    [HttpPost]
    public IActionResult Create([FromBody] UserDto dto) { ... }

    // PUT: api/users/5
    [HttpPut("{id}")]
    public IActionResult Update(int id, [FromBody] UserDto dto) { ... }

    // DELETE: api/users/5
    [HttpDelete("{id}")]
    public IActionResult Delete(int id) { ... }
}
```

Что даёт `[ApiController]`:
- Автоматическая валидация моделей — если модель не прошла валидацию, возвращает 400 Bad Request без твоего кода.
- Вывод источника привязки — для сложных объектов подразумевается `[FromBody]`, для простых — `[FromQuery]`.
- Обязательная атрибутная маршрутизация.
- Автоматические ответы с `ProblemDetails` при ошибках.

Важно: ControllerBase — для API (без поддержки представлений). Controller — для MVC с представлениями.

## Model Binding

`Model Binding` автоматически извлекает данные из HTTP-запроса и преобразует их в параметры метода или свойства модели

- `[FromRoute]` - из маршрута: `[HttpGet("{id}")] Get(int id)`
- `[FromQuery]` - Query-строка	`GET /api/users?page=2`
- `[FromBody]` - Тело запроса (JSON) `POST /api/users` с JSON в теле
- `[FromForm]`	Данные формы	POST с `application/x-www-form-urlencoded`
- `[FromHeader]`	Заголовки HTTP	`Authorization: Bearer ...`
- `[FromServices]`	DI-контейнер	Внедрение сервиса прямо в метод

```c#
[HttpGet]
public IActionResult Get(
    [FromQuery] int page = 1,           // /api/users?page=2
    [FromQuery] int pageSize = 10)      // /api/users?page=2&pageSize=20
{
    // ...
}

[HttpGet("{id}")]
public IActionResult GetById(
    [FromRoute] int id,                 // /api/users/5 → id = 5
    [FromQuery] bool includeOrders)     // /api/users/5?includeOrders=true
{
    // ...
}

[HttpPost]
public IActionResult Create(
    [FromBody] CreateUserRequest request) // JSON из тела запроса
{
    // ...
}

[HttpGet]
public IActionResult Get([FromServices] IUserService service)
{
    return Ok(service.GetAll());
}
//Привязка сложных моделей
public class CreateUserRequest
{
    public string Name { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
}
// Где-то в контроллере
[HttpPost]
public IActionResult Create([FromBody] CreateUserRequest request)
{
    // request.Name, request.Email, request.Age уже заполнены
}
```

## Валидация

DataAnnotations — набор атрибутов для декларативной валидации моделей

- `[Required]`	Поле обязательно для заполнения
- `[StringLength(max, MinimumLength = min)]`	Ограничение длины строки
- `[Range(min, max)]`	Числовой диапазон
- `[RegularExpression("pattern")]` Проверка по регулярному выражению
- `[EmailAddress]`	Проверка формата email
- `[Phone]`	Проверка формата телефона
- `[Url]`	Проверка формата URL
- `[Compare("otherProperty")]`	Сравнение с другим свойством

```c#
public class CreateUserRequest
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Range(18, 99)]
    public int Age { get; set; }

    [RegularExpression(@"^\+?[1-9]\d{1,14}$")]
    public string Phone { get; set; }
}
```

Проверка валидации в контроллере
С [ApiController] проверка происходит автоматически

```c#
[ApiController]
public class UsersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create([FromBody] CreateUserRequest request)
    {
        // Если модель невалидна, сюда даже не попадём — вернётся 400 Bad Request
        // Если попали — всё ок
        return Ok();
    }
}
```

Ручная проверка (если не используешь [ApiController]):

```c#
[HttpPost]
public IActionResult Create([FromBody] CreateUserRequest request)
{
    if (!ModelState.IsValid)                    // проверяем валидность[reference:21]
    {
        return BadRequest(ModelState);          // возвращаем ошибки
    }
    // ...
}
```

## Возвращаемые значения

Конкретный тип — для простых случаев

```csharp
[HttpGet]
public List<User> Get()                         // всегда 200 OK
{
    return _context.Users.ToList();
}
```

ActionResult — для нескольких вариантов ответа

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    var user = _context.Users.Find(id);
    if (user == null)
    {
        return NotFound();                      // 404
    }
    return Ok(user);                            // 200
}
```

ActionResult<T> — типизированный вариант (рекомендуется)

```csharp
[HttpGet("{id}")]
public ActionResult<User> Get(int id)           // явно указываем тип
{
    var user = _context.Users.Find(id);
    if (user == null)
    {
        return NotFound();                      // 404
    }
    return Ok(user);                            // 200 + User
}
```

Некоторые возвратные значение:
`Ok()` - 200, всё ок
`NoContent()` - 204, всё ок, но там, где смотрим - пусто
`BadRequest()` - 400 пользователь дурак и бака и не правильно спросил сервер
`NotFound()` - 404 ничего не найдено
`Unauthorized()` - 401 кто-то попытался получить доступ туда, куда не авторизовался
`Forbid()` - 403 тебе туда нельзя

## Комплексный пример котроллера

```c#
[ApiController] // Авто валидация, требуется атрибутика
[Route("api/[controller]")] // Путь /api/users имя подставилось автоматически
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly ILogger<UsersController> _logger;

    // Здесь через DI контроллер узнаёт о модели и получает логгер
    public UsersController(IUserService userService, ILogger<UsersController> logger)
    {
        _userService = userService;
        _logger = logger;
    }
    // Получение пользователей с пагинацией
    [HttpGet]
    public async Task<ActionResult<IEnumerable<UserDto>>> Get(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 10)
    {
        var users = await _userService.GetAllAsync(page, pageSize);
        return Ok(users);
    }
    // Получение конкретного пользователя
    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetById(int id)
    {
        var user = await _userService.GetByIdAsync(id);
        if (user == null)
        {
            _logger.LogWarning("User {Id} not found", id);
            return NotFound();
        }
        return Ok(user);
    }
    // Создать пользователя, контент из тела POST в формате json
    [HttpPost]
    public async Task<ActionResult<UserDto>> Create([FromBody] CreateUserRequest request)
    {
        // Валидация автоматическая благодаря [ApiController]
        var user = await _userService.CreateAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }
    // Обновить данные пользователя. Так же из тела POST в формате json, но id указывается в Query
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateUserRequest request)
    {
        var success = await _userService.UpdateAsync(id, request);
        if (!success)
        {
            return NotFound();
        }
        return NoContent();
    }
    // Удалить пользователя. 
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        var success = await _userService.DeleteAsync(id);
        if (!success)
        {
            return NotFound();
        }
        return NoContent();
    }
}
```

## Фильтры

Фильтры позволяют вклиниваться между этапами обработки запроса и выполнять код.

Зачем нужны фильтры:

- Избежать дублирования кода (cross-cutting concerns).
- Обрабатывать сквозные задачи: логирование, кэширование, авторизация, обработка ошибок.
- Выполнять код до/after выполнения действия контроллера.

Фильтры vs Middleware:

- Middleware работает на уровне всего конвейера (глобально для всех запросов).
- Фильтры работают на уровне действий контроллера (можно применить к конкретному действию или контроллеру).

Middleware - Фильтры
Работает на уровне конвейера (все запросы) - Работает на уровне действий контроллера
Не имеет доступа к контексту контроллера - Имеет доступ к контексту действия (аргументы, результат)
Не может быть применён к конкретному действию - Можно применить к конкретному действию или контроллеру

## Типы фильтров и порядок выполнения

- Authorization Filter - выполняется первый, проверяет авторизованность пользователя, может прервать pipeline
- Resource Filter - после авторизации, до связывания данных, может работать с кешированием.
- Action Filter - до или после api метода, может менять аргументы и результат
- Exception Filter - выполняется, если было выброшено необработанное исключение  в api методе
- Result Filter - до или после выполнения Result Action (например перед или после сериализации результата)

Порядок выполнения: 

`Authorization → Resource → Model Binding → Action → (исключение → Exception) → Result → Result Filter`



## Кастомный фильтр

```c#
// Синхронный Action Filter
public class LogActionFilter : IActionFilter
{
    private readonly ILogger<LogActionFilter> _logger;

    public LogActionFilter(ILogger<LogActionFilter> logger)
    {
        _logger = logger;
    }

    public void OnActionExecuting(ActionExecutingContext context)
    {
        _logger.LogInformation("Выполняется действие: {ActionName}", 
            context.ActionDescriptor.DisplayName);
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        _logger.LogInformation("Действие завершено: {ActionName}", 
            context.ActionDescriptor.DisplayName);
    }
}

// Асинхронный Action Filter
public class AsyncLogActionFilter : IAsyncActionFilter
{
    private readonly ILogger<AsyncLogActionFilter> _logger;

    public AsyncLogActionFilter(ILogger<AsyncLogActionFilter> logger)
    {
        _logger = logger;
    }

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context, 
        ActionExecutionDelegate next)
    {
        _logger.LogInformation("До выполнения действия");
        var resultContext = await next(); // выполнение действия
        _logger.LogInformation("После выполнения действия");
    }
}
```

Использование кастомного фильтра:

```c#
// На конкретное действие
[ServiceFilter(typeof(LogActionFilter))]
[HttpGet]
public IActionResult Get() { ... }

// На весь контроллер
[ServiceFilter(typeof(LogActionFilter))]
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase { ... }

// Глобально для всех контроллеров
builder.Services.AddControllers(options =>
{
    options.Filters.Add<LogActionFilter>();
});
```

Важно: для фильтров с зависимостями из DI используй `[ServiceFilter]` или `[TypeFilter]`, а не `[LogActionFilter]` напрямую.

Пример кастомного `ExceptionFilter`

```c#
public class CustomExceptionFilter : IExceptionFilter
{
    private readonly ILogger<CustomExceptionFilter> _logger;

    public CustomExceptionFilter(ILogger<CustomExceptionFilter> logger)
    {
        _logger = logger;
    }

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Необработанное исключение");

        // Формируем ответ в формате ProblemDetails
        context.Result = new ObjectResult(new
        {
            Title = "Произошла ошибка",
            Status = 500,
            Detail = context.Exception.Message
        })
        {
            StatusCode = 500
        };

        context.ExceptionHandled = true; // помечаем, что исключение обработано
    }
}
```

## Глобальная обработка ошибок

Фильтры не перехватывают исключения, возникшие в middleware (например, в маршрутизации или model binding). Для глобальной обработки всех исключений используется Exception Handling Middleware

```c#
var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage(); // подробная информация об ошибке
}
else
{
    app.UseExceptionHandler(options =>
    {
        options.Run(async context =>
        {
            context.Response.StatusCode = 500;
            context.Response.ContentType = "application/json";

            var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
            var response = new
            {
                Title = "Произошла ошибка",
                Status = 500,
                Detail = exception?.Message
            };

            await context.Response.WriteAsJsonAsync(response);
        });
    });
}
```

Можно сделать кастомный обработчик ошибок глобальных:

```c#
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Необработанное исключение");
            await HandleExceptionAsync(context, ex);
        }
    }

    private static Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = 500;

        var response = new
        {
            Title = "Произошла ошибка",
            Status = 500,
            Detail = exception.Message
        };

        return context.Response.WriteAsJsonAsync(response);
    }
}

// Регистрация в Program.cs (самый первый middleware)
app.UseMiddleware<GlobalExceptionMiddleware>();
```

Порядок имеет значение: middleware глобальной обработки ошибок должен быть самым первым в конвейере, чтобы перехватывать исключения из всех последующих middleware.

## EF Core интеграция

Сначала регистрируем контекст в DI

```c#
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

После чего через конструктор класса, который будет пользоваться контекстом базы данных просим DI передать его:

```c#
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly AppDbContext _context;
    private readonly ILogger<UsersController> _logger;

    public UsersController(AppDbContext context, ILogger<UsersController> logger)
    {
        _context = context;
        _logger = logger;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<User>>> Get()
    {
        var users = await _context.Users.AsNoTracking().ToListAsync();
        return Ok(users);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<User>> GetById(int id)
    {
        var user = await _context.Users.FindAsync(id);
        if (user == null)
        {
            return NotFound();
        }
        return Ok(user);
    }

    [HttpPost]
    public async Task<ActionResult<User>> Create([FromBody] CreateUserRequest request)
    {
        var user = new User
        {
            Name = request.Name,
            Email = request.Email
        };

        _context.Users.Add(user);
        await _context.SaveChangesAsync();

        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateUserRequest request)
    {
        var user = await _context.Users.FindAsync(id);
        if (user == null)
        {
            return NotFound();
        }

        user.Name = request.Name;
        user.Email = request.Email;
        await _context.SaveChangesAsync();

        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        var user = await _context.Users.FindAsync(id);
        if (user == null)
        {
            return NotFound();
        }

        _context.Users.Remove(user);
        await _context.SaveChangesAsync();

        return NoContent();
    }
}
```

Важные моменты при работе с EF Core в контроллерах

- AsNoTracking() для read-запросов — отключает отслеживание изменений, повышает производительность.
- async/await — все операции с БД должны быть асинхронными (ToListAsync(), SaveChangesAsync()).
- Обработка DbUpdateException — при сохранении могут возникнуть ошибки (нарушение уникальности, внешних ключей). Их нужно обрабатывать.
- Время жизни DbContext — регистрируется как Scoped (один на запрос), что соответствует времени жизни HTTP-запроса.