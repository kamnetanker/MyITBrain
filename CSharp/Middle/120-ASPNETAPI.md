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

## IHost и фоновые процессы

Положим, что стоит задача сделать сервис, который работает не по команде, а постоянно, но при этом приложение должно предоставлять API для взаимодействия с внешним миром (например, выводить туда метрики для Prometheus, управлять конфигурацией сервиса, передавать ему какие-то новые данные и т.п.), тогда нам необходимо в параллель с основным API добавить BackgroundService и управлять этим будет встроенный `IHost`.

`IHost` - механизм управления жизненным циклом компонент программы, внедрением зависимостей, конфигурацией (читает `appsettings.json`) и логированием(через `ILogger`).

`ASP.NET Core` уже имеет этот механизм в настроенном и подготовленном для взаимодействия виде, т.е. создавая `WebApplication` мы сразу получаем механизм `IHost`.

**Как это работает?**

В самом верху иерархии находится `IHostedService`, который определяет методы взаимодействия с сервисами, которые регистрируются в приложении. Можно написать класс, который реализует этот интерфейс, но в таком случае придётся самостоятельно реализовать его методы, что может сильно замедлить разработку и привести к ошибкам.

На такой случай разработчики заготовили для других разработчиков классы, которые реализуют этот интрефейс, например `BackgroundService`, у него есть сигнатура метода `ExecuteAsync(CancellationToken)` который необходимо переопределить в своём классе. Вся остальная инфраструктурная работа, не связанная с решением непосредственной задачи уходит на плечи разработчиков механизма `IHost`. 

```c#
public interface IHostedService{
    // Тут куча методов, которые нас не интересуют
    public async Task ExecuteAsync(CancellationToken token); // Метод, который интересует уже нас
}
public abstract class BackgroundService : IHostedService{ // Он реализует интерфейс
   
    // Тут что-то ещё реализованное заранее разрабами .Net
    // ....
    // А вот тут сигнатура метода, который необходимо переопределить
     public virtual async Task ExecuteAsync(CancellationToken token); 
}
public sealed class MyBackgroundService : BackgroundService {
    public override async Task ExecuteAsync(CancellationToken token){
        // Тут уже пишем наш сервис. Приведу пример, как рекомендует это структурировать Microsoft в своей документации
        try
        {
            while(!stoppingToken.IsCancellationRequested){
                // чота делаем
                await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken); // Вот здесь может возникнуть исключение
            }
        }
        catch (OperationCanceledException) // А здесь мы отловим исключение из ожидания
        {
            // Здесь они в документации говорят, что возвращать не-нуль нет необходимости, поскольку данное исключение "ожидаемое", т.е. мы ожидаем, что сервис однажды закроется.
        }
        catch(Exception ex){
             logger.LogError(ex, "{Message}", ex.Message);

            // Terminates this process and returns an exit code to the operating system.
            // This is required to avoid the 'BackgroundServiceExceptionBehavior', which
            // performs one of two scenarios:
            // 1. When set to "Ignore": will do nothing at all, errors cause zombie services.
            // 2. When set to "StopHost": will cleanly stop the host, and log errors.
            //
            // In order for the Windows Service Management system to leverage configured
            // recovery options, we need to terminate the process with a non-zero exit code.
            // Коротко переводя: мы должны вернуть не нулевое значение, чтобы система поняла,
            // что это не штатное поведение.
            // Собственно говоря, если следовать правилам отлова ошибок из конспекта /Junior/060-Исключения.md
            // То в данное место попадёт только необрабатываемая ошибка, а тут уж некуда деваться, приложение упало. 
            Environment.Exit(1);
        }
    }
}
```

**Собственно, как зарегистрировать фоновый сервис в `WebApplication`?**

```c#
// Регистрация фонового сервиса
builder.Services.AddHostedService<MyBackgroundService>(); // Всё. 
// В момент app.Run(); для этого сервиса будет вызван ExecuteAsync и он начнёт фунциклировать.
```

## IOptions и управление конфигурацией

**А как управлять этим фоновым сервисом?**

Для этого существует семейство интерфейсов `IOptions`

