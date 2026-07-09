# LINQ

`LINQ` `(Language Integrated Query)` - встроенный в C# язык запросов для обработки данных, аналогичный есть в JAVA (Stream API), но более куцый и созданный, чтобы не уступать шарпу в его фичах.

`LINQ` выполняет типовые операции по обработке данных напрямую в пространстве памяти приложения, без необходимости использовать дополнительные языки для обработки XML, например.

## Запросы

`LINQ` операция состоит из 3 основных частей:

1. Получение данных. Это может быть массив, коллекция, DBSet из EF Core или XML документ.
2. Создание запросы
   1. Откуда
   2. Что делаем (можно много раз)
   3. Возврат результата
3. Выполнение запроса и извлечение результата.

Важно: данные, за некоторыми исключениями, не обрабатываются в момент формирования запроса, они обрабатываются только в момент извлечения результата.

Пример простого запроса с использованием `LINQ`:

```c#
// 1. Источник данных
int[] scores = [97, 92, 81, 60];

// 2. Создание запроса (запрос пока НЕ выполнен)
IEnumerable<int> scoreQuery = from score in scores
                              where score > 80
                              select score;

// 3. Выполнение запроса (только здесь данные извлекаются)
foreach (var i in scoreQuery)
{
    Console.Write(i + " ");  // Вывод: 97 92 81
}
```

## Два синтаксиса описания запросов

1. Query Syntax - упрощённый декларативный вариант задания запроса:

```c#
int[] numbers = [97, 92, 81, 60];
IEnumerable<int> numQuery1 = from num in numbers
                             where num % 2 == 0
                             orderby num
                             select num;
```

2. Method Syntax - исходный вариант синтаксиса, практически полностью слизанный java семью годами позже.

```c#
IEnumerable<int> numQuery2 = numbers
                             .Where(num => num % 2 == 0)
                             .OrderBy(n => n);
```

Некоторые операции (например, Count, Max, Average) не имеют эквивалента в Query Syntax и должны быть вызваны как методы

Методы типа `Where`, `Select`, `OrderBy` — это методы-расширения (extension methods) для интерфейса `IEnumerable<T>`. Они определены в классе `System.Linq.Enumerable`. Чтобы они были доступны, нужно подключить пространство имён `using System.Linq;`

## Время выполнения

Важнейшая концепция, которую необходимо понять про запросы `LINQ` - момент времени, когда запрос выполняется.

```c#
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// Запрос создан, но НЕ выполнен
var query = numbers.Where(n => n > 2);

// Добавляем элемент в источник ПОСЛЕ создания запроса
numbers.Add(6);

// Запрос выполняется ТОЛЬКО здесь — и увидит 6!
foreach (var n in query)  // Результат: 3, 4, 5, 6
{
    Console.WriteLine(n);
}
```

Все методы, которые возвращают одиночное значение (singleton) — Count(), Max(), Min(), Average(), Sum(), First(), ToList(), ToArray(), исполняются в момент вызова, поскольку это получение результата. First(), Single(), Any() тоже немедленные.

```c#
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// Выполняется СРАЗУ — результат зафиксирован
var count = numbers.Where(n => n > 2).Count();  // count = 3

numbers.Add(6);  // Добавляем элемент ПОСЛЕ выполнения

// count всё ещё 3 — он уже посчитан и не изменится
// ToList() и ToArray() так же возвращают результат СРАЗУ
// Принудительное выполнение — результат кэшируется в списке
var cachedResult = numbers.Where(n => n > 2).ToList();
```

## Базовые операции

`Where` - фильтрация

Возврат только того, что удовлетворяет условию.

```c#
int[] numbers = [3, 6, 2, 1 ,7, 8, 10];

// Query Syntax
IEnumerable<int> evenNumbers = from n in numbers
                               where n%2 == 0
                               select n;

// Method Syntax
IEnumerable<int> evenNumbers2 = numbers.Where(n => n%2==0); // Здесь Inline делегат.
// По сути, запросы в Query Syntax это синтаксический сахар для Method Syntax. При сборке проекта, вся красота Query Syntax превращается в Method Syntax и набор делегатов.
```

`OrderBy()` и `OrderByDescending()` - сортировка по возрастанию или убыванию.

```c#
// Query Syntax
var sortedQuery = from n in numbers
                  orderby n
                  select n;

// Method Syntax
var sorted = numbers.OrderBy(n => n);        // по возрастанию
var sortedDesc = numbers.OrderByDescending(n => n); // по убыванию
```

Принимаются базовые типы для которых реализовано сравнение, либо `IComparable`

`Select()` - позволяет модифицировать данные.

