## 1. Тема, мета, посилання

### 1.1 Тема
«Створення бази даних у MySQL. Підключення Node.js до MySQL. Робота з ORM Sequelize».

### 1.2 Мета
Створити базу даних для сервісу `TaskFlow`, налаштувати взаємодію `Node.js` із `MySQL`, описати сутності через `Sequelize` та реалізувати практичну роботу із задачами, користувачами й групами задач.

### 1.3 Посилання
- Репозиторій власного проєкту (GitHub): буде додано після публікації.
- Розгорнутий застосунок: буде додано після розгортання.
- Репозиторій звітного HTML-документа (GitHub): буде додано після публікації.
- Звітний HTML-документ: буде додано після розгортання.

---

## 2. Хід виконання роботи

### 2.1 Проєктування структури бази даних
Для сервісу `TaskFlow` було створено окрему базу даних `taskflow_db`. У її межах реалізовано таблиці:
- `users`
- `task_groups`
- `tasks`

Основною таблицею є `tasks`, оскільки саме вона описує робочі записи системи. Кожна задача має назву, опис, статус, пріоритет, посилання на вкладення та зовнішні ключі на користувача й групу задач. Такий підхід дозволяє не лише зберігати окремі елементи, а й будувати зрозумілий робочий процес, у якому видно, хто є виконавцем і до якого напряму належить задача.

Таблиця `users` використовується для зберігання виконавців. Таблиця `task_groups` дає змогу групувати задачі за напрямами, наприклад за підтримкою, плануванням або окремими процесами. Завдяки цьому структура даних є більш наближеною до реального сценарію використання.

### 2.2 Налаштування підключення
Підключення застосунку до `MySQL` виконано через `Sequelize`, а конфігурацію винесено у змінні середовища. Це дозволяє запускати застосунок на різних машинах або портах без змін у коді контролерів.

Фрагмент конфігурації:

```js
const { Sequelize } = require("sequelize");
require("dotenv").config();

const sequelize = new Sequelize(
  process.env.DB_NAME,
  process.env.DB_USER,
  process.env.DB_PASSWORD,
  {
    host: process.env.DB_HOST,
    port: Number(process.env.DB_PORT || 3306),
    dialect: "mysql",
    logging: false
  }
);
```

У цьому фрагменті:
- використовується окремий екземпляр `Sequelize`;
- параметри підключення читаються із `.env`;
- вимкнено службовий лог SQL-запитів;
- порт бази даних може змінюватися незалежно від решти застосунку.

### 2.3 Моделі та зв'язки між сутностями
У лабораторній роботі було створено три моделі:
- `User`
- `TaskGroup`
- `Task`

Між ними налаштовано зв'язки:
- `User.hasMany(Task)`
- `Task.belongsTo(User)`
- `TaskGroup.hasMany(Task)`
- `Task.belongsTo(TaskGroup)`

Фрагмент опису асоціацій:

```js
User.hasMany(Task, {
  foreignKey: "user_id",
  as: "tasks"
});
Task.belongsTo(User, {
  foreignKey: "user_id",
  as: "assignee"
});

TaskGroup.hasMany(Task, {
  foreignKey: "task_group_id",
  as: "tasks"
});
Task.belongsTo(TaskGroup, {
  foreignKey: "task_group_id",
  as: "group"
});
```

Після такого налаштування сервер може повертати задачу разом із пов'язаними даними про виконавця та групу. Це особливо корисно в системах керування задачами, де запис без контексту майже не має практичної цінності.

### 2.4 Реалізація операцій над задачами
У межах роботи було реалізовано базові CRUD-операції:
- створення задач;
- перегляд списку задач;
- редагування існуючих записів;
- видалення задач.

Для прикладного сервісу це є мінімально необхідним функціоналом, адже робота із задачами постійно передбачає оновлення статусів, зміну виконавця або пріоритету.

Фрагмент контролера створення задачі:

```js
async function createTask(req, res) {
  const { title, description, status, priority, attachmentUrl, userId, taskGroupId } = req.body;

  if (!title || !status || !priority || !userId || !taskGroupId) {
    return res.status(400).json({
      message: "title, status, priority, userId and taskGroupId are required"
    });
  }

  const task = await Task.create({
    title,
    description,
    status,
    priority,
    attachmentUrl,
    user_id: userId,
    task_group_id: taskGroupId
  });
```