`IOptions<T>` - простая конфигурация, не обновляется после старта приложения.

`IOptionsSnapshot<T>` - конфигурация, которая обновляется с каждым HTTP запросом, в рамках одного запроса не изменяется. Это полезно для контроллеров. Если какой-то контроллер начал работу с текущей версией конфигурации, но она изменилась в процессе обработки - ничего страшного не произойдёт, контроллер закончит обработку со старой версией, а следующую начнёт с изменённой. Но для сервиса, работающего постоянно данный подход не применим. 

`IOptionsMonitor<T>` - обновляется при каждом изменении файла конфигурации и создаёт событие, что конфигурация изменилась, требуется лишь подписаться на это событие и обработать его.

Ниже в полном примере будет показано как пользоваться всеми этими конфигурациями без лишней воды вокруг. 

```csharp
// 1. Модель конфигурации (общая для всех примеров)
public class MySettings
{
    public string Name { get; set; } = "Default";
    public int MaxItems { get; set; } = 10;
}
// 2. IOptions<T> — фиксированная конфигурация на всё время работы приложения
// Регистрация. Конфигурация берётся из appsetings.json автоматически, отдельно нигде это указывать не надо.
builder.Services.Configure<MySettings>(
    builder.Configuration.GetSection("MySettings")
);

// Внедрение в контроллере (или любом другом сервисе)
public class MyController : ControllerBase
{
    private readonly MySettings _settings;

    public MyController(IOptions<MySettings> options)// Сюда попадает ТОЛЬКО секция MySettings, другие секции конфигурации сюда НЕ попадут, опять же DI делает это за нас.
    {
        _settings = options.Value; // Значение фиксируется при создании контроллера
    }
}
// В фоновом сервисе (не обновляется)
public class MyBackgroundService : BackgroundService
{
    private readonly MySettings _settings;

    public MyBackgroundService(IOptions<MySettings> options)
    {
        _settings = options.Value;
    }
}

// 3. IOptionsSnapshot<T> — обновляется на каждый HTTP-запрос (для контроллеров)
// Регистрация — та же, что и для IOptions
builder.Services.Configure<MySettings>(
    builder.Configuration.GetSection("MySettings")
);

// Внедрение в контроллер
public class MyController : ControllerBase
{
    private readonly IOptionsSnapshot<MySettings> _snapshot;

    public MyController(IOptionsSnapshot<MySettings> snapshot)
    {
        _snapshot = snapshot;
    }

    [HttpGet]
    public IActionResult Get()
    {
        var settings = _snapshot.Value; // Актуальная конфигурация на момент запроса
        return Ok(settings);
    }
}
// По сути DI проверяет при каждом запросе: изменилась ли конфигурация (по метаданным файла)? Если нет, то он отправляет то, что уже есть, не тратя системные ресурсы на перечитывание и создание новых экземпляров, а если изменилась, то перечитывает конфигурацию и обновляет в памяти её.

// Важно: IOptionsSnapshot<T> НЕ РАБОТАЕТ в фоновых сервисах, потому что они не привязаны к HTTP-запросам. Если попытаться использовать его в BackgroundService — значение будет зафиксировано при создании.

// 4. IOptionsMonitor<T> — обновляется при изменении файла (для фоновых сервисов и подписки)
// Регистрация — та же, что и для IOptions
builder.Services.Configure<MySettings>(
    builder.Configuration.GetSection("MySettings")
);

// Использование в фоновом сервисе
public class MyBackgroundService : BackgroundService
{
    private readonly IOptionsMonitor<MySettings> _monitor;
    private readonly ILogger<MyBackgroundService> _logger;

    public MyBackgroundService(IOptionsMonitor<MySettings> monitor, ILogger<MyBackgroundService> logger)
    {
        _monitor = monitor;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Подписка на изменения
        using var listener = _monitor.OnChange(settings =>
        { // Напоминалка: делегат захватывает локальные переменные. /Middle/020-ДелегатыИСобытия.md
            _logger.LogInformation("Конфигурация изменилась: Name={Name}, MaxItems={MaxItems}", 
                settings.Name, settings.MaxItems);
        });// Делегат мы используем для отслеживания, обновлять значения через него не нужно, это просто уведомление, что значения изменились.

        while (!stoppingToken.IsCancellationRequested)
        {
            var settings = _monitor.CurrentValue; // Всегда актуальное значение
            _logger.LogInformation("Текущая конфигурация: Name={Name}, MaxItems={MaxItems}", 
                settings.Name, settings.MaxItems);

            await Task.Delay(5000, stoppingToken);
        }
    }
}

// Использование в контроллере (тоже работает)
public class MyController : ControllerBase
{
    private readonly IOptionsMonitor<MySettings> _monitor;

    public MyController(IOptionsMonitor<MySettings> monitor)
    {
        _monitor = monitor;
    }

    [HttpGet]
    public IActionResult Get()
    {
        return Ok(_monitor.CurrentValue);
    }
}
```