```c#
var result = numbers
    .Where(n => n > 5)           // сначала фильтруем
    .OrderBy(n => n)             // потом сортируем
    .Select(n => n * 2);         // потом преобразуем
// Результат: 12, 16, 20, 24 (отфильтрованные, отсортированные, умноженные на 2)
```

Так же, как можно заметить по примерам, методы группируются и возвращают `IEnumerable<T>`. Цепочка, в целом, может быть сколь угодно большая, но надо понимать, что результат будет вычисляться в момент извлечения данных, что может стать статором для пользователя при обработке большого запроса.

Порядок указания методов обработки напрямую указывает, в каком порядке операции будут выполнятся, поэтому по возможности следует сокращать объём данных перед, например, сортировкой, поскольку фильтрация имеет сложноть `N`, а сортировка `N*Log(N)`

## Продвинутые операторы

`GroupBy` - группировка результата. При обрабоке объектов можно выделять группы и группировать объекты по каким-либо их полям.

```c#
var students = new[]
{
    new { Name = "Анна", Group = "A" },
    new { Name = "Борис", Group = "B" },
    new { Name = "Виктор", Group = "A" },
    new { Name = "Галина", Group = "C" },
    new { Name = "Дмитрий", Group = "B" }
};

// Query Syntax
var groupQuery = from s in students
                 group s by s.Group into g
                 select new { Group = g.Key, Count = g.Count() };

// Method Syntax
var groupMethod = students
    .GroupBy(s => s.Group)
    .Select(g => new { Group = g.Key, Count = g.Count() });

// Результат:
// Group A: 2 (Анна, Виктор)
// Group B: 2 (Борис, Дмитрий)
// Group C: 1 (Галина)
```

`group s by s.Group into g` можно прочитать как: сгруппируй(group) s по ключу(by) s.Group в (into) группу g.

После группировки в следующую ступень поступает `IEnumerable` группы g, содержащей ключ и коллекцию объектов, которая в группу входит.

В Method Syntax всё немного проще:
1. Возьми коллекцию
2. Укажи, по какому ключу группируемся. После этого шага будет передана IEnumerable коллекция, содержащая группы объектов по ключу. Т.е. множество групп, содержащих сгруппированные объекты
3. Преобразуй в какой-то вывод. В итоговое преобразование группы подаются по одному и доступны объекты в группе.

`Join` - объединение двух коллекций по ключу. Аналог SQL `INNER JOIN`

```c#
var customers = new[]
{
    new { Id = 1, Name = "Иван" },
    new { Id = 2, Name = "Мария" },
    new { Id = 3, Name = "Петр" }
};

var orders = new[]
{
    new { CustomerId = 1, Product = "Ноутбук" },
    new { CustomerId = 2, Product = "Телефон" },
    new { CustomerId = 1, Product = "Планшет" }
};

// Query Syntax
var joinQuery = from c in customers
                join o in orders on c.Id equals o.CustomerId
                select new { c.Name, o.Product };

// Method Syntax
var joinMethod = customers
    .Join(orders,
          c => c.Id,
          o => o.CustomerId,
          (c, o) => new { c.Name, o.Product });

// Результат:
// Иван: Ноутбук
// Иван: Планшет
// Мария: Телефон
```

Как воспользоваться?
1. Берём коллекцию к которой необходимо заджоинить новые данные по ключу: `from c in customers`
2. джоиним. Т.е. Включить o в orders (т.е. взять вторую коллекцию), по c.Id эквивалентному o.CustomerId: `join o in orders on c.Id equals o.CustomerId`
3. Вернуть новую коллекцию `select new {c.Name, o.Product};`. На последний этап подаются уже пары объектов c, o.

Method Syntax чуть проще. 
1. Взять коллекцию `customers`
2. Объединить `.Join(`
3. С чем? С orders `.Join(orders,`
4. По какому ключу? `c => c.Id,`
5. С каким ключом? `o => o.CustomerId,`
6. В каком порядке и что вернуть? `(c, o) => new {c.Name, o.Product});` 

`SelectMany` - разворачивание вложенных коллекций. Превращает вложенные коллекции в плоскую последовательность.

```c#
var families = new[]
{
    new { Family = "Ивановы", Members = new[] { "Иван", "Мария" } },
    new { Family = "Петровы", Members = new[] { "Петр", "Анна" } }
};

// Query Syntax
var allMembersQuery = from f in families
                      from m in f.Members
                      select $"{f.Family}: {m}";

// Method Syntax
var allMembersMethod = families
    .SelectMany(f => f.Members, (f, m) => $"{f.Family}: {m}");

// Результат:
// Ивановы: Иван
// Ивановы: Мария
// Петровы: Петр
// Петровы: Анна
```

