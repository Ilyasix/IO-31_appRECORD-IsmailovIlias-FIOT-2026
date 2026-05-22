## 1. Тема, мета, посилання

### 1.1 Тема
«Розширені можливості Node.js-додатків: логування, завантаження файлів, моніторинг продуктивності».

### 1.2 Мета
Розширити сервер `TaskFlow` засобами логування, додати завантаження вкладень до робочих процесів та реалізувати базовий моніторинг стану застосунку.

### 1.3 Посилання
- Репозиторій власного проєкту (GitHub): буде додано після публікації.
- Розгорнутий застосунок: буде додано після розгортання.
- Репозиторій звітного HTML-документа (GitHub): буде додано після публікації.
- Звітний HTML-документ: буде додано після розгортання.

---

## 2. Хід виконання роботи

### 2.1 Розширення прикладного сервісу
На етапі четвертої лабораторної роботи `TaskFlow` було доповнено можливостями, які важливі для практичного серверного застосунку:
- фіксація HTTP-запитів і внутрішніх подій;
- завантаження вкладень для задач;
- контроль швидкодії та стану процесу.

Для системи керування задачами це природне продовження попередніх етапів. Якщо користувачі вже можуть створювати задачі та змінювати їхній стан, то далі виникає потреба:
- прикладати файли до робочих записів;
- відстежувати звернення до API;
- аналізувати навантаження на сервер.

### 2.2 Логування запитів і подій
Для базового логування HTTP-запитів було використано `Morgan`, а для більш структурованого запису подій і помилок — `Winston`.

Було створено окремий logger:

```js
const logger = winston.createLogger({
  level: "info",
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({
      filename: path.join(logDirectory, "error.log"),
      level: "error"
    }),
    new winston.transports.File({
      filename: path.join(logDirectory, "combined.log")
    })
  ]
});
```

У результаті застосунок отримав:
- `combined.log` для загального журналу подій;
- `error.log` для збереження лише помилок.

Після цього HTTP-запити були підключені до логера через `Morgan`:

```js
const requestLogger = morgan("combined", {
  stream: {
    write(message) {
      logger.info(message.trim());
    }
  }
});
```

Завдяки такій зв'язці в логах фіксуються звернення до маршрутів, а інформація зберігається не лише у консолі, а й у файлах.

### 2.3 Вимірювання часу виконання
Щоб бачити, як сервер поводиться під навантаженням, було додано middleware для замірювання тривалості кожного запиту.

Фрагмент middleware:

```js
function performanceMiddleware(req, res, next) {
  const startTime = process.hrtime.bigint();

  res.on("finish", () => {
    const endTime = process.hrtime.bigint();
    const durationMs = Number(endTime - startTime) / 1_000_000;

    logger.info("request_completed", {
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      durationMs: Number(durationMs.toFixed(2))
    });
  });

  next();
}
```

Такий підхід дає змогу:
- фіксувати швидкість відповіді;
- бачити потенційно повільні маршрути;
- отримувати базовий журнал продуктивності без додаткових сервісів.

Для `TaskFlow` це особливо корисно, тому що в системі надалі можуть з'явитися складніші операції із задачами, файлами та ролями доступу.

### 2.4 Завантаження вкладень до задач
У `TaskFlow` можливість додавати вкладення логічно пов'язана з робочими задачами. Для цього було використано `Multer`, а самі файли зберігаються в директорії:
- `uploads/attachments`

Фрагмент конфігурації збереження:

```js
const storage = multer.diskStorage({
  destination(req, file, cb) {
    cb(null, uploadDirectory);
  },
  filename(req, file, cb) {
    const safeName = file.originalname.replace(/[^a-zA-Z0-9._-]/g, "_");
    cb(null, `${Date.now()}-${safeName}`);
  }
});
```

Така реалізація дає змогу:
- відокремити завантажені файли від решти коду;
- уникнути конфліктів імен;
- зберігати вкладення в передбачуваному місці.

### 2.5 Валідація типів файлів
Щоб вкладення не перетворилися на неконтрольоване сховище будь-яких файлів, у middleware було додано перевірку MIME-типів.

Допустимими визначено:
- `text/plain`
- `application/pdf`
- `image/png`
- `image/jpeg`
- `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

Фрагмент перевірки:

```js
const allowedMimeTypes = new Set([
  "application/pdf",
  "text/plain",
  "image/png",
  "image/jpeg",
  "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
]);

