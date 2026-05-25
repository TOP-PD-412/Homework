# Домашнее задание 4: Отображение и редактирование профиля студента

Сейчас мы можем только просматривать данные студента, однако мы не можем их редактировать. На одном из прошлых занятий мы реализовывали функционал по редактированию заметок. Примените полученные знания, и реализуйте аналогичный функционал в данном проекте.

## Ссылки на группы с исходниками проектов:

| Группа | Ссылка на проект                                             | Ссылка на PR с изменениями                          |
| ------ | ------------------------------------------------------------ | --------------------------------------------------- |
| ПД-412 | https://github.com/TOP-PD-412/PersonalAccount/tree/Lesson-10 | проект был перенесен                                |
| П-318  | https://github.com/TOP-P-318/PersonalAccount/tree/lesson-9   | https://github.com/TOP-P-318/PersonalAccount/pull/4 |
| П-319  | https://github.com/TOP-P-319/PersonalAccount/tree/lesson9-10 | https://github.com/TOP-P-319/PersonalAccount/pull/2 |

# Ход работы

## 1.1 Отображение данных студента на странице

У некоторых групп эта часть задания уже реализована в ходе работы в классе. Ее необходимо выполнить только тем, у кого она еще не готова.

### 1.1.1 Доработать `View` для отображения всех полей студента.

```HTML
@model StudentModel;
<!-- File: Views/Cabinet/Index.cshtml -->
<!-- Тут просто отображаете все поля студента, кроме его ID.
    Если обладаете чувством прекрасного, то можете стилизовать данную страницу.
    В проекте уже подключена библиотека bootstrap, поэтому моежет воспользоваться классами из нее -->
```

### 1.1.2 Доработать метод `Index` в контроллере `CabinetController`

```C#
// File: Controllers/CabinetController.cs
public class CabinetController(/*...*/, IStudentService students) : Controller
{
    // ...

    /**
    * Метод, возвращающий страницу представления студента
    */
    [HttpGet]
    public async Task<IActionResult> Index() { // метод асинхронных, так как выполняет запрос в БД
        // TODO: получаем id студента из Cookie
        // TODO: остальные данные достаем из БД
        var student = // обращение к StudentService students
        return View(student); // Тут мы можем сразу передавать объект student, так он является моделью для этого View
    }
}
```

### 1.1.3 Как вы заметили, в контроллере мы обращаемся к некоему `StudentService`, а значит, его надо реализовать

```C#
// File: Services/IStudentService.cs
public interface IStudentService
{
    Task<StudentModel?> GetByIdAsync(int id);
}
```

В соседнем файле реализуйте данный сервис. Зависеть он будет от `IStudentRepo<StudentModel>` (не `StudentAuthModel`)

### 1.1.4 В репозиторий `StudentRepo` необходимо джобавить новый метод

```C#
// File: Repositories/IStudentRepo.cs
public interface IStudentRepo<T> where T : StudentModel
{
    Task<T?> GetByIdAsync(int id);
}
```

### 1.1.5 Добавить все зависимости в `Program.cs`

У нас появились:

- Сервис `StudentService`
- Репозиторий `StudentRepo<StudentModel>` (Из-за различия в `Generic`-параметре, это полностью другой класс от `StudentRepo<StudentAuthModel>`)

## Требования

- Все поля StudentModel отображаются на сайте

## 2.1 Реализация формы для редакирования

### 2.1.1 Добавление отдельной `View` с полями для редактирования полей студента

Редактируемыми должны быть поля:

- `FullName` - имя студента
- `GroupName` - название группы
- `PhotoUrl` - ссылка на фото

[ ВАЖНО ] Поле `Email` редактироваться не должно.

### 2.1.2 Для работы с редактируемыми полями необходимо добавить соответсвующую `ViewModel` с валидацонными полями

```C#
// New file: Models/StudentEditViewModel.cs
public class StudentEditViewModel
{
    [Required(ErrorMessage = /*сообщение об ошибке*/)]
    public string FullName { get; set; } = string.Empty;
    [Required(ErrorMessage = /*сообщение об ошибке*/)]
    public string GroupName { get; set; } = string.Empty;
    public string? PhotoUrl { get; set; }
}
```

### 2.1.3 Вместе их будет связывать соответвующая пара методов `Edit` в `CabinetController`

```C#
// File: Controllers/CabinetController.cs
public class CabinetController : Controller
{
    // ...

    /**
    * Метод, возвращающий страницу формы для редактирования
    */
    [HttpGet]
    public async Task<IActionResult> Edit() {
        // TODO: получаем id студента из Cookie
        // TODO: остальные данные достаем из БД

        return View(new StudentEditViewModel{
            // TODO: заполеняем данными студента из БД
        });
    }


    /**
    * Метод, применяющий изменения на основе данных из формы
    */
    [HttpPost]
    [ValidateAntiforgeryToken]
    public async Task<IActionResult> Edit(StudentEditViewModel model) {
        // TODO: проверяем ModelState на наличие ошибок
        // TODO: обновляем данные в БД на основе model

       return RedirectToAction("Index");
    }

    // ...
}
```

### 2.1.4 Обновить `StudentService`

```C#
// File: Services/IStudentService.cs
public interface IStudentService
{
    Task UpdateByIdAsync(int id, StudentModel student);
}
```