По сути, в QuerySyntax происходит двойная выборка и проход по обеим выборкам. Читать как: `Возьми фамилии, внутри фамилии возьми члена фамилии, сформируй фамилия: имя. 

Method Syntax напротив, чуть сложнее для понимания: 
1. Возьми фамилии `families`
2. Множественная выборка. `.SelectMany`
3. По какому полю требуется сделать разворачивание `f=>f.Members`
4. Как сформировать результат? `(f,m) => $"{f.Family}: {m}"`

`Aggregate` - свёртывает последовательность в одно значение. Как Min(), Max(), Avg().

```c#
int[] numbers = [1, 2, 3, 4, 5];

// Сумма (похоже на Sum, но показывает принцип)
int sum = numbers.Aggregate((acc, n) => acc + n);  // 15

// Произведение
int product = numbers.Aggregate((acc, n) => acc * n);  // 120

// Строка с разделителями
string[] words = ["Привет", "мир", "LINQ"];
string sentence = words.Aggregate((acc, w) => acc + " " + w);  // "Привет мир LINQ"
```

Тут всё мега просто, Первый аргумент передаваемого делегата должен быть аккумулятором (т.е. результатом предыдущих вычислений), а второй - очередным аргументом.

## IEnumerable vs IQueryable

LINQ позволяет не только формировать вычисления прямо там, где запущено приложение, но и сформировать SQL запрос к базе данных, делегировав вычисления. Удобно для API, нагруженных серверов и прочего.

`IEnumerable<T>` запрос уже известен - работает в памяти, исполнение в момент получения результата вычислений.

`IQueryable<T>` - новое. Здесь запрос формируется к источнику данных (БД, Веб-сервис). Используется в LINQ to SQL, EF Core. Транслирует вызовы в язык источника данных и выполняется там (чаще всего SQL). Вычисления осуществляются на стороне источника. `Where`, `Select` и др. работают с Expression-деревьями источника данных, а не с делегатами.

```c#
// IEnumerable — фильтрация в памяти
IEnumerable<User> usersInMemory = dbContext.Users.ToList(); // все данные загружены
var result1 = usersInMemory.Where(u => u.Age > 18); // фильтрация в памяти

// IQueryable — фильтрация на стороне БД
IQueryable<User> usersQueryable = dbContext.Users; // запрос не выполнен
var result2 = usersQueryable.Where(u => u.Age > 18); // SQL с WHERE Age > 18
```

Важно запомнить. IEnumerable работает в памяти, IQueryable работает на стороне источника и это его проблемы, как обработать запрос.

## LINQ to Objects vs LINQ to SQL/EF Core

`LINQ to Object` - концепция применения `LINQ` к коллекциям в памяти приложения. Используемый тип: `IEnumerable`. Все операции выполняются с загруженными в память данными. Возвращает так же `IEnumerable`. Примитивный пример: `new[] {1,2,3}.Where(x => x>1)`

`LINQ to SQL/EF Core` - концепция трансляции запросов в язык источника (SQL) и делегация вопросов фильтрации, выборки, сортировки, модификации на стороне источника данных. Тип данных `IQueryable<T>`. Вся последовательность транслируется в SQL. Возвращаемое значение `IQueryable<T>`. Примитивный пример: `dbContext.Users.Where(u => u.Age > 18)`

```c#
// LINQ to Objects
var numbers = new[] { 1, 2, 3, 4, 5 };
var evenNumbers = numbers.Where(n => n % 2 == 0); // выполняется в памяти

// LINQ to SQL (через EF Core)
using (var context = new AppDbContext())
{
    var adults = context.Users
        .Where(u => u.Age >= 18)        // транслируется в SQL
        .Select(u => new { u.Name, u.Age })
        .ToList(); // выполняется при ToList()
}
```

## Вопросы производительности

1. Фильтруй как можно раньше. Сортировка, модификация, Join - дорогие по времени операции, лучше сделать так, чтобы они обрабатывали как можно меньше данных.
2. Если делигировал обработку данных на сторону SQL сервера, то не вызывай ToList() пока не закончишь обработку, ибо каждый раз при вызове ToList() данные загружаются в память приложения.
3. используй Select для проекции таблицы, поскольку тащить все столбцы, когда тебе нужно всего 2 - лишняя трата памяти и операций по перемещению, копированию и т.п.
4. N+1 проблема. В чём заключается? вызовы выборки данных внутри цикла. 

    ```c#
    // ❌ Плохо: N+1 запросов
    foreach (var order in context.Orders.ToList())
    {
        var customer = context.Customers.Find(order.CustomerId); // запрос на каждой итерации
    }

    // ✅ Хорошо: один запрос с Include
    var orders = context.Orders.Include(o => o.Customer).ToList();
    ```

5. Для сложных отчётов используй Dapper вместо EF Core. EF Core генерирует SQL, но для сложных запросов (CTE, оконные функции) лучше написать ручной SQL через Dapper.