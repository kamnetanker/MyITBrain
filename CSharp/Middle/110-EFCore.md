# Погружение в EF Core 

## Формирование подключения к базе данных.

`1 способ` - `OnConfiguring`(для простых проектов):

```c#
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer("Server=localhost;Database=MyDb;User Id=sa;Password=MyPass;");
}
```

`2 способ` - построить контекст базы данных через AppBuilder

```c#
builder.Services.AddDbContext<AppDbContext>(
    options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"))
    );
```

в appsettings.json необходимо добавить строку подключения:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyDb;User Id=sa;Password=MyPass;"
  }
}
```

## Формирование модели данных

Для формирования модели данных EF Core использует подход из 3-х компонент:

1. Соглашение (convention) - набор правил для поиска типичных паттернов. Как пример из уже изученного модель данных на основании публичных свойств объекта
2. Атрибуты (data annotations) - набор атрибутов, используемый для явного обозначения того, как формируется модель и какие ограничения в SQL на неё накладываются.
3. Fluent API  - формирование модели через вызовы `ModelBuilder` и `OnModelCreating`

EF Core автоматически способен определить исходя из соглешений:

- Первичный ключ - свойство с именем Id или <тип>Id: user.Id или user.UserId - оба поля могут быть определены как первичный ключ
- Тип столбца - на основе типов свойств C#
- Отношения - на основе навигационных свойств.

Атрибуты используются напрямую в классе сущности:

```c#
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

public class User
{
    [Key]
    public int Id { get; set; }

    [Required]
    [MaxLength(100)]
    public string Name { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Column("RegistrationDate")]
    public DateTime CreatedAt { get; set; }
}
```

Fluent API (рекомендуется для сложных конфигураций) формируется в OnModelCreating и имеет наивысший приоритет в вопросе определения модели данных:

```c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>(entity =>
    {
        entity.HasKey(u => u.Id);
        entity.Property(u => u.Name).IsRequired().HasMaxLength(100);
        entity.Property(u => u.Email).IsRequired();
        entity.Property(u => u.CreatedAt).HasColumnName("RegistrationDate");
        entity.Property(u => u.IsActive).HasDefaultValue(true);
    });
}
```

Для уменьшения размера OnModelCreating можно вынести конфигурацию в отдельные классы, реализующие IEntityTypeConfiguration<TEntity>

```c#
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.Property(u => u.Name).IsRequired().HasMaxLength(100);
    }
}

// В OnModelCreating:
modelBuilder.ApplyConfigurationsFromAssembly(typeof(UserConfiguration).Assembly);
```

## Миграции

Миграции - способ обновления схемы базы данных вслед за изменением требований со стороны приложения, использующего эту базу данных.

Шаг 1. Установить инструменты для миграций:

```cmd
dotnet tool install --global dotnet-ef
```

Шаг 2. Добавить в проект зависимость

```cmd
dotnet add package Microsoft.EntityFrameworkCore.Design
```

Шаг 3. Создать миграцию

```cmd
dotnet ef migrations add InitialCreate
```

Эта команда создаёт папку Migrations с файлами, описывающими изменения (методы Up() и Down()).

По сути, миграция - отдельная программа, которую создаёт разработчик для приведения схемы базы данных к форме, используемой в приложении, в случае, если она изменилась.

Шаг 4. Применить миграцию

Шаг не порядковый, но принципиальный. Миграции применяются к схеме отдельно. 

```cmd
dotnet ef database update
```

Для продакшена рекомендуется генерировать SQL-скрипты — их можно проверить, настроить и передать DBA

```cmd
# Скрипт от пустой БД до последней миграции
dotnet ef migrations script

# Скрипт от конкретной миграции до последней
dotnet ef migrations script AddNewTables

# Идемпотентный скрипт (проверяет, какие миграции уже применены)
dotnet ef migrations script --idempotent
```

Что происходит при применении:

- EF Core сравнивает модель с последней миграцией
- Генерирует SQL-скрипт для обновления схемы
- Выполняет его в базе данных
- Сохраняет состояние миграции в таблице __EFMigrationsHistor

**Практическая реализация миграций.**

---

**Шаг 1. Определим исходное состояние модели данных:**

```c#
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
}