function fileFilter(req, file, cb) {
  if (!allowedMimeTypes.has(file.mimetype)) {
    return cb(new Error("Unsupported file type"));
  }

  cb(null, true);
}
```

Крім того, було встановлено обмеження на розмір файлу:

```js
limits: {
  fileSize: 5 * 1024 * 1024
}
```

Це означає, що сервер не лише приймає коректні вкладення, а й блокує непридатні типи, що окремо перевірялося під час тестування.

### 2.6 Маршрут для вкладень
Було створено окремий захищений маршрут:

```js
router.post("/attachments", authMiddleware, attachmentUpload.single("file"), uploadAttachment);
```

Тобто:
- завантаження доступне лише авторизованому користувачу;
- приймається один файл за раз;
- після успішної обробки клієнт отримує метадані про вкладення.

Фрагмент контролера:

```js
async function uploadAttachment(req, res) {
  if (!req.file) {
    return res.status(400).json({ message: "file is required" });
  }

  return res.status(201).json({
    message: "Attachment uploaded successfully",
    file: {
      originalName: req.file.originalname,
      fileName: req.file.filename,
      mimeType: req.file.mimetype,
      size: req.file.size,
      path: `/uploads/attachments/${path.basename(req.file.filename)}`
    }
  });
}
```

Це дозволяє використовувати завантаження не як окрему абстрактну функцію, а як практичний інструмент для роботи з задачами.

### 2.7 Моніторинг стану сервера
Для контролю стану процесу було реалізовано маршрут `GET /api/status`, який повертає:
- `uptime`;
- інформацію про пам'ять;
- показники CPU;
- часову мітку.

Фрагмент контролера:

```js
async function getStatus(req, res) {
  const memoryUsage = process.memoryUsage();
  const cpuUsage = process.cpuUsage();

  res.json({
    app: "TaskFlow",
    uptime: Number(process.uptime().toFixed(2)),
    memoryUsage: {
      rss: memoryUsage.rss,
      heapTotal: memoryUsage.heapTotal,
      heapUsed: memoryUsage.heapUsed
    },
    cpuUsage: {
      user: cpuUsage.user,
      system: cpuUsage.system
    },
    timestamp: new Date().toISOString()
  });
}
```

Таким чином, сервер можна швидко перевірити з точки зору продуктивності навіть без зовнішніх інструментів моніторингу.

### 2.8 Практичний результат
У результаті виконання четвертої лабораторної роботи `TaskFlow` отримав:
- файлове та консольне логування;
- фіксацію HTTP-запитів;
- журнал часу виконання маршрутів;
- завантаження вкладень із перевіркою типів;
- захищений маршрут для upload;
- endpoint для перевірки продуктивності;
- збереження вкладень і логів у файловій системі.

У сукупності це переводить проєкт у більш зрілий стан, де вже важливі не лише маршрути й дані, а й зручність супроводу, діагностики та роботи з файлами.

---

## 3. Скріншоти результату

![Logs and uploaded attachments](/assets/labs/lab-4/screen-11.png)

![Get tasks after updates](/assets/labs/lab-4/screen-10.png)

![Update task](/assets/labs/lab-4/screen-9.png)

![Create task](/assets/labs/lab-4/screen-8.png)

![Create task group](/assets/labs/lab-4/screen-7.png)

![Upload valid attachment](/assets/labs/lab-4/screen-6.png)

![Login admin](/assets/labs/lab-4/screen-5.png)

![Performance status](/assets/labs/lab-4/screen-4.png)

![Health check](/assets/labs/lab-4/screen-3.png)

![Server start](/assets/labs/lab-4/screen-2.png)

![Additional server view](/assets/labs/lab-4/screen-1.png)

---

## 4. Висновки

У межах четвертої лабораторної роботи для `TaskFlow` було реалізовано логування запитів і подій, завантаження вкладень до задач та моніторинг стану серверного процесу. Додатково було впроваджено перевірку типів файлів, збереження вкладень у файловій системі та фіксацію тривалості виконання маршрутів. Отриманий результат покращує практичну придатність сервісу й створює основу для подальших етапів, пов'язаних із безпекою, стабільністю та продуктивністю.