Во всех примерах appsettings.json выглядел так:

```json
{
  "MySettings": {
    "Name": "DefaultName",
    "MaxItems": 10
  }
}
```

## PeriodicTimer - выполнение периодических задач

`PeriodicTimer` - рекомендуемый современный способ организации периодических действий, созданный для асинхронного контекста.

Пример без воды:

```c#
// 1. Простой пример внутри BackgroundService
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    using var timer = new PeriodicTimer(TimeSpan.FromSeconds(10));

    while (await timer.WaitForNextTickAsync(stoppingToken))
    {
        // Твоя работа, которая выполняется каждые 10 секунд
        Console.WriteLine($"Tick at {DateTime.Now}");
    }
}

// 2. Пример с изменяемым периодом
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    var periodMs = 5000;
    using var timer = new PeriodicTimer(TimeSpan.FromMilliseconds(periodMs));

    while (await timer.WaitForNextTickAsync(stoppingToken))
    {
        // Работа
        Console.WriteLine("Tick");
        
        // Если нужно изменить период — меняем свойство Period
        timer.Period = TimeSpan.FromSeconds(3);
    }
}

// 3. Пример с перехватом OperationCanceledException (для Task.Delay внутри цикла)
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));

    try
    {
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await Task.Delay(1000, stoppingToken); // может выбросить исключение
        }
    }
    catch (OperationCanceledException)
    {
        // Ожидаемое завершение
    }
}
```

## Комплексный пример

Задача: требуется отслеживать изменения в папке и управлять конфигурацией через WebAPI 

В чём смысл? Через `IHost` механизм регистрируется фоновый процесс, который просто смотрит содержимое папки периодически и управляется через основной API, который так же регистрируется.

Пошаговая реализация:

**Шаг 1 заводим проект**

```bash
dotnet new webapi -n FolderMonitor
cd FolderMonitor
dotnet add package Microsoft.Extensions.Hosting
dotnet add package Microsoft.Extensions.Options
```

**Шаг 2 Смотрим внимательно структуру проекта**

```bash
FolderMonitor/
├── Program.cs                           # Точка входа: создание и запуск хоста
├── appsettings.json                     # Основная конфигурация (MonitorConfig + ConnectionStrings)
├── appsettings.Development.json         # Переопределение для разработки
├── FolderMonitor.csproj                 # Файл проекта
├── Models/
│   └── MonitorConfig.cs                 # Класс с настройками: FolderPath, PeriodSeconds
├── Services/
│   └── FileSystemMonitorService.cs      # Фоновый сервис, наследующий BackgroundService
├── Controllers/
│   └── MonitorController.cs             # API для управления настройками
└── Utils/
    └── ConfigurationHelper.cs           # Методы для сохранения настроек в appsettings.json
```

**Шаг 3 определяем модель конфигурации**

`Models/MonitorConfig.cs`

```c#
namespace FolderMonitor.Models;

public class MonitorConfig
{
    public string FolderPath { get; set; } = "./monitored";  // Папка для мониторинга
    public int PeriodSeconds { get; set; } = 10;             // Интервал сканирования (сек)
}
```

**Шаг 4 переводим модель в конфигурацию**

`appsettings.json`

```json
{
  "MonitorConfig": {
    "FolderPath": "./monitored",
    "PeriodSeconds": 10
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  }
}
```