public class AppDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        => optionsBuilder.UseSqlServer("YourConnectionString");
}
```

---

**Шаг 2. Изменяем модель, добавляя свойство к пользователям**

```c#
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string PhoneNumber { get; set; }  // новое поле
}
```

---

**Шаг 3. Создание миграции**

```cmd
dotnet ef migrations add AddPhoneNumberToUser
```

В папке Migrations появится три файла:

`XXXXXXXXXXXXXX_AddPhoneNumberToUser.cs` — основной файл миграции с методами Up() и Down().

`XXXXXXXXXXXXXX_AddPhoneNumberToUser.Designer.cs` — метаданные для EF Core.

`AppDbContextModelSnapshot.cs` — снимок текущей модели (используется для сравнения при следующих миграциях).

Содержимое файла миграции (упрощенно):

```c#
public partial class AddPhoneNumberToUser : Migration
{
    // Накат миграции
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "PhoneNumber",
            table: "Users",
            nullable: true);
    }

    // Откат миграции
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "PhoneNumber",
            table: "Users");
    }
}
```

---

**Шаг 4a. Применение миграции (Вариант для Dev среды)**

```cmd
dotnet ef database update
```

EF Core сравнит состояние БД с последней миграцией, применит недостающие изменения и запишет факт применения в таблицу `__EFMigrationsHistory`

---

**Шаг 4b. Применение миграции (Вариант для прода)**

Вместо прямого применения в продакшене рекомендуется генерировать SQL-скрипты. Это безопаснее, так как скрипт можно проверить, настроить и отдать DBA

```cmd
# Скрипт от пустой БД до последней миграции
dotnet ef migrations script

# Скрипт от конкретной миграции до последней
dotnet ef migrations script AddPhoneNumberToUser

# Если контекстов много, следует указывать для какого создаётся миграция
dotnet ef migrations add AddPhoneNumberToUser --context AppDbContext

# Идемпотентный скрипт (проверяет, какие миграции уже применены)
dotnet ef migrations script --idempotent
```

Идемпотентный скрипт особенно полезен, если ты не знаешь, какая миграция последняя применена, или разворачиваешься на несколько БД с разным состоянием

---

**Шаг 5. Применение скриптов**

Сгенерированный SQL-скрипт можно выполнить через SQL Server Management Studio, Azure Data Studio или любой другой инструмент.

```sql
-- Фрагмент сгенерированного скрипта
ALTER TABLE [Users] ADD [PhoneNumber] nvarchar(max) NULL;
GO

INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
VALUES (N'20240715093000_AddPhoneNumberToUser', N'8.0.0');
GO
```

---

## Отношения

База реляционная? реляционная! а связи где? а вот они?

EF Core формируют реляционные связи через внешние ключи, поверх ключей надстраиваются навигационные свойства, которые предоставляют объекто-ориентированный способ чтения и управления отношениями.

Свойства бывают:

- Reference navigation - ссылка на другую сущность. Реализует сторону "один"
- Collection navigation - коллекция связанных сущностей. Реализует сторону "многие".

Итого можно реализовать реляционные схема 1 к многим, многие к 1 к многим. 

`Один-ко-многим` (многоногим) (One-to-many)

```c#
// Principal (родитель) — сторона "один"
public class Blog
{
    public int Id { get; set; }
    public ICollection<Post> Posts { get; } = new List<Post>(); // collection navigation
}

// Dependent (дочерний) — сторона "многие"
public class Post
{
    public int Id { get; set; }
    public int BlogId { get; set; }      // внешний ключ (required)
    public Blog Blog { get; set; } = null!; // reference navigation
}
```

Обязательная vs необязательная связь: если внешний ключ не допускает NULL — связь обязательная (каждое Post должно быть связано с Blog). Если допускает — необязательная

`Многие-ко-Многим` (многоногоногим) (Many-to-Many)

Реализуется через двойную связь 1 ко многим через прослойку реляционную.

```c#
public class Post
{
    public int Id { get; set; }
    public List<PostTag> PostTags { get; } = [];
}

public class Tag
{
    public int Id { get; set; }
    public List<PostTag> PostTags { get; } = [];
}

