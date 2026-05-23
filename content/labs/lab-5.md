## 1. Тема, мета, посилання

### 1.1 Тема
«Безпека та продуктивність серверних додатків. Безпека Node.js-додатків. Оптимізація запитів і кешування. Тестування API».

### 1.2 Мета
Підвищити безпеку та продуктивність сервісу `TaskFlow`, реалізувати захист маршрутів, додати валідацію даних, запровадити кешування відповідей і виконати автоматизовану перевірку частини API.

### 1.3 Посилання
- Репозиторій власного проєкту (GitHub): [Ilyasix/TaskFlow](https://github.com/Ilyasix/TaskFlow)
- Розгорнутий застосунок: локальний запуск із репозиторію проєкту.
- Репозиторій звітного HTML-документа (GitHub): [Ilyasix/IO-31_appRECORD-IsmailovIlias-FIOT-2026](https://github.com/Ilyasix/IO-31_appRECORD-IsmailovIlias-FIOT-2026)
- Звітний HTML-документ: [ilyasix.github.io/IO-31_appRECORD-IsmailovIlias-FIOT-2026](https://ilyasix.github.io/IO-31_appRECORD-IsmailovIlias-FIOT-2026/)

---

## 2. Хід виконання роботи

### 2.1 Загальна ідея роботи
На етапі п'ятої лабораторної роботи `TaskFlow` було розвинуто в напрямку безпеки та продуктивності. Якщо на попередніх етапах система вже підтримувала авторизацію, логування, завантаження вкладень і маршрути роботи з задачами, то далі постало завдання зробити її більш стійкою до некоректних запитів і більш ефективною під час повторного використання ресурсів.

У межах роботи було реалізовано:
- захисні HTTP-заголовки;
- обмеження кількості запитів;
- валідацію даних для ключових маршрутів;
- стиснення відповідей;
- кешування списку задач через `Redis`;
- автоматизовані тести через `Jest` і `Supertest`.

### 2.2 Захисний шар для API
Для підвищення базової безпеки сервера було використано `Helmet`. Це дозволяє автоматично додавати захисні HTTP-заголовки, які зменшують частину ризиків, пов'язаних із небажаним вбудовуванням сторінок, MIME-sniffing і деякими браузерними атаками.

Разом із цим було додано `express-rate-limit`, який обмежує кількість запитів від одного клієнта.

Фрагмент конфігурації:

```js
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 40,
  standardHeaders: true,
  legacyHeaders: false,
  message: {
    message: "Too many requests, please try again later"
  }
});
```

У результаті сервер:
- повертає `RateLimit`-заголовки;
- обмежує надмірну активність;
- не дозволяє нескінченно навантажувати маршрути за короткий час.

### 2.3 Стиснення та middleware-послідовність
Щоб відповіді займали менше трафіку, було підключено `compression`. Це особливо корисно для JSON-відповідей списків задач, оскільки вони можуть містити пов'язані сутності та додаткові поля.

Фрагмент підключення:

```js
app.use(helmetMiddleware);
app.use(compressionMiddleware);
app.use(limiter);
app.use(requestLogger);
app.use(performanceMiddleware);
```

Тобто безпека, оптимізація та технічний моніторинг були винесені в загальний стек middleware ще до підключення основних маршрутів.

### 2.4 Валідація вхідних даних
Щоб не допустити створення задач із неповними або некоректними полями, у проєкті було впроваджено перевірки через `express-validator`.

Для логіну:

```js
const loginValidator = [
  body("email").isEmail().withMessage("email must be valid"),
  body("password").notEmpty().withMessage("password is required")
];
```

Для створення задачі:

```js
const createTaskValidator = [
  body("title").trim().isLength({ min: 3 }).withMessage("title must be at least 3 characters long"),
  body("status").trim().notEmpty().withMessage("status is required"),
  body("priority").trim().notEmpty().withMessage("priority is required"),
  body("userId").isInt({ min: 1 }).withMessage("userId must be a positive integer"),
  body("taskGroupId").isInt({ min: 1 }).withMessage("taskGroupId must be a positive integer")
];
```

Після цього було використано окреме middleware для формування єдиної відповіді при помилках:

```js
function validationMiddleware(req, res, next) {
  const errors = validationResult(req);

  if (errors.isEmpty()) {
    return next();
  }

  return res.status(400).json({
    message: "Validation failed",
    errors: errors.array()
  });
}
```

Завдяки цьому сервер не приймає некоректні задачі або логіни й повертає структурований формат помилок.

### 2.5 Кешування списку задач через Redis
Щоб зменшити навантаження на базу даних, було реалізовано кешування списку задач через `Redis`. Для цього створено окремий модуль роботи з кешем і підключено Redis-контейнер через `docker-compose`.

Фрагмент створення клієнта:

```js
client = createClient({
  url: process.env.REDIS_URL || "redis://127.0.0.1:6379"
});
```

Логіка маршруту `GET /api/tasks` працює так:
- якщо значення вже є в Redis, воно повертається одразу;
- якщо значення в кеші немає, сервер отримує дані з БД, записує їх у Redis та повертає клієнту.

Фрагмент:

```js
const cachedTasks = await getCache(TASKS_CACHE_KEY);

if (cachedTasks) {
  return res.json({
    source: "cache",
    data: cachedTasks
  });
}
```

Після отримання з БД:

```js
await setCache(TASKS_CACHE_KEY, tasks, 120);
res.json({
  source: "database",
  data: tasks
});
```

Це дозволяє чітко побачити різницю між першим і повторним запитом.

### 2.6 Очищення кешу після змін
Для системи задач особливо важливо, щоб користувачі не бачили застарілий стан записів. Саме тому після створення, оновлення чи видалення задачі кеш очищається.

Фрагмент:

```js
await deleteCache(TASKS_CACHE_KEY);
```

Аналогічно кеш очищується після створення нової групи задач, оскільки це також впливає на структуру робочих даних.

### 2.7 Автоматизоване тестування
Щоб перевірити, що базові маршрути не зламалися після додавання безпеки, кешування та валідації, було реалізовано тести через `Jest` і `Supertest`.

Фрагмент тестів:

```js
test("GET /api/health should return 200", async () => {
  const response = await request(app).get("/api/health");

  expect(response.statusCode).toBe(200);
  expect(response.body.status).toBe("ok");
});
```

Також перевіряється маршрут продуктивності:

```js
test("GET /api/status should return performance payload", async () => {
  const response = await request(app).get("/api/status");

  expect(response.statusCode).toBe(200);
  expect(response.body.app).toBe("TaskFlow");
  expect(response.body.memoryUsage).toBeDefined();
});
```

І окремо перевіряється валідація логіну:

```js
test("POST /api/auth/login should validate email", async () => {
  const response = await request(app)
    .post("/api/auth/login")
    .send({
      email: "wrong-format",
      password: "123456"
    });

  expect(response.statusCode).toBe(400);
  expect(response.body.message).toBe("Validation failed");
});
```

### 2.8 Практичний результат
У підсумку `TaskFlow` отримав:
- захист заголовків через `Helmet`;
- обмеження запитів через `rate limit`;
- валідацію даних для логіну, груп задач і задач;
- стиснення відповідей;
- кешування списку задач через `Redis`;
- очищення кешу після змін;
- автоматизовану перевірку основних маршрутів.

Це робить систему не лише функціональною, а й більш стійкою до некоректного використання, повторних звернень і зайвого навантаження.

---

## 3. Скріншоти результату

### 3.1 Запуск автоматизованих тестів
![Запуск тестів](/assets/labs/lab-5/screen-1.png)

### 3.2 Підняті сервіси MySQL і Redis
![Docker compose services](/assets/labs/lab-5/screen-2.png)

### 3.3 Перевірка маршруту health
![API health](/assets/labs/lab-5/screen-3.png)

### 3.4 Перевірка маршруту status
![API status](/assets/labs/lab-5/screen-4.png)

### 3.5 Успішна авторизація
![Login success](/assets/labs/lab-5/screen-5.png)

### 3.6 Валідація помилки логіну
![Login validation error](/assets/labs/lab-5/screen-6.png)

### 3.7 Перший запит до списку задач
![Tasks from database](/assets/labs/lab-5/screen-7.png)

### 3.8 Повторний запит до списку задач
![Tasks from cache](/assets/labs/lab-5/screen-8.png)

### 3.9 Валідація при створенні задачі
![Task validation error](/assets/labs/lab-5/screen-9.png)

---

## 4. Висновки

У межах п'ятої лабораторної роботи для `TaskFlow` було реалізовано базовий захист API, перевірку вхідних даних, оптимізацію відповідей і кешування через `Redis`. Додатково було впроваджено автоматизовані тести, що дозволило перевірити працездатність критичних маршрутів після розширення серверної логіки. Отриманий результат є основою для фінального етапу курсу, де вже можна перейти до документування API та підготовки проєкту до публічного розгортання.
