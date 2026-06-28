# Экзамен ASP.NET Core — П-318. Теория по темам (подробно)

Материал для подготовки студентов и опора преподавателю в разговоре. По каждой теме:
**что это → как было у нас (с кодом) → почему именно так → подводные камни → о чём поговорить**.
Список тем — в [Exam-Topics.md](Exam-Topics.md).

Формат экзамена — разговор. От студента нужно объяснить **идею своими словами** и показать,
что понимает, **зачем** оно нужно. Дословные определения не требуются.

---

## 1. Как устроено веб-приложение. Razor Pages vs MVC vs Web API

### Что это
Веб-приложение обрабатывает запросы по схеме «запрос → обработка → ответ». Клиент (браузер
или другой фронтенд) отправляет HTTP-запрос, сервер его обрабатывает и возвращает ответ.
ASP.NET Core предлагает три способа организовать серверный код — мы прошли все три, и в
[PersonalAccount/Program.cs](https://github.com/TOP-P-318/PersonalAccount/blob/master/Program.cs)
эти схемы записаны прямо в комментариях.

### Как было у нас

**Razor Pages (Notes).** Страница и её логика рядом. Поток:
```
Client → Server → Page → PageModel → On[GET/POST] → Update(Page) → Client
```
Браузер запрашивает страницу → отрабатывает `OnGet`/`OnPostDelete` в `IndexModel` →
рендерится HTML → уходит клиенту.

**MVC (PersonalAccount).** Запрос принимает контроллер:
```
Client → Server → Controller → выбирает действие → готовит Model → передаёт во View → HTML → Client
```
Один `CabinetController` обслуживает несколько представлений (`Index`, `Student`, `Admin`,
`Edit`). Логика отделена от разметки.

**Web API (Marketplace).** Контроллер возвращает данные, а не страницу:
```
Client → Backend → Controller → Service → Repository
Controller → [JSON] → Client
Client → Frontend (Next.js) → рисует HTML/CSS/JS
```

### Почему именно так
Главное различие — **кто формирует то, что увидит пользователь**:
- Razor Pages и MVC рендерят **готовый HTML на сервере** — браузер получает страницу.
- Web API отдаёт **данные (JSON)**, а HTML строит уже отдельный фронтенд на клиенте.

Отсюда и выбор под задачу: простой страничный сайт — Razor Pages; сайт с заметной логикой и
множеством экранов — MVC; бэкенд, который должен кормить разные клиенты (веб, мобилку), —
Web API + отдельный фронт. Marketplace специально сделали по третьей схеме, чтобы бэкенд и
фронтенд развивались независимо.

### Подводные камни / частые путаницы
- «MVC обязательно отдаёт HTML, а API — JSON» — да, в этом и суть разницы; технически это
  всё контроллеры, но базовый класс и назначение разные (`Controller` против `ControllerBase`).
- MVVM (упоминали) — это про фронтенд-паттерн, не путать с серверными MVC/Razor Pages.

### О чём поговорить
- «Над какими проектами работали и какой по какой схеме сделан?»
- «Чем ответ Web API отличается от ответа MVC?» → JSON против HTML.
- «Почему для Marketplace выбрали API + отдельный фронт?» → развязка фронта и бэка.

---

## 2. Dependency Injection и принципы проектирования

### Что это
Класс **не создаёт свои зависимости сам**, а получает их готовыми — обычно через конструктор.
Кто и какие объекты подставит, решает **IoC-контейнер** (Inversion of Control), встроенный в
ASP.NET Core. Мы регистрировали зависимости в `Program.cs`:
```csharp
builder.Services.AddScoped<INoteService, SqliteNoteService>();
builder.Services.AddDbContext<AppDbContext>(o => o.UseSqlite(connectionString));
```
А `IndexModel` просто объявляет, что ему нужен `INoteService`, и контейнер сам его передаёт.

### Почему именно так
Две связанные идеи:

**1. Работа через интерфейс = инверсия зависимостей (буква D в SOLID).**
Из ДЗ №1 (дословно близко): интерфейс `INoteService` определяется не тем, что *умеет*
`InMemoryNoteService`, а тем, что *нужно* `IndexModel`. Метод `Delete` появился в интерфейсе,
потому что у страницы возникла потребность удалять, — а не потому что список умеет удалять.
Зависимость «перевёрнута»: высокоуровневый код (страница) диктует контракт, низкоуровневый
(сервис) его реализует.

Практическая выгода — **подмена реализации без правок потребителя**. В Notes мы одной строкой
переключались между двумя реализациями `INoteService`:
```csharp
// builder.Services.AddSingleton<INoteService, InMemoryNoteService>();
builder.Services.AddScoped<INoteService, SqliteNoteService>();
```
`IndexModel` при этом не менялся вообще.

**2. Меньше связанность, проще тестировать.** Контейнер сам собирает граф зависимостей. Класс
зависит от абстракции, а не от конкретики, — его легко переиспользовать и подменять.

### Жизненные циклы (lifetime) — важная подтема
- **Singleton** — один объект на всё приложение. `InMemoryNoteService` был синглтоном,
  **потому что хранит список заметок прямо в себе** — если бы пересоздавался, данные бы
  терялись. В ДЗ №1 это подчёркнуто: «важно, чтобы сервис был синглтоном, иначе логика не
  будет работать».
- **Scoped** — один объект на HTTP-запрос. `SqliteNoteService` — scoped, **потому что завязан
  на `DbContext`**, а `DbContext` тоже scoped (живёт в рамках одного запроса). Сделать его
  синглтоном нельзя — «проект вообще не соберётся / будет конфликт».
- **Transient** — новый экземпляр на каждое обращение.

### Подводные камни
- Нельзя инжектить scoped-сервис в singleton (singleton переживёт запрос и «захватит»
  устаревший scoped-объект). Поэтому in-memory-сервис и sqlite-сервис имеют разные lifetime.
- В PersonalAccount видно осмысленный выбор: **репозитории и сервисы — Scoped** (работают с
  БД per-request), **мапперы — Singleton** (без состояния, чистое преобразование),
  **PasswordHasher — Singleton**.

### О чём поговорить
- «Что такое DI простыми словами и зачем он?» → не создаём зависимости сами, подменяемость,
  тестируемость.
- «Почему `IndexModel` работает с `INoteService`, а не с конкретным классом?» → инверсия,
  подмена реализации.
- «Почему in-memory был Singleton, а sqlite — Scoped?» → состояние внутри vs завязка на
  DbContext.

---

## 3. Слои приложения и модели данных

### Что это
Код разнесён по слоям с разной ответственностью. В Marketplace это видно чётче всего:
```
Controller → Service → Repository → DbContext / БД
```
- **Controller** (`ProductsController`) — тонкий: принял запрос, позвал сервис, вернул ответ
  с нужным статусом.
- **Service** (`ProductsService`, `PurchaseService`) — бизнес-логика.
- **Repository** (`Repo<TModel,TEntity>`) — доступ к данным, запросы к базе.

Параллельно одни и те же данные описываются **разными классами**:
- **Entity** (`ProductEntity`) — как строка лежит в БД.
- **Model** (`ProductModel`) — доменная модель внутри кода.
- **DTO**: в API это **Request/Response** (`CreateProductRequest`, `GetProductPreviewResponse`),
  в MVC — **ViewModel** (`StudentEditViewModel`, `AdminCabinetViewModel`).

Перекладывает между слоями **маппер** (`ProductMapper`):
```csharp
public override ProductModel MapToModel(ProductEntity entity) =>
    base.MapToModel(entity) with
    {
        Name = entity.Name,
        Price = entity.Price,
        // ...
    };
```

### Почему именно так
**Разделение ответственности.** Если свалить всё в контроллер — он превращается в «кашу».
Это прямым текстом сформулировано в ДЗ №5: контроллер не должен сам собирать данные из разных
репозиториев, сопоставлять аккаунты с профилями и формировать ViewModel — для этого заводим
сервис (`AdminCabinetService`).

**Почему разные модели, а не одна на всё:**
- БД-формат, внутренняя логика и то, что видит клиент, — это **три разные вещи** и **меняются
  независимо**. Меняешь структуру таблицы — трогаешь Entity, не ломая контракт API.
- Безопасность: наружу отдаём только нужные поля. В ДЗ №4 `Email` намеренно нередактируемый, а
  `StudentEditViewModel` содержит только `FullName`, `GroupName`, `PhotoUrl`.
- В ДЗ №5 отдельный `AdminCabinetStudentViewModel` завели, **хотя поля поначалу дублировали**
  профиль студента: «это всё равно разные модели с разной ответственностью» — у профиля и у
  отображения в админ-панели разное назначение, и со временем поля разойдутся.

### Подводные камни
- Соблазн отдавать Entity прямо клиенту — так делать не стоит: течёт схема БД и лишние поля.
- «Зачем маппер, если можно скопировать поля руками» — маппер централизует преобразование и
  убирает дублирование.

### О чём поговорить
- «Опиши путь от контроллера до базы в Marketplace.» → Controller → Service → Repository → БД.
- «Зачем и Entity, и Model, и ViewModel — почему не один класс?» → разная ответственность,
  независимость изменений, безопасность.
- «Почему в админ-кабинете завели отдельную ViewModel, хотя поля совпадали?» → разное
  назначение.

---

## 4. Базы данных через EF Core

### Что это
EF Core — ORM: работаем с базой как с C#-объектами, не пишем SQL руками.
- **DbContext** (`AppDbContext`) — точка входа к базе; **DbSet\<T>** — таблица.
- **Миграции** версионируют схему под изменения в коде:
  ```bash
  dotnet ef migrations add AddRoleSystem
  dotnet ef database update
  ```
- **SaveChanges** фактически отправляет накопленные изменения в БД.

### Как было у нас
Из ДЗ №1 — ключевая мысль про отложенность операций:
```csharp
var note = _context.Notes.Find(id);   // найти
_context.Notes.Remove(note);          // пометить на удаление (в памяти контекста)
_context.SaveChanges();               // только здесь реально уходит запрос в БД
```
> «Прошлые операции лишь формировали запрос, отправка происходит при `SaveChanges()`».

Фильтрация в `SqliteNoteService` (ДЗ №2) — строим запрос через `IQueryable`:
```csharp
var query = _context.Notes.AsQueryable();
switch (filter)
{
    case ImportanceFilter.ImportantOnly:    query = query.Where(n => n.IsImportant);  break;
    case ImportanceFilter.NotImportantOnly: query = query.Where(n => !n.IsImportant); break;
    // All — Where не добавляем
}
```

В Marketplace репозиторий обобщённый, с `AsNoTracking` для чтения:
```csharp
await set.AsNoTracking().Select(e => mapper.MapToModel(e)).ToArrayAsync();
```

### Почему именно так
- **Миграции** дают воспроизводимую, версионируемую схему: не правим базу руками, а
  описываем изменение в коде и применяем. В Marketplace миграции вынесли в **отдельный
  контейнер**, чтобы накатить схему один раз до старта сервисов.
- **SaveChanges одним разом** — эффективнее (один поход в БД) и безопаснее (это одна
  транзакция: либо все изменения, либо никакие).
- **Фильтровать запросом, а не в памяти.** `IQueryable.Where` транслируется в SQL → из базы
  приходят только нужные строки. Если сделать `ToList()` и потом `Where` в памяти — притащим
  всю таблицу и отфильтруем зря. В ДЗ №2 это прямо объясняется: запрос должен быть
  «минималистичным и не перегруженным». Для `InMemoryNoteService` об этом не парились — там
  всё в оперативке.
- **AsNoTracking** для чтения: EF не отслеживает такие сущности (не нужно — мы их не меняем),
  это быстрее и экономит память.

### Подводные камни
- Забыть `SaveChanges` → «удалил/добавил, а в базе не изменилось».
- `enum` в EF Core по умолчанию хранится **числом** (`Admin → 0`, `Student → 1`). Всплыло в
  ДЗ №5 при ручной вставке студентов в SQLite через SQL.
- Разница `List.Where` (LINQ to Objects, в памяти) vs `IQueryable.Where` (LINQ to Entities,
  уезжает в SQL) — одинаково выглядит, по-разному исполняется.

### О чём поговорить
- «Что делает SaveChanges и почему Add/Remove без него ничего не меняют?»
- «Что такое миграция и какие команды?»
- «Почему в sqlite-сервисе фильтровали запросом, а не в памяти?» → меньше данных из БД.

---

## 5. Формы и обработка запросов (GET/POST, model binding, PRG)

### Что это
Браузер меняет состояние сервера через формы. Ключевая пара методов:
- **GET** — получить/показать, **ничего не меняет** на сервере.
- **POST** — **изменяет** данные на сервере.

### Как было у нас
**Фильтр — GET** (ДЗ №2): значение уходит в URL.
```html
<form method="get">
    <select name="filter" onchange="this.form.submit()"> ... </select>
</form>
```
**Удаление — POST** (ДЗ №1): меняет данные, поэтому форма + скрытое поле с id.
```html
<form method="post" asp-page-handler="Delete">
    <input type="hidden" name="id" value="@Model.Id"/>
    <button type="submit">Удалить</button>
</form>
```
**Model binding** — атрибут `name="id"` связывает поле формы с параметром обработчика:
```csharp
public IActionResult OnPostDelete(int id, ImportanceFilter filter) { ... }
```

**PRG (Post-Redirect-Get)** — после POST не возвращаем HTML, а перенаправляем:
```csharp
_notes.Delete(id);
return RedirectToPage(new { filter });   // клиент заново запросит страницу (GET)
```

### Почему именно так
- **GET для фильтра** (из ДЗ №2, дословно близко): выбор фильтра не меняет данные, а только
  способ отображения. Плюс фильтр оказывается в URL — страницу можно обновить и расшарить.
- **POST для изменений** — семантически правильно: GET должен быть безопасным и повторяемым.
- **PRG** спасает от повторной отправки: после прямого ответа на POST нажатие F5 повторило бы
  запрос (удаление/создание ещё раз). После `RedirectToPage` обновление страницы — это просто
  GET. Заодно **сохраняем фильтр**: его прокидываем во все обработчики и в `RedirectToPage`,
  иначе после действия пользователя выкинет в режим «Все».

### Подводные камни
- Скрытое поле `filter` нужно в формах **действий** (удаление/важность), но **не** в самой
  форме фильтрации (ДЗ №2).
- Забыли прокинуть `filter` в Redirect → фильтр сбрасывается после каждого действия.
- `type="hidden"` — значение вычисляется автоматически, не вводится пользователем.

### О чём поговорить
- «Почему фильтр через GET, а удаление через POST?»
- «Зачем после POST делаем Redirect, а не возвращаем страницу сразу?» → PRG, защита от
  двойной отправки.
- «Как id заметки попадает в обработчик?» → скрытое поле + model binding.

---

## 6. Валидация данных

### Что это
Проверка того, что прислал пользователь. Правила вешаем атрибутами на **ViewModel**, проверяем
в контроллере через `ModelState`.

### Как было у нас
ДЗ №4 — форма редактирования профиля:
```csharp
public class StudentEditViewModel
{
    [Required(ErrorMessage = "Укажите имя")]
    public string FullName { get; set; } = string.Empty;
    [Required(ErrorMessage = "Укажите группу")]
    public string GroupName { get; set; } = string.Empty;
    public string? PhotoUrl { get; set; }   // необязательное
}
```
```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Edit(StudentEditViewModel model)
{
    if (!ModelState.IsValid) return View(model);   // не пропускаем невалидное, показываем ошибки
    // ... обновляем данные
    return RedirectToAction("Index");
}
```

### Почему именно так
- **Клиенту нельзя доверять.** Проверки в браузере (JS) легко обойти — отправить запрос мимо
  формы (через curl/Postman). Поэтому серверная проверка обязательна. В ДЗ №4 про смену пароля
  прямо сказано: проверку «новый пароль ≠ старый» делаем и на клиенте (для удобства), **и
  дублируем на сервере** — «юзер мог отправить запрос мимо формы».
- **ViewModel с правилами** держит валидацию рядом с данными формы и отделяет ввод от
  доменной модели.

### CSRF и antiforgery
`[ValidateAntiForgeryToken]` защищает POST-формы от **CSRF** (межсайтовой подделки запроса).
Идея: ASP.NET кладёт в форму скрытый одноразовый токен и проверяет его на сервере. Чужой сайт
не сможет подделать запрос от имени залогиненного пользователя — у него нет валидного токена.

### Подводные камни
- «Раз есть проверка на клиенте — на сервере не надо» — главная ошибка. Клиент = удобство,
  сервер = надёжность.
- Забыть `ModelState.IsValid` → в базу уедут невалидные данные.

### О чём поговорить
- «Как валидировали форму профиля?» → атрибуты + ModelState.
- «Почему проверок на клиенте недостаточно?» → клиент обходится.
- «От чего защищает antiforgery-токен?» → CSRF, чужие формы.

---

## 7. Аутентификация и авторизация

### Что это — две разные вещи
- **Аутентификация** — «кто ты» (вход по логину/паролю).
- **Авторизация** — «что тебе можно» (доступ по роли/правам).

### Как было у нас (cookie-based)
**Вход** (`AccountService.SignInAsync`): проверили пароль, собрали claims, выписали cookie.
```csharp
var claims = new List<Claim>
{
    new(ClaimTypes.NameIdentifier, account.Id.ToString()),
    new(ClaimTypes.Role, account.Role.ToString()),
    new(ClaimTypes.Email, account.Email),
};
var principal = new ClaimsPrincipal(new ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme));
await ctx.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, principal);
```
Настройка в `Program.cs`:
```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(o => { o.LoginPath = "/Account/Login"; });
```
**Авторизация по ролям** (ДЗ №5): роль кладём в claim, дальше ограничиваем доступ:
```csharp
[Authorize(Roles = "Admin")]
public async Task<IActionResult> Admin() { ... }
// или внутри кода:
if (User.IsInRole(AccountRole.Admin.ToString())) ...
```

**Хэширование паролей** (`AccountService`, `UsersService`):
```csharp
account.PasswordHash = hasher.HashPassword(account, password);                 // при регистрации
var result = hasher.VerifyHashedPassword(account, account.PasswordHash, input);// при входе
if (result == PasswordVerificationResult.Failed) return null;
```

### Почему именно так
- **Cookie + claims:** после входа сервер выписывает зашифрованную cookie, в которой лежат
  claims (Id, Email, Role). На каждом следующем запросе браузер шлёт cookie, и приложение по
  ней понимает, кто это, — не спрашивая пароль повторно.
- **Роли** — механизм разграничения прав: студент видит свой кабинет, админ — админский. Роль
  попала в claim при входе, поэтому `[Authorize(Roles="Admin")]` срабатывает.
- **Храним хэш, а не пароль.** Хэш — необратимое преобразование. При входе хэшируем введённое
  и сравниваем с сохранённым. Если базу украдут — паролей не видно. Хранить пароль открытым
  нельзя (утечка = компрометация всех аккаунтов, тем более люди переиспользуют пароли).

### Порядок в пайплайне
```csharp
app.UseAuthentication();  // сначала: определить, кто пользователь (разобрать cookie)
app.UseAuthorization();   // потом: проверить, можно ли ему сюда
```
Нельзя проверить права, не зная, кто перед нами, — отсюда именно такой порядок.

### Подводные камни
- Путают аутентификацию и авторизацию — это разные шаги (кто ты / что можно).
- 401 vs 403: 401 — не аутентифицирован (не вошёл), 403 — вошёл, но прав нет. В Marketplace
  cookie-обработчик специально возвращает эти коды вместо редиректа на логин (это же API).
- «Хэш можно расшифровать» — нет, это односторонняя функция; сравнивают хэши, а не
  восстанавливают пароль.

### О чём поговорить
- «Чем аутентификация отличается от авторизации?»
- «Почему храним хэш, а не пароль?»
- «Как ограничили админ-кабинет только админам?» → роль в claim + `[Authorize(Roles)]`.
- (для сильных) «Почему `UseAuthentication` идёт раньше `UseAuthorization`?»

---

## 8. Конфигурация и секреты

### Что это
Настройки приложения вынесены из кода. Несекретные — в `appsettings.json`, читаем через
`IConfiguration` или **Options-паттерн**. Секретные — отдельно, мимо Git.

### Как было у нас
Строка подключения и SMTP — в конфиге, через Options:
```csharp
builder.Services.Configure<SmtpSettings>(builder.Configuration.GetSection("Smtp"));
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlite(builder.Configuration.GetConnectionString("SqliteDefaultConnection")));
```
Секреты (пароль Google для писем, логин/пароль админа из `DbBootstrap`) — в **user-secrets**
(ДЗ №5):
```bash
dotnet user-secrets set "Smtp:Password" "<App Password>"
dotnet user-secrets set "DbBootstrap:Password" "<пароль админа>"
```
В Marketplace — через **переменные окружения / .env** (`compose.yaml`, `DotEnv.Load()`).

### Почему именно так
- **Секреты в репозитории = утечка.** Кто получит доступ к коду — получит и доступ к почте/
  аккаунтам. В ДЗ №5 буквально: если бы пароль попал в Git, чужие люди получили бы доступ к
  Google-аккаунту преподавателя. Поэтому секрет лежит локально на машине разработчика и **не**
  уходит на GitHub. Каждый разработчик заполняет секреты у себя.
- **Options-паттерн** даёт типизированные настройки (`IOptions<SmtpSettings>`), которые
  инжектятся как зависимость, — лучше, чем дёргать `IConfiguration["..."]` по строкам.
- **App Password** (Google) — отдельная иллюстрация: одноразовый отзываемый токен. Потерял —
  не страшно, выпустил новый; основной пароль аккаунта не светится.

### Подводные камни
- Разные конфиги под среду: `appsettings.Development.json` vs `appsettings.json`; `DbBootstrap`
  (засев тестового админа) включается **только в Development**.
- Положить секрет в `appsettings.json` «на минутку» — и он уже в истории Git.

### О чём поговорить
- «Где лежат настройки и строки подключения?»
- «Пароли мы в Git не кладём — куда тогда и почему?» → user-secrets / env, защита от утечки.

---

## 9. Web API и REST

### Что это
Web API — бэкенд, отдающий **данные (JSON)**, а не HTML. Контроллеры наследуют
`ControllerBase` и помечены `[ApiController]`.

### Как было у нас
`ProductsController` — классический CRUD:
```csharp
[ApiController]
[Route("api/products")]
public sealed class ProductsController(ProductsService productsService) : ControllerBase
{
    [HttpGet("previews")]                 // получить список
    public async Task<ActionResult<...>> GetPreviewsAsync() => Ok(await productsService.GetPreviewsAsync());

    [HttpPost]                            // создать
    public async Task<ActionResult<CreateProductResponse>> CreateAsync([FromBody] CreateProductRequest request)
        => CreatedAtRoute(Routes.Product.Name.Get, new { id = response.Id }, response);

    [HttpGet("{id:guid}")]                // получить по id
    public async Task<ActionResult<...>> GetAsync([FromRoute] Guid id)
        => response is null ? NotFound() : Ok(response);

    [HttpPut("{id:guid}")]  // обновить
    [HttpDelete("{id:guid}")] // удалить → NoContent() / NotFound()
}
```

### Почему именно так
- **HTTP-методы под CRUD:** GET — получить, POST — создать, PUT — обновить, DELETE — удалить.
  Стандарт делает API предсказуемым: не нужно гадать, что делает ручка.
- **Статус-коды вместо «просто объекта»:** `Ok` (200), `Created`/`CreatedAtRoute` (201),
  `NoContent` (204), `NotFound` (404), `Forbid` (403). Клиент по статусу понимает результат —
  это часть контракта. `CreatedAtRoute` ещё и возвращает в заголовке `Location` ссылку на
  созданный ресурс.
- **Откуда параметры:** `[FromBody]` — из тела (JSON), `[FromRoute]` — из URL (`{id}`).
- **Swagger/OpenAPI** — интерактивная документация: можно потыкать API руками без фронтенда.
- **Разделение фронта и бэка:** одно API кормит и веб, и потенциально мобилку.

### Подводные камни
- Вернуть 200 на «не найдено» — клиент не отличит успех от ошибки; для этого и `NotFound()`.
- `Controller` (с View) vs `ControllerBase` (только данные) — для API берём `ControllerBase`.

### О чём поговорить
- «Чем API отличается от MVC-сайта?» → JSON vs HTML.
- «Какие методы под какие операции?» → CRUD.
- «Зачем возвращать NotFound/NoContent, а не просто данные?» → статус — часть ответа.

---

## 10. Микросервисы и инфраструктура

### Что это
Marketplace разбит на отдельные сервисы: **ProductsApi**, **UsersApi**, **PurchaseApi**, плюс
общая библиотека **Shared** и контейнер **Migrations**. Перед ними — **API Gateway** на nginx.
Всё поднимается через **docker compose**.

### Как было у нас
**Gateway (nginx)** — единая точка входа, раскидывает по пути:
```nginx
location /api/products  { proxy_pass http://products_api; }
location /api/users     { proxy_pass http://users_api; }
location /api/purchases { proxy_pass http://purchases_api; }
```
**compose.yaml** — gateway + 3 API + Postgres + миграции, с порядком старта:
```yaml
gateway-api:   { image: nginx:alpine, ports: ["6139:80"], depends_on: [products-api, users-api, purchases-api] }
products-api:  { build: ..., expose: ["8080"], depends_on: [migrations] }
migrations:    { build: ..., depends_on: [database], restart: on-failure }
database:      { image: postgres:18-alpine, volumes: [database-data:/var/lib/postgresql] }
```
**Общая авторизация между сервисами** (`AuthenticationServiceCollectionEx`): cookie, выписанная
UsersApi, должна приниматься в ProductsApi → общий ключ **DataProtection** на общем томе:
```csharp
services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("/home/app/.aspnet/keys"))
    .SetApplicationName("MarketplaceApi");
```

### Почему именно так
- **Разбиение на сервисы** даёт независимость: каждый отвечает за свою область (товары /
  пользователи / покупки), их можно разрабатывать, разворачивать и масштабировать отдельно.
- **Gateway** прячет внутреннюю структуру за одним адресом и одним портом — клиент не знает,
  что внутри три сервиса. Заодно удобно для общих вещей (маршрутизация, проксирование cookie).
- **Docker** делает запуск воспроизводимым: «работает у меня» → «работает везде одинаково».
  `depends_on` + отдельный `migrations` гарантируют, что схема накатится до старта API.
- **Общий ключ DataProtection** — потому что cookie шифруется этим ключом; без общего ключа
  каждый сервис шифровал бы по-своему и не понимал чужую cookie.

### Образ vs контейнер (частый вопрос)
Образ — шаблон/рецепт (`products-api-318`); контейнер — запущенный из него экземпляр
(`products-api-318` running). Из одного образа можно поднять много контейнеров.
`ports` (наружу, как gateway `6139:80`) vs `expose` (только внутри сети compose, как API 8080).

### О чём поговорить
- «Зачем дробить на несколько сервисов, а не делать один большой?»
- «Что делает gateway?» → единая точка входа, маршрутизация.
- «Чем образ отличается от контейнера?»
- (для сильных) «Три сервиса проверяют одну cookie — как договорились?» → общий ключ
  DataProtection.

---

## 11. Транзакции и конкурентный доступ

### Что это
Операция, меняющая несколько вещей, должна примениться **целиком или никак**. Это транзакция:
`BeginTransaction → Commit` (успех) / `Rollback` (ошибка).

### Как было у нас
`PurchaseService.CreatePurchaseAsync` — покупка меняет четыре вещи разом:
```csharp
await using var transaction = await transactionManager.BeginTransactionAsync();
try
{
    var product = await productsRepo.FindByIdWithLockAsync(request.ProductId);
    if (product.Amount == 0) throw new InvalidOperationException("sold out");
    var buyer  = await usersRepo.FindByIdWithLockAsync(buyerId);
    if (buyer.Balance < product.Price) throw new InvalidOperationException("not enough money");
    var seller = await usersRepo.FindByIdWithLockAsync(product.SellerId);

    buyer  = buyer.WithDecBalance(product.Price);   // списать у покупателя
    seller = seller.WithIncBalance(product.Price);  // начислить продавцу
    product = product.WithDecAmount();              // уменьшить остаток
    // + создать запись о покупке

    await transaction.CommitAsync();
}
catch { await transaction.RollbackAsync(); throw; }
```
Блокировка строки (`Repo.FindByIdWithLockAsync`):
```sql
SELECT * FROM users WHERE id = @p0 FOR UPDATE
```

### Почему именно так
- **Атомарность.** Без транзакции при сбое на середине данные останутся неконсистентными:
  деньги у покупателя списали, а товар не уменьшили, или продавцу не начислили. Транзакция
  гарантирует «всё или ничего»: при ошибке `Rollback` откатывает все изменения.
- **Блокировка от гонки (race condition).** Два покупателя одновременно берут **последний**
  товар. Без блокировки оба прочитают `Amount = 1`, оба пройдут проверку и оба купят — товар
  «продан дважды». `SELECT ... FOR UPDATE` блокирует строку до конца транзакции: второй запрос
  ждёт, а когда дождётся — увидит уже `Amount = 0` и получит «sold out».

### Подводные камни
- Транзакция без `try/catch` + `Rollback` — при исключении изменения зависнут незакоммиченными.
- Блокировать нужно именно те строки, которые меняем (товар, покупатель, продавец) — иначе
  гонка остаётся.

### О чём поговорить
- «Почему покупку оборачиваем в транзакцию?» → атомарность.
- «Что будет без неё при ошибке на середине?» → неконсистентные данные.
- (для сильных) «От чего спасает `FOR UPDATE` при двух одновременных покупках?» → race
  condition, двойная продажа.

---

## 12. Основы HTTP

### Что это
HTTP — протокол «запрос–ответ» между клиентом и сервером. Сквозная база под авторизацию, API
и формы.

### Ключевое
- **Методы:** GET (получить), POST (создать/изменить), PUT (обновить), DELETE (удалить).
- **Статус-коды:** 200 OK, 201 Created, 204 No Content, 401 Unauthorized (не вошёл),
  403 Forbidden (вошёл, но нет прав), 404 Not Found, 500 — ошибка сервера.
- **Cookie** — данные на стороне клиента, шлются с каждым запросом; так держится состояние
  входа (наша cookie-авторизация).
- **JSON** — формат обмена данными между API и фронтендом.

### Почему важно
Это «алфавит» всего, что мы делали: авторизация = cookie + 401/403, Web API = методы + статусы
+ JSON, формы = GET/POST. Поняв HTTP, легко связать остальные темы.

### О чём поговорить
- «В чём разница 401 и 403?» → не аутентифицирован / нет прав.
- «Что такое cookie и зачем она в авторизации?»
- «В каком формате API общается с фронтом?» → JSON.

---

## 13. Фронтенд и связь с API

### Что это
Фронтенд Marketplace — на Next.js / React / TypeScript. Собирает UI из компонентов и берёт
данные из нашего API.

### Как было у нас
Серверный компонент-страница запрашивает товары и рендерит список:
```tsx
export default async function ProductsPage() {
    const products = await ProductsApi.getAll();
    return <ProductCardsList cards={products} />;
}
```
Поход в API (axios):
```ts
const response = await axios.get<ProductPreview[]>(routes.productsApi.products.get.all, { baseURL });
return response.data;
```
Компонент с **props** и **типами**:
```tsx
export type ProductCardProps = Omit<ProductPreview, 'id'>;
export default function ProductCard({ name, previewUrl, createdAt, updatedAt }: ProductCardProps) { ... }
```
```ts
export default interface ProductPreview {
    id: UUID; name: string; previewUrl: string | null; createdAt: Date; updatedAt: Date;
}
```

### Почему именно так
- **Компоненты** = переиспользуемые кирпичики UI (карточка товара, список карточек, шапка).
  Собираешь страницу из них, а не пишешь монолитный HTML.
- **Props** — входные данные компонента, делают его настраиваемым и переиспользуемым.
- **Типы TypeScript** (`ProductPreview`) описывают форму данных от API: автодополнение в IDE +
  ошибки ловятся при написании кода, а не в рантайме. `Omit<ProductPreview,'id'>` — пример
  переиспользования типа: карточке id не нужен.
- Это замыкает картину: **Web API отдаёт JSON → фронтенд его типизирует и рисует**.

### Подводные камни
- Тема мягкая, без глубокого погружения — достаточно понимания «компонент + props + поход в
  API за JSON».

### О чём поговорить
- «Что такое компонент и props на примере карточки товара?»
- «Как фронт получает список товаров?» → axios GET на `/api/products`, JSON → карточки.
- «Зачем типы, если можно без них?» → меньше ошибок, автодополнение.

---

## 14. Git и рабочий процесс

### Что это
То, как мы вели разработку и сдавали домашки.

### Как было у нас
- **Fork** — своя копия учебного репозитория с правами на запись.
- **origin vs upstream:** `origin` — свой форк (туда пушим), `upstream` — учебный репозиторий
  (оттуда тянем свежие изменения с занятий):
  ```bash
  git fetch upstream
  git checkout -b lessonX-XX upstream/lessonX-XX
  ```
- **Цикл изменений:** `git add .` → `git commit -m "..."` → `git push`.
- **Ветки:** домашку делали в отдельной ветке `homework-xxx` от ветки занятия, не от master.
- **Pull Request** — домашку сдавали ссылкой на **открытый** PR (ветка `homework-xxx` → ветка
  занятия в своём же форке).

### Почему именно так
- **Ветки** изолируют работу: эксперименты и домашка не ломают рабочую/основную ветку.
- **Fork** даёт право писать в свою копию (в чужой репозиторий писать нельзя).
- **origin/upstream** — это просто две «переменные» с адресами репозиториев: своего и
  исходного. Так удобно и пушить своё, и подтягивать чужие обновления.
- **Pull Request** — способ показать и обсудить изменения (виден diff) перед вливанием. Ровно
  так устроена работа в реальных командах: не пушим напрямую в общую ветку, а предлагаем
  изменения через PR.

### Подводные камни
- Делать домашку от master, а не от ветки занятия — задание может быть неактуально для master.
- Точка в `git add .` (пробел перед точкой) — добавляет все изменения.

### О чём поговорить
- «В чём разница origin и upstream в нашем процессе?»
- «Зачем сдавали домашку через Pull Request, а не пушем в master?» → обзор изменений, не ломаем
  общую ветку.
- «Базовый цикл: сделал правки → они на GitHub. Какие команды?» → add → commit → push.

---

## Памятка преподавателю

- Это **разговор**, а не зубрёжка. При заминке — наводящий вопрос или подсказка.
- Засчитываем, если студент объяснил **идею своими словами** и понимает **зачем** оно нужно;
  дословные формулировки не требуем.
- **5–10 минут** на человека: реально 1 основная тема + 2–3 уточняющих вопроса.
- Распределение по уровню: база — темы 1–5; средний — + 7, 9; сильный — 10–11.
- Тема сложности алгоритмов (ДЗ №3) — по минимуму или мимо.