// Join-сущность — таблица-соединитель
public class PostTag
{
    public int PostsId { get; set; }
    public int TagsId { get; set; }
    public Post Post { get; set; } = null!;
    public Tag Tag { get; set; } = null!;
}
```

В БД это создаст таблицу PostTag с составным первичным ключом из двух внешних ключей

В случае реализации неявной, потребуется Fluent API. 

```c#
public class Post
{
    public int Id { get; set; }
    public ICollection<Tag> Tags { get; set; }  // напрямую ссылаемся на Tag
}

public class Tag
{
    public int Id { get; set; }
    public ICollection<Post> Posts { get; set; }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Post>()
        .HasMany(p => p.Tags)
        .WithMany(t => t.Posts);
}
```

Т.к. соглашения не позволяют определить эту связь и какую таблицу требуется создать для этой связи при неявном объявлении. В таком случае - Fluent API.

## Загрузка связных данных

Проблема: базы данных бывают огромные. Неприлично огромные. Мало того если таблица большая, сложный запрос может вернуть таблицу кратно большую по размерам, чем исходные (через JOIN, например), что же тогда делать? 

`Первый вариант`: Ну если плевать и оперативной памяти как воздуха и она может вместить в себя хоть всю БД, то просто загрузить её Eager Loading (жадная загрузка): 

```c#
using (var context = new AppDbContext())
{
    // Загружаем блоги вместе с постами
    var blogs = context.Blogs
        .Include(b => b.Posts)              // один уровень
        .ThenInclude(p => p.Comments)       // вложенный уровень
        .ToList();
}
```

Когда использовать: когда знаешь, что связанные данные понадобятся сразу. Рекомендуется для веб-приложений — это предотвращает N+1 проблему. Минус: оператива. Много оперативы будет сожрано на это.

`Второй вариант`: Загружать только связные данные для конкретной сущности Explicit Loading (Явная загрузка)

```c#
using (var context = new AppDbContext())
{
    var blog = context.Blogs.First();
    
    // Явно загружаем посты для этого блога
    context.Entry(blog).Collection(b => b.Posts).Load();
}
```

Опять же, память, даже одна сущность может вести к большому количеству зависимостей и тянуть их все может быть нецелесообразно, лучше брать данные пачками по 100, например. 

Когда использовать: когда не знаешь заранее, понадобятся ли связанные данные, и хочешь загрузить их по требованию.

`Третий вариант`: Lazy Loading (Ленивая загрузка)

Смысл? Данные загружаются тогда, когда они нужны, а не всегда. Проблема N+1 здесь пахнет всеми своими сортами Г. Плюс нужна доп настройка. Крч не рекомендую, но покажу.

1. Устанавливаем пакет `Microsoft.EntityFrameworkCore.Proxies`
2. Включаем в момент конфигурации ленивую загрузку:

    ```c#
    optionsBuilder.UseLazyLoadingProxies()
              .UseSqlServer("YourConnectionString");
    ```

3. Навигационные свойства делаем виртуальными:

    ```c#
    public class Blog
    {
        public int Id { get; set; }
        public virtual ICollection<Post> Posts { get; set; }
    }
    ```

Когда использовать: в десктопных приложениях с небольшим количеством запросов. Не рекомендуется для веб-приложений, так как легко вызывает N+1 проблему и множество лишних round-trip к БД

Оптимизации потребления памяти: Pagination - получение данных "страницами". Есть два пути: 

1. `LIMIT offset_n, count_k` - скип offset_n строк и выборка count_k последующих

    А как в C# и EF Core? 

    ```c#
    // Пример: получаем 3-ю страницу (page = 3), размер страницы 10 записей (pageSize = 10)
    int page = 3;
    int pageSize = 10;
    int position = (page - 1) * pageSize; // 20

    var posts = await context.Posts
        .OrderBy(b => b.PostId)           // сортировка ОБЯЗАТЕЛЬНА
        .Skip(position)                   // пропускаем 20 записей
        .Take(pageSize)                   // берём 10
        .ToListAsync();                   // SELECT ... OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY
    ```

    Проблем у данного метода много. Если сортировка идёт не по ключу, а например по дате, записи могут быть разбросаны в выдаче и одна и та же запись может попастся и на первой и на второй странице при такой пагинации, а может и не попастся вовсе. Лучше тогда по дате сгруппировать и внутри группы отсортировать по ключу, а уже к этим данным применять пагинацию.

    Так же такая пагинация требует полной выборки данных, а это для SQL сервера накладно по ресурсам. Дизлайк, идём дальше

2. `WHERE` пагинация. - рекомендуемый способ, особенно для больших данных, т.к. WHERE фильтрует выборку. 

    ```c#
    // Запоминаем ID последней записи на текущей странице
    var lastId = 55; // например, последний PostId на предыдущей странице

    var nextPage = await context.Posts
        .OrderBy(b => b.PostId)
        .Where(b => b.PostId > lastId)    // берём только те, что больше lastId
        .Take(10)                         // берём 10
        .ToListAsync();                   // SELECT ... WHERE PostId > 55 ...
    ```

    Этот запрос чрезвычайно эффективен при наличии индекса на PostId и не чувствителен к параллельным изменениям в более ранних ID.

`Пример с загрузкой связанных данных`

Если нужно загружать связанные сущности (например, посты вместе с комментариями) постранично, комбинируй Include() с Skip()/Take():

```c#
var page = 1;
var pageSize = 100;

