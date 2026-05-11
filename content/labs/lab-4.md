## 1. Тема, Мета, Посилання

### 1.1 Тема
**Тема:** «Розширені можливості Node.js-додатків: логування, завантаження файлів, моніторинг продуктивності»

### 1.2 Мета
**Мета:** реалізувати систему комплексного логування серверних подій за допомогою бібліотек Morgan та Winston. Впровадити функціонал завантаження медіафайлів (фото репетиційних баз) за допомогою Middleware Multer із налаштуванням валідації типів та розмірів. Розробити інструменти моніторингу стану сервера (uptime, memory usage) та вимірювання часу відповіді. Забезпечити стабільну роботу застосунку за допомогою менеджера процесів PM2.

### 1.3 Посилання

* Репозиторій власного веб-застосунку (GitHub): [посилання](https://github.com/wh0smarg0/IO-35_webLAB-SakhnoMargaryta-FIOT-2026.git)
* Репозиторій звітного HTML-документа (GitHub): [посилання](https://github.com/wh0smarg0/IO-35_appRECORD-SakhnoMargaryta-FIOT-2026.git)
* Звітний HTML-документ (Жива сторінка): [посилання](https://wh0smarg0.github.io/IO-35_appRECORD-SakhnoMargaryta-FIOT-2026/lab/lab-4)

---

## 2. Короткі теоретичні відомості

### 2.1 Winston та Morgan
Winston — це універсальна бібліотека для логування, яка дозволяє записувати події одночасно в різні місця: консоль, файли або зовнішні сервіси. Morgan — це Middleware, що спеціалізується виключно на логуванні HTTP-запитів, автоматично збираючи дані про метод, шлях, статус-код та час відповіді.


### 2.2 Multer
Multer — це Middleware для Node.js, призначений для обробки `multipart/form-data`. Він забезпечує збереження завантажених файлів на диск, дозволяє фільтрувати їх за розширенням (MIME-типи) та обмежувати максимальний об'єм даних.


### 2.3 Моніторинг та PM2
Моніторинг передбачає збір метрик про стан системи: використання оперативної пам'яті (`process.memoryUsage()`) та тривалість роботи сервера (`process.uptime()`). PM2 — це менеджер процесів, який забезпечує фонову роботу сервера, автоматичний перезапуск у разі збоїв та агрегацію логів.


---

## 3. Проєктування бази даних
Для підтримки завантаження фото репетиційних баз модель `Room` була адаптована для зберігання шляхів до фізичних файлів.

Оновлена модель Room:
```javascript
const Room = sequelize.define('Room', {
    name: { type: DataTypes.STRING, allowNull: false },
    badge: { type: DataTypes.STRING },
    location: { type: DataTypes.STRING },
    gear: { type: DataTypes.TEXT },
    price: { type: DataTypes.INTEGER },
    image: { type: DataTypes.STRING, allowNull: false } // Шлях до файлу: /uploads/image_name.png
});
```

---

## 4. Реалізація backend-частини

### 4.1. Налаштування Winston (Завдання 2, 3)
Реалізовано систему логування з ротацією файлів щодня.
```javascript
const logger = winston.createLogger({
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.Console(),
        new winston.transports.File({ filename: 'logs/app.log', level: 'info' }),
        new winston.transports.DailyRotateFile({
            filename: 'logs/application-%DATE%.log',
            datePattern: 'YYYY-MM-DD',
            maxFiles: '14d'
        })
    ]
});
```

### 4.2. Налаштування Multer та валідація (Завдання 5, 6, 7)
Додано перевірку типів файлів (тільки зображення) та обмеження розміру (1MB).
```javascript
const storage = multer.diskStorage({
    destination: 'uploads/',
    filename: (req, file, cb) => cb(null, Date.now() + path.extname(file.originalname))
});

const upload = multer({
    storage: storage,
    limits: { fileSize: 1024 * 1024 }, // 1MB
    fileFilter: (req, file, cb) => {
        const allowedTypes = /jpeg|jpg|png|gif/;
        const isAllowed = allowedTypes.test(file.mimetype);
        if (isAllowed) cb(null, true);
        else cb(new Error("Недопустимий формат файлу!"));
    }
});
```

### 4.3. Моніторинг та час відповіді (Завдання 8, 9)
Розроблено Middleware для вимірювання швидкості обробки запитів.
```javascript
app.use((req, res, next) => {
    const start = Date.now();
    res.on('finish', () => {
        const duration = Date.now() - start;
        logger.info({ method: req.method, url: req.url, duration: `${duration}ms` });
    });
    next();
});

app.get('/status', (req, res) => {
    res.json({
        uptime: process.uptime().toFixed(0) + "s",
        memory: process.memoryUsage().heapUsed
    });
});
```

---

## 5. Інтеграція з Frontend-частиною
1.  **Сторінка `add-base.html`**: Реалізовано динамічну форму з `FormData` для передачі текстових даних разом із файлом зображення.
2.  **Обробка результату**: Додано UI-фідбек, який показує статус завантаження та назву збереженого файлу з сервера.
3.  **Виклик статус-панелі**: На головній сторінці додано приховану перевірку статусу сервера для дебагу.

---

## 6. Результати виконання

### 6.1. Логування HTTP-запитів (Завдання 2, 3, 9)
У файлі `logs/app.log` зафіксовано успішне виконання запитів із вказанням часу відповіді:
```json
{"level":"info","message":{"duration":"12ms","method":"GET","url":"/api/rooms"},"timestamp":"2026-05-11T12:00:01.450Z"}
```

### 6.2. Валідація завантаження (Завдання 7)
Спроба завантажити файл `.txt` через Postman або форму:
* **Результат:** Статус `400 Bad Request`.
* **JSON Відповідь:** `{"error": "Недопустимий формат файлу!"}`.

### 6.3. Моніторинг стану (Завдання 8)
Запит на `GET /status`:
```json
{
  "uptime": "3600s",
  "memoryUsage": { "rss": 45056000, "heapTotal": 15073280, "heapUsed": 10245600 }
}
```

### 6.4. Управління процесами (Завдання 10)
Виклик команди `pm2 status` показує стабільний процес `drumspace` з 0 рестартів та актуальним споживанням CPU.

---

## 9. Висновки

У ході виконання лабораторної роботи було успішно реалізовано систему моніторингу та логування для веб-застосунку DrumSpace. 

1.  **Надійність:** Використання Winston дозволило централізувати обробку помилок, що значно полегшує дебаг у майбутньому.
2.  **Контент:** Завдяки Multer користувачі тепер можуть наповнювати каталог новими кімнатами, завантажуючи реальні фото баз. Система валідації гарантує безпеку сервера від завантаження шкідливих або занадто великих файлів.
3.  **Продуктивність:** Створені інструменти моніторингу дозволяють відстежувати "витоки" пам'яті та ідентифікувати повільні маршрути через логування `duration`.
4.  **Стійкість:** Інтеграція PM2 забезпечує автоматичне відновлення сервера після неочікуваних помилок, що критично для реальних веб-проєктів.

---

## 10. Перелік використаних джерел

1. Офіційна документація Multer — [npmjs.com/package/multer](https://www.npmjs.com/package/multer)
2. Гід з логування Winston — [github.com/winstonjs/winston](https://github.com/winstonjs/winston)
3. Документація PM2 Process Manager — [pm2.keymetrics.io](https://pm2.keymetrics.io/)
4. Node.js `process` API — [nodejs.org/api/process.html](https://nodejs.org/api/process.html)
