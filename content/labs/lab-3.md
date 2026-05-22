## 1. Тема, мета, посилання

### 1.1 Тема
«REST API. Реєстрація та авторизація. Валідація даних».

### 1.2 Мета
Реалізувати для сервісу `TaskFlow` механізми реєстрації та авторизації користувачів, валідацію вхідних даних, захист маршрутів через `JWT`, оновлення профілю та контроль доступу до робочих операцій над задачами.

### 1.3 Посилання
- Репозиторій власного проєкту (GitHub): буде додано після публікації.
- Розгорнутий застосунок: буде додано після розгортання.
- Репозиторій звітного HTML-документа (GitHub): буде додано після публікації.
- Звітний HTML-документ: буде додано після розгортання.

---

## 2. Хід виконання роботи

### 2.1 Логіка розвитку проєкту
На цьому етапі `TaskFlow` було переведено з рівня простої роботи з таблицями та задачами на рівень керування доступом до системи. Для сервісів, де різні користувачі створюють, переглядають і змінюють робочі записи, базовий CRUD без авторизації не є достатнім. Саме тому третя лабораторна робота зосереджена на тому, щоб пов'язати робочі дії з конкретним користувачем і захистити приватні маршрути.

Було реалізовано:
- реєстрацію нового учасника системи;
- вхід із перевіркою email і пароля;
- видачу `access token` та `refresh token`;
- перевірку прав доступу на окремих маршрутах;
- перегляд власного профілю;
- оновлення профілю;
- зміну пароля;
- вихід із системи;
- доступ до службових маршрутів через роль адміністратора.

У випадку `TaskFlow` ці механізми особливо важливі, оскільки надалі система повинна працювати не просто із записами, а з відповідальними виконавцями, статусами та контрольованими змінами задач.

### 2.2 Оновлення моделі користувача
Щоб реалізувати безпечну роботу з обліковими записами, модель `User` була розширена полями:
- `passwordHash`
- `refreshToken`

Поле `passwordHash` використовується для зберігання хешу пароля, а не його відкритого значення. Поле `refreshToken` дозволяє підтримувати механізм оновлення доступу без постійного повторного логіну.

У практичному сенсі це означає, що учасник робочого процесу після успішної авторизації може продовжувати взаємодіяти з API, а система при цьому контролює дійсність його сесії.

### 2.3 Реєстрація та перевірка коректності даних
На маршруті реєстрації було впроваджено перевірку даних ще до запису користувача в базу:
- перевірка наявності всіх обов'язкових полів;
- перевірка формату email;
- перевірка мінімальної довжини пароля;
- перевірка збігу `password` і `confirmPassword`;
- перевірка відсутності дубліката email.

Фрагмент коду реєстрації:

```js
async function register(req, res) {
  const { name, email, password, confirmPassword, role } = req.body;

  if (!name || !email || !password || !confirmPassword) {
    return res.status(400).json({ message: "All registration fields are required" });
  }

  if (!validateEmail(email)) {
    return res.status(400).json({ message: "Email format is invalid" });
  }

  if (password.length < 6) {
    return res.status(400).json({ message: "Password must be at least 6 characters long" });
  }

  if (password !== confirmPassword) {
    return res.status(400).json({ message: "Password confirmation does not match" });
  }
```

Після перевірок виконується хешування пароля та створення користувача:

```js
const passwordHash = await bcrypt.hash(password, 10);

const user = await User.create({
  name,
  email,
  role: role === "admin" ? "admin" : "user",
  passwordHash
});
```

Такий підхід дозволяє уникнути зберігання відкритих паролів і одразу формує базу для безпечнішої роботи системи.

### 2.4 Авторизація та токени доступу
Після входу користувач повинен отримати право виконувати дії в системі: переглядати свої дані, працювати з маршрутами та оновлювати робочі записи. Для цього в `TaskFlow` використано `JWT`.

Під час логіну:
- система знаходить користувача за email;
- звіряє пароль із хешем;
- створює `accessToken`;
- створює `refreshToken`;
- зберігає `refreshToken` у таблиці `users`.

Фрагмент перевірки логіну:

```js
const user = await User.findOne({ where: { email } });
if (!user) {
  return res.status(401).json({ message: "Invalid credentials" });
}

const passwordMatches = await bcrypt.compare(password, user.passwordHash);
if (!passwordMatches) {
  return res.status(401).json({ message: "Invalid credentials" });
}
```

Фрагмент створення токенів:

```js
function signAccessToken(user) {
  return jwt.sign(
    {
      id: user.id,
      email: user.email,
      role: user.role
    },
    process.env.JWT_SECRET,
    { expiresIn: "1h" }
  );
}
```