var blogsWithPosts = await context.Blogs
    .Include(b => b.Posts)           // загружаем связанные посты
    .OrderBy(b => b.Id)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

Здесь за один запрос с JOIN'ами подтянутся посты, но только для тех блогов, которые попали на текущую страницу. Include() применяется после пагинации, а не до неё.

`Практический API-контроллер с пагинацией (ASP.NET Core)`

```c#
[HttpGet]
public async Task<IActionResult> GetPosts(int page = 1, int pageSize = 10)
{
    var query = _context.Posts.AsNoTracking(); // отключаем отслеживание для read-запросов

    var totalCount = await query.CountAsync(); // общее количество записей

    var items = await query
        .OrderBy(p => p.CreatedAt)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();

    return Ok(new
    {
        TotalCount = totalCount,
        Page = page,
        PageSize = pageSize,
        Items = items
    });
}
```

## CRUD-операции и отслеживание изменений

EF Core не просто консьежка, это сраный контроллёр на максималках. Каждое обновление в контексте CRUD это некоторый пакет изменений, который сначала подготавливается, а потом фиксируется в базе данных. По логике похоже на работу с удалённым репозиторием Git. Сначала ты работаешь-работешь, делаешь свои локальные коммиты, а потом устраиваешь PUSH в репозиторий и крашишь прод, потому что не переключил ветку. 

Состояние отслеживания:

- Added - сущность в контексте, при фиксации будет INSERT
- Unchanged - сущность не менялась с момента загрузки
- Modified - сущность изменена, при фиксации будет UPDATE
- Deleted - сущность помечена на удаление, будет выполнен DELETE
- Detached - сущность не отслеживается контекстом.

`CREATE`

```c#
using (var context = new AppDbContext())
{
    var blog = new Blog { Name = "Новый блог" };
    context.Blogs.Add(blog);        // состояние: Added
    await context.SaveChangesAsync(); // INSERT
}
```

`READ`

```c#
// Получить все
var blogs = context.Blogs.ToList();

// Получить один
var blog = context.Blogs.FirstOrDefault(b => b.Id == id);

// С проекцией (только нужные поля)
var blogNames = context.Blogs.Select(b => new { b.Id, b.Name }).ToList();
```

`UPDATE`

```c#
using (var context = new AppDbContext())
{
    var blog = context.Blogs.Find(id);  // состояние: Unchanged
    blog.Name = "Новое имя";            // состояние: Modified
    await context.SaveChangesAsync();   // UPDATE
}
```

Важно: если сущность не отслеживается (например, пришла с фронтенда), нужно явно указать состояние:

```c#
context.Entry(blog).State = EntityState.Modified;
await context.SaveChangesAsync();
```

`DELETE`

```c#
using (var context = new AppDbContext())
{
    var blog = context.Blogs.Find(id);  // состояние: Unchanged
    context.Blogs.Remove(blog);          // состояние: Deleted
    await context.SaveChangesAsync();   // DELETE
}
```

`Отключение Change Tracking для read-запросов`

Для read-only сценариев (например, отчёты) имеет смысл отключить change tracking для повышения производительности

```c#
var blogs = context.Blogs
    .AsNoTracking()  // отключает отслеживание
    .ToList();
```

Можно так же отключить полностью, если микросервис не модифицирует данные:

```c#
optionsBuilder.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
```