У цьому блоці:
- перевіряються обов'язкові поля;
- виконується створення нового запису через модель `Task`;
- зберігаються зовнішні ключі на користувача та групу задач;
- у подальшому створений запис повертається вже разом із пов'язаними сутностями.

### 2.5 Отримання та оновлення даних
Окрему увагу було приділено тому, щоб система повертала задачі не ізольовано, а разом із контекстом. Для цього у вибірці використано `include`, що дає змогу одразу отримувати пов'язані моделі.

Фрагмент отримання списку задач:

```js
async function getTasks(req, res) {
  const tasks = await Task.findAll({
    include: [
      { model: User, as: "assignee", attributes: ["id", "name", "email", "role"] },
      { model: TaskGroup, as: "group", attributes: ["id", "name", "description"] }
    ],
    order: [["id", "ASC"]]
  });

  res.json(tasks);
}
```

Також було реалізовано оновлення вже існуючої задачі:

```js
async function updateTask(req, res) {
  const task = await Task.findByPk(req.params.id);
  if (!task) {
    return res.status(404).json({ message: "task not found" });
  }

  const { title, description, status, priority, attachmentUrl, userId, taskGroupId } = req.body;

  await task.update({
    title: title ?? task.title,
    description: description ?? task.description,
    status: status ?? task.status,
    priority: priority ?? task.priority,
    attachmentUrl: attachmentUrl ?? task.attachmentUrl,
    user_id: userId ?? task.user_id,
    task_group_id: taskGroupId ?? task.task_group_id
  });
}
```

Це дає змогу частково змінювати задачу, не дублюючи весь об'єкт у запиті. Такий варіант є зручним у випадках, коли потрібно змінити лише статус, пріоритет або виконавця.

### 2.6 Робота через SQL та API
У роботі використовувалися два підходи:
- прямі SQL-запити через `mysql2`;
- прикладний рівень через `Sequelize` та `Express`.

Через SQL-скрипт було продемонстровано команди `INSERT`, `SELECT`, `UPDATE`, `DELETE`. Через HTTP-маршрути було перевірено роботу з користувачами, групами задач та самими задачами.

Основні маршрути, які використовувалися під час перевірки:
- `GET /api/health`
- `GET /api/users`, `POST /api/users`
- `GET /api/task-groups`, `POST /api/task-groups`
- `GET /api/tasks`, `POST /api/tasks`, `PUT /api/tasks/:id`, `DELETE /api/tasks/:id`

### 2.7 Підсумок виконання
У результаті виконання лабораторної роботи було отримано функціональну серверну частину сервісу `TaskFlow`, яка вже підтримує пов'язану структуру даних, демонстрацію SQL-операцій, роботу з ORM-моделями та базові API-маршрути. Сформована база є зручною для подальшого переходу до авторизації, валідації, ролей користувачів та розширення бізнес-логіки.

---

## 3. Скріншоти результату

![PUT task](/assets/labs/lab-2/screen-11.png)

![POST task](/assets/labs/lab-2/screen-10.png)

![POST task group](/assets/labs/lab-2/screen-9.png)

![POST user](/assets/labs/lab-2/screen-8.png)

![GET tasks](/assets/labs/lab-2/screen-7.png)

![GET task groups](/assets/labs/lab-2/screen-6.png)

![GET users](/assets/labs/lab-2/screen-5.png)

![API health](/assets/labs/lab-2/screen-4.png)

![Raw SQL demo](/assets/labs/lab-2/screen-3.png)

![Seed successful](/assets/labs/lab-2/screen-2.png)

![MySQL container running](/assets/labs/lab-2/screen-1.png)

---

## 4. Висновки

Під час виконання лабораторної роботи було створено базу даних `MySQL` для сервісу `TaskFlow`, реалізовано моделі `User`, `TaskGroup` і `Task`, налаштовано зв'язки між ними та продемонстровано роботу з даними через SQL і через `Sequelize`. Отримана структура дозволяє працювати із задачами як із повноцінними об'єктами процесу, де важливими є виконавець, група задач, статус і пріоритет. Це створює надійну основу для наступних етапів реалізації серверної логіки.