### 2.1.5 В `StudentRepo` необходимо добавить метод для обновления записи в БД

```C#
// File: Repositories/IStudentRepo.cs
public interface IStudentRepo<T> where T : StudentModel
{
    Task UpdateByIdAsync(int id, StudentModel student);
}
```

### 2.1.4 Чтобы иметь возможность перейти в режим редактирвоания, необзодимо добавить кнопку на главную страницу кабинета

```HTML
<!-- File: Views/Cabinet/Index.cshtml -->

<form method="post" asp-controller="Cabinet" asp-action="Edit">
    <!-- Просто фрма с кнопкой редактироватью.
      Она через метод GET перебросит на страницу для редактирования,
      а та, в свою очередь будет иметь форму уже с методом POST,
      которая будет вызывать метод для применения результатов редактирования -->
    <button type="submit">Редактировать</button>
</form>
```

## Требования

- Страница редактирования переключается по нажатию на кнопку
- Поля `FullName`, `GroupName`, `PhotoUrl` редактируются
- Данные формы валидируются, и ошибки отображаются на экране
- Контроллер не пропускает форму с невалидными данными
- При валидных данных контроллер обновляет данные студента

## x.1 (Дополнительное задание) Добавить механизм смены пароля.

Идея следующая - в форме редактирования вы добавляете кнопку сменить пароль, после чего должна открыться страница редактирования пароля. Форма редактирования пароля требует ввода текущего пароля, ввода нового пароля, и подтверждения нового пароля.

### x.1.1 Добавить `ViewModel` для формы смены пароля

```C#
// New file: Models/PasswordChangeViewModel.cs
public class PasswordChangeViewModel
{
    [Required(ErrorMessage = /*сообщение об ошибке*/)]
    public string OldPassword { get; set; } = string.Empty;
    [Required(ErrorMessage = /*сообщение об ошибке*/)]
    public string NewPassword { get; set; } = string.Empty;
}
```

### x.1.2 Добавить `View` форму редактирования пароля

В ней будет 3 поля:

- Старый пароль
- Новый пароль
- Подтверждение нового пароля

[ВАЖНО] Проверка того. что новый пароль, и его подтверждение совпадают должна выполняться на клиенте через `JS` код
[ВАЖНО] Наджо проверить, что новый пароль не совпадает со старым

### x.1.3 В Контроллер `CabinetController` добавить новую пару методов `ChangePassword`

```C#
// File: Controllers/CabinetController.cs
public class CabinetController(/*...*/, IPasswordService password) : Controller
{
    // ...

    /**
    * Метод, возвращающий страницу формы для смены пароля
    */
    [HttpGet]
    public IActionResult ChangePassword() {
        return View(new PasswordChangeViewModel());
    }


    /**
    * Метод, применяющий изменения на основе данных из формы
    */
    [HttpPost]
    [ValidateAntiforgeryToken]
    public async Task<IActionResult> ChangePassword(PasswordChangeViewModel model) {
        // TODO: получаем id студента из Cookie
        // TODO: проверяем ModelState на наличие ошибок
        // TODO: допольнительно прямо тут проверяем, что model.OldPassword != model.NewPassword (юзер мог отправить запрос мимо формы, поэтому эту проверку надо продублировать)
        // TODO: сверяем хэш от model.OldPassword и student.PasswordHash в PasswordService

       return RedirectToAction("Index");
    }

    // ...
}
```

### x.1.4 Добавить новый сервис `PasswordService` для смены пароля

```C#
// File: Services/IPasswordService.cs
public interface IPasswordService
{
    Task<bool> ValidatePasswordAsync(int id, string password); // сюда передается OldPasword
    Task UpdatePasswordAsync(int id, string password); // а сюда NewPassword
}
```

Реализация `ValidatePasswordAsync` почти 1 в 1 повторит `StudentAuthService.ValidateStudentAsync`, только искать будет не по `email`, а по `id`

### x.1.5 В репозиторий придется добавить новый метод для `UpdatePasswordHashAsync`

```C#
// File: Repositories/IStudentRepo.cs
public interface IStudentRepo<T> where T : StudentModel
{
    Task UpdatePasswordHashAsync(int id, string passwordHash);
}
```

### x.1.6 Не забываем добавить зависимости в `Program.cs`

У нас появились:

- Сервис `PasswordService`

### x.1.7 Добавить новую кнопку в форму редактирования студента кнопку для перехода на страницу редактирования пароля

Необходимо учесть, что сейчас основная форма при любом `submit` отправить запрос на `[HttpPost] Edit()`.
Поэтому новая кнопка внутри исходной формы должна перехватывать управление на себя

```HTML
<button type="submit" formmethod="get" formaction="@Url.Action("ChangePassword", "Cabinet")">
    Сменить пароль
</button>
```

## Требования

- Внутри формы редактирования есть кнопка смены пароля
- При нажатии на кнопку открывается новая форма с 3 полями и кнопкой отправки
- Форма не позволяет ввести одинаковый старый и новый пароль, а также не позволяет, чтобы новый пароль и подтверждение пароля совпадали
- Контроллер валидирует, что старый пароль - это текущих пароль пользователя
- Контроллер дублирует проверку на то, что новый и старый пароли не совпадают
- Контрроллер созраняет хэш нового пароля в БД