У `TaskFlow` це має практичне значення, тому що зміни у задачах, групах задач та службових маршрутах повинні виконувати лише авторизовані користувачі.

### 2.5 Захист маршрутів
Для приватних маршрутів було створено middleware, яке перевіряє наявність `Bearer`-токена в заголовку `Authorization`.

Фрагмент middleware:

```js
function authMiddleware(req, res, next) {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return res.status(401).json({ message: "Authorization token is missing" });
  }

  const token = authHeader.split(" ")[1];

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    req.user = payload;
    next();
  } catch (error) {
    return res.status(401).json({ message: "Invalid or expired token" });
  }
}
```

Після цього захищеними стали не лише профільні маршрути, а й операції створення або редагування робочих сутностей, наприклад задач і груп задач.

### 2.6 Робота з профілем та сесією
Окрім логіну та реєстрації, у сервісі реалізовано:
- `GET /api/auth/me`
- `PUT /api/auth/profile`
- `PUT /api/auth/change-password`
- `POST /api/auth/logout`

Маршрут `me` дозволяє отримати поточні дані користувача на основі токена. Маршрут `profile` дає змогу змінити ім'я або email. Маршрут `change-password` перевіряє поточний пароль і оновлює хеш. Вихід із системи очищає `refreshToken`, тим самим завершуючи сесію.

Фрагмент зміни пароля:

```js
const passwordMatches = await bcrypt.compare(currentPassword, user.passwordHash);
if (!passwordMatches) {
  return res.status(401).json({ message: "Current password is incorrect" });
}

const passwordHash = await bcrypt.hash(newPassword, 10);
await user.update({ passwordHash });
```

### 2.7 Поєднання авторизації з робочими маршрутами
Особливістю `TaskFlow` є те, що після реалізації авторизації API починає працювати не просто як набір технічних CRUD-методів, а як захищене середовище для керування процесом задач.

Після логіну адміністратор або авторизований користувач може:
- переглянути свій профіль;
- отримати список користувачів;
- створити нову групу задач;
- створити нову задачу;
- оновити статус або пріоритет задачі;
- видалити задачу.

Фрагмент створення задачі:

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

Фрагмент оновлення задачі:

```js
await task.update({
  title: title ?? task.title,
  description: description ?? task.description,
  status: status ?? task.status,
  priority: priority ?? task.priority,
  attachmentUrl: attachmentUrl ?? task.attachmentUrl,
  user_id: userId ?? task.user_id,
  task_group_id: taskGroupId ?? task.task_group_id
});
```

Таким чином, авторизація тут напряму пов'язана з робочою логікою, а не існує окремо від неї.

### 2.8 Підсумок реалізації
У результаті виконання лабораторної роботи `TaskFlow` отримав:
- базовий механізм автентифікації;
- `JWT`-авторизацію;
- валідацію даних під час реєстрації;
- підтримку `refresh token`;
- захист профільних і робочих маршрутів;
- можливість безпечної роботи з задачами після входу до системи.

Це означає, що далі сервіс уже можна розвивати як багатокористувацьку систему, де записи мають не лише структуру, а й контрольований доступ.

---

## 3. Скріншоти результату

![POST logout](/assets/labs/lab-3/screen-12.png)

![PUT change password](/assets/labs/lab-3/screen-11.png)

![PUT update profile](/assets/labs/lab-3/screen-10.png)

![DELETE task](/assets/labs/lab-3/screen-9.png)

![PUT task](/assets/labs/lab-3/screen-8.png)

![POST task](/assets/labs/lab-3/screen-7.png)

![POST task group](/assets/labs/lab-3/screen-6.png)

![GET users](/assets/labs/lab-3/screen-5.png)

![POST refresh](/assets/labs/lab-3/screen-4.png)

![GET me](/assets/labs/lab-3/screen-3.png)

![POST login](/assets/labs/lab-3/screen-2.png)

![POST register](/assets/labs/lab-3/screen-1.png)

---

## 4. Висновки

У межах третьої лабораторної роботи для сервісу `TaskFlow` було реалізовано реєстрацію, авторизацію через `JWT`, перевірку вхідних даних, захист маршрутів і керування сесією користувача. Додатково було інтегровано ці механізми з робочими маршрутами системи задач, що дозволило перейти від звичайного CRUD-рівня до контрольованої багатокористувацької взаємодії. Отриманий результат формує основу для подальшого розвитку сервісу в напрямку безпеки, ролей і розширеної бізнес-логіки.
