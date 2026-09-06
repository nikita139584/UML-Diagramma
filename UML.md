# План архітектури для проєкту «Кулінарний блог»

Архітектура будується як **ASP.NET Core MVC** монолітний веб-застосунок з розділенням відповідальності між моделями, бізнес-логікою, доступом до даних та представленнями. Такий підхід відповідає SRS і не створює зайвого overengineering для поточного обсягу проєкту. У системі передбачені реєстрація, авторизація, рецепти, блоги, пости, коментарі, оцінювання та адміністративне керування.

## Зміст

1. [Загальна архітектура Backend](#1-загальна-архітектура-backend)
2. [Структура бази даних SQL Server](#2-структура-бази-даних-sql-server)
3. [Архітектура Frontend](#3-архітектура-frontend)
4. [Взаємодія компонентів та потоки даних](#4-взаємодія-компонентів-та-потоки-даних)
5. [Авторизація та безпека](#5-авторизація-та-безпека)
6. [Розподіл відповідальності компонентів](#6-розподіл-відповідальності-компонентів)
7. [Підсумкова схема архітектури](#7-підсумкова-схема-архітектури)

---

## 1. Загальна архітектура Backend

Використовується ASP.NET Core MVC з класичним розділенням на `Models`, `Data`, `Services`, `Controllers` і `Views`.

```
CookingBlog.sln
│
└── CookingBlog/
    │
    ├── Controllers/
    │   ├── AccountController.cs        # Реєстрація та авторизація
    │   ├── RecipesController.cs        # Перегляд, пошук, фільтрація рецептів
    │   ├── BlogsController.cs          # Створення та редагування блогів
    │   ├── PostsController.cs          # CRUD постів
    │   ├── CommentsController.cs       # Додавання та видалення коментарів
    │   ├── RatingsController.cs        # Оцінювання рецептів
    │   └── AdminController.cs          # Адміністративне керування
    │
    ├── Models/
    │   ├── Entities/                   # Сутності БД
    │   │   ├── ApplicationUser.cs
    │   │   ├── Recipe.cs
    │   │   ├── Blog.cs
    │   │   ├── Post.cs
    │   │   ├── Comment.cs
    │   │   └── Rating.cs
    │   │
    │   ├── ViewModels/                 # Моделі для Views
    │   │   ├── LoginViewModel.cs
    │   │   ├── RegisterViewModel.cs
    │   │   ├── RecipeListViewModel.cs
    │   │   ├── RecipeDetailsViewModel.cs
    │   │   ├── CreatePostViewModel.cs
    │   │   └── AdminUserViewModel.cs
    │   │
    │   └── Enums/
    │       └── UserRole.cs
    │
    ├── Data/
    │   ├── ApplicationDbContext.cs      # EF Core DbContext
    │   ├── Configurations/              # Fluent API конфігурації
    │   └── Migrations/                  # EF Core migrations
    │
    ├── Services/
    │   ├── IRecipeService.cs
    │   ├── RecipeService.cs
    │   ├── IBlogService.cs
    │   ├── BlogService.cs
    │   ├── IPostService.cs
    │   ├── PostService.cs
    │   ├── ICommentService.cs
    │   ├── CommentService.cs
    │   ├── IRatingService.cs
    │   └── RatingService.cs
    │
    ├── Views/
    │   ├── Shared/
    │   │   ├── _Layout.cshtml
    │   │   ├── _ValidationScriptsPartial.cshtml
    │   │   └── _Error.cshtml
    │   │
    │   ├── Account/
    │   │   ├── Login.cshtml
    │   │   └── Register.cshtml
    │   │
    │   ├── Recipes/
    │   │   ├── Index.cshtml
    │   │   ├── Details.cshtml
    │   │   └── Popular.cshtml
    │   │
    │   ├── Blogs/
    │   │   ├── Index.cshtml
    │   │   ├── Create.cshtml
    │   │   └── Edit.cshtml
    │   │
    │   ├── Posts/
    │   │   ├── Create.cshtml
    │   │   ├── Edit.cshtml
    │   │   └── Details.cshtml
    │   │
    │   └── Admin/
    │       ├── Users.cshtml
    │       └── Posts.cshtml
    │
    ├── wwwroot/
    │   ├── css/
    │   ├── js/
    │   ├── images/
    │   └── lib/
    │
    ├── Areas/
    │   └── Identity/                   # ASP.NET Core Identity UI
    │
    ├── appsettings.json
    ├── Program.cs
    └── CookingBlog.csproj
```

### Основний принцип взаємодії

```
Browser
   ↓
Controller
   ↓
Service
   ↓
Entity Framework Core
   ↓
SQL Server
   ↓
Database
```

Для відображення:

```
Controller
   ↓
ViewModel
   ↓
Razor View (.cshtml)
   ↓
HTML + Bootstrap + JavaScript
   ↓
Browser
```

Такий варіант добре відповідає заданому в SRS MVC-підходу.

---

## 2. Структура бази даних SQL Server

База даних реалізується через **Entity Framework Core Code First**. Дані зберігаються в SQL Server, а конфігурація зв'язків виконується через Fluent API.

Основні сутності безпосередньо випливають із функціональних вимог до рецептів, блогів, постів, коментарів, оцінювання та користувачів.

### Users

Для користувачів використовується ASP.NET Core Identity.

```
AspNetUsers
- Id [PK]
- UserName
- Email
- PasswordHash
- FirstName
- LastName
- IsBlocked
```

Identity також забезпечує необхідні механізми автентифікації та зберігання паролів у захищеному вигляді, що відповідає вимогам безпеки SRS.

### Roles

```
AspNetRoles
- Id [PK]
- Name
```

Основні ролі:

- Guest
- User
- Author
- Administrator

Розподіл ролей відповідає опису користувачів у SRS.

### Recipes

```
Recipes
- Id [PK]
- Title
- Description
- Ingredients
- Instructions
- ImageUrl
- CreatedAt
- AuthorId [FK]
```

### Blogs

```
Blogs
- Id [PK]
- Name
- Description
- CreatedAt
- AuthorId [FK]
```

### Posts

```
Posts
- Id [PK]
- BlogId [FK]
- Title
- Content
- CreatedAt
- UpdatedAt
```

### Comments

```
Comments
- Id [PK]
- RecipeId [FK]
- UserId [FK]
- Content
- CreatedAt
```

### Ratings

```
Ratings
- Id [PK]
- RecipeId [FK]
- UserId [FK]
- Value
- CreatedAt
```

Бажано встановити унікальне обмеження:

```sql
UNIQUE (RecipeId, UserId)
```

щоб один користувач не створював кілька оцінок для одного рецепту.

### Загальна структура зв'язків

```
User
 ├── 1:N → Blogs
 ├── 1:N → Posts
 ├── 1:N → Comments
 └── 1:N → Ratings

Blog
 └── 1:N → Posts

Recipe
 ├── 1:N → Comments
 └── 1:N → Ratings
```

---

## 3. Архітектура Frontend

Оскільки в SRS не передбачений React або інший SPA-фреймворк, клієнтська частина реалізується безпосередньо через Razor Views + HTML + CSS + JavaScript + Bootstrap.

```
wwwroot/
│
├── css/
│   ├── site.css
│   ├── recipes.css
│   ├── blogs.css
│   └── admin.css
│
├── js/
│   ├── site.js
│   ├── recipes.js
│   ├── comments.js
│   └── ratings.js
│
├── images/
│   ├── recipes/
│   ├── posts/
│   └── avatars/
│
└── lib/
    └── bootstrap/
```

### Razor Views

```
Views/
│
├── Account/
│   ├── Login.cshtml
│   └── Register.cshtml
│
├── Recipes/
│   ├── Index.cshtml
│   ├── Details.cshtml
│   └── Popular.cshtml
│
├── Blogs/
│   ├── Index.cshtml
│   ├── Create.cshtml
│   └── Edit.cshtml
│
├── Posts/
│   ├── Create.cshtml
│   ├── Edit.cshtml
│   └── Details.cshtml
│
└── Admin/
    ├── Users.cshtml
    ├── Posts.cshtml
    └── Comments.cshtml
```

### Основні сторінки користувача

```
Home
   ↓
Recipe List
   ├── Search
   ├── Filter
   └── Popular Recipes

Recipe Details
   ├── Rating
   └── Comments

Profile
   ├── My Blogs
   └── My Posts

Admin
   ├── Users
   ├── Posts
   └── Comments
```

Це покриває заявлені в SRS перегляд, пошук і фільтрацію рецептів, роботу з блогами та постами, коментарі, оцінювання й адміністративне керування.

---

## 4. Взаємодія компонентів та потоки даних

### 4.1. Перегляд списку рецептів

```
Browser
   ↓
GET /Recipes
   ↓
RecipesController
   ↓
RecipeService
   ↓
ApplicationDbContext
   ↓
SQL Server
   ↓
Recipe entities
   ↓
RecipeListViewModel
   ↓
Recipes/Index.cshtml
   ↓
HTML + Bootstrap
```

Controller не повинен містити складну бізнес-логіку. Його завдання — отримати параметри запиту, викликати сервіс та передати результат у View.

### 4.2. Пошук і фільтрація

```
User
 ↓
Search / Filter form
 ↓
RecipesController
 ↓
RecipeService
 ↓
LINQ query
 ↓
SQL Server
 ↓
Filtered recipes
 ↓
RecipeListViewModel
 ↓
Index.cshtml
```

Наприклад:

```
/Recipes?search=паста&category=...
```

SRS прямо визначає пошук як обов'язкову функцію, а фільтрацію — як Should.

### 4.3. Створення посту

```
Author
   ↓
Create Post Form
   ↓
PostsController
   ↓
Model Validation
   ↓
PostService
   ↓
ApplicationDbContext
   ↓
SQL Server
   ↓
Post created
   ↓
Redirect → Post Details
```

Цей потік відповідає **UC-02**: відкриття форми → заповнення → валідація → збереження посту.

### 4.4. Додавання коментаря

```
Authenticated User
   ↓
Comment Form
   ↓
CommentsController
   ↓
Validation
   ↓
CommentService
   ↓
EF Core
   ↓
SQL Server
   ↓
Comment saved
   ↓
Recipe Details
```

Це відповідає **UC-03**, де авторизований користувач вводить коментар, після чого система перевіряє дані та зберігає запис.

### 4.5. Оцінювання рецепту

```
User
 ↓
Select rating
 ↓
RatingsController
 ↓
RatingService
 ↓
Check existing rating
 ↓
Save / Update Rating
 ↓
Calculate average
 ↓
Recipe Details
```

Таким чином виконується вимога зберігати оцінку користувача та оновлювати рейтинг рецепту.

### 4.6. Адміністративне керування

```
Administrator
     ↓
AdminController
     ↓
[Authorize(Roles = "Administrator")]
     ↓
AdminService / Identity
     ↓
SQL Server
     ↓
Users / Posts / Comments
```

Адміністративні операції повинні бути закриті атрибутом авторизації, оскільки SRS прямо вимагає доступ до адміністративних сторінок лише для адміністраторів.

---

## 5. Авторизація та безпека

Для авторизації використовується **ASP.NET Core Identity**, що безпосередньо відповідає SRS.

Схема:

```
Registration
     ↓
ASP.NET Core Identity
     ↓
Password Hashing
     ↓
AspNetUsers
```

Після входу:

```
Login
 ↓
Identity
 ↓
Authenticated User
 ↓
Claims / Role
 ↓
[Authorize]
 ↓
Access to protected functionality
```

Для адміністратора:

```csharp
[Authorize(Roles = "Administrator")]
```

Для функцій авторизованого користувача:

```csharp
[Authorize]
```

Також застосовуються:

- Client-side validation
- Server-side validation
- Anti-forgery protection
- Authorization

Це покриває вимоги **NFR-03–NFR-07**.

---

## 6. Розподіл відповідальності компонентів

Щоб архітектура не перетворилася на один великий Controller, відповідальність можна розподілити так:

```
Controller
│
├── приймає HTTP-запит
├── перевіряє авторизацію
├── викликає Service
└── повертає View / Redirect

Service
│
├── бізнес-логіка
├── перевірки
├── робота з Entity
└── підготовка даних

DbContext
│
├── доступ до БД
├── LINQ
└── SaveChangesAsync()

ViewModel
│
└── дані для конкретної сторінки

View
│
├── HTML
├── Bootstrap
└── JavaScript
```

Тобто `Controller → Service → EF Core → SQL Server`, а не Controller, який безпосередньо містить всю логіку.

---

## 7. Підсумкова схема архітектури

```
                    ┌──────────────────────┐
                    │       Browser         │
                    │ HTML/CSS/JS/Bootstrap │
                    └──────────┬────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   ASP.NET Core MVC    │
                    ├──────────────────────┤
                    │ Controllers           │
                    │ Views / Razor         │
                    │ ViewModels            │
                    └──────────┬────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │       Services        │
                    ├──────────────────────┤
                    │ RecipeService          │
                    │ BlogService            │
                    │ PostService            │
                    │ CommentService         │
                    │ RatingService          │
                    └──────────┬────────────┘
                               │
                ┌──────────────┴──────────────┐
                ▼                             ▼
       ┌─────────────────┐          ┌──────────────────┐
       │ ASP.NET Identity │          │   EF Core         │
       │ Authentication   │          │   DbContext       │
       │ Authorization    │          │   LINQ            │
       └────────┬─────────┘          └────────┬──────────┘
                │                             │
                └──────────────┬──────────────┘
                               ▼
                    ┌──────────────────────┐
                    │      SQL Server       │
                    ├──────────────────────┤
                    │ Users / Roles          │
                    │ Recipes                │
                    │ Blogs                  │
                    │ Posts                  │
                    │ Comments               │
                    │ Ratings                │
                    └──────────────────────┘
```
