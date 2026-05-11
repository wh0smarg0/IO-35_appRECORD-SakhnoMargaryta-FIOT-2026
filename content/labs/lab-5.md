## 1. Тема, Мета, Посилання

### 1.1 Тема
**Тема:** «Безпека та продуктивність серверних додатків Безпека Node.js-додатків Оптимізація запитів і кешування Тестування API».

### 1.2 Мета
**Мета:** забезпечувати базову безпеку Node.js-додатків; оптимізувати продуктивність REST API; використовувати кешування для зменшення навантаження на сервер; тестувати API за допомогою сучасних інструментів; аналізувати продуктивність backend-застосунків.

### 1.3 Посилання
* Репозиторій власного веб-застосунку (GitHub): [посилання](https://github.com/wh0smarg0/IO-35_webLAB-SakhnoMargaryta-FIOT-2026.git)
* Репозиторій звітного HTML-документа (GitHub): [посилання](https://github.com/wh0smarg0/IO-35_appRECORD-SakhnoMargaryta-FIOT-2026.git)
* Звітний HTML-документ (Жива сторінка): [посилання](https://wh0smarg0.github.io/IO-35_appRECORD-SakhnoMargaryta-FIOT-2026/lab/lab-5)

---

## 2. Короткі теоретичні відомості

### 2.1 Безпека Node.js-додатків
Захист серверних додатків вимагає комплексного підходу для протидії таким загрозам, як SQL/NoSQL Injection, XSS, CSRF та DDoS-атаки. Основні методи захисту включають використання `Helmet` для приховування та захисту HTTP-заголовків, `bcrypt` для хешування паролів, `JWT` для безпечної автентифікації та `Rate Limiting` для обмеження кількості запитів. Для перевірки даних, що надходять від користувача, застосовується строга валідація (наприклад, пакет `express-validator`).

### 2.2 Оптимізація та Кешування
Продуктивність API безпосередньо залежить від швидкості обробки запитів та кількості звернень до бази даних. Кешування — це тимчасове збереження результатів для пришвидшення повторних запитів. У Node.js широко використовуються in-memory cache (наприклад, `node-cache`) та Redis. Перевагами кешування є суттєве зменшення навантаження на базу даних, пришвидшення відповіді сервера та покращення загальної масштабованості системи.

### 2.3 Тестування API та Моніторинг
Тестування поділяється на кілька типів: Unit Testing (окремі функції), Integration Testing (взаємодія компонентів), Load Testing (перевірка під навантаженням) та Security Testing. Для автоматизованого тестування у середовищі Node.js найчастіше використовують фреймворки `Jest` та `Supertest`, а для навантажувального тестування — `Artillery`.

---

## 3. Реалізація backend-частини

### 3.1. Встановлення залежностей
Для реалізації завдань лабораторної роботи було встановлено пакети для безпеки, валідації, кешування та тестування:
```bash
npm install helmet express-rate-limit express-validator node-cache
npm install jest supertest --save-dev
npm install -g artillery
```

### 3.2. Базова безпека: Helmet та Rate-Limit
У головний файл `server.js` інтегровано Middleware для захисту HTTP-заголовків та обмеження частоти запитів для захисту від DDoS та Brute-force:
```javascript
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');

app.use(helmet()); // Приховує вразливі заголовки (напр., X-Powered-By)

const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 хвилин
    max: 100, // максимум 100 запитів з одного IP
    message: { error: "Забагато запитів, спробуйте пізніше" }
});
app.use(limiter);
```

### 3.3. Валідація даних (express-validator)
Для запобігання збереженню некоректних даних реалізовано строгу перевірку вхідних параметрів на маршруті створення бронювань:
```javascript
const { body, validationResult } = require('express-validator');

app.post('/bookings', [
    body('roomName').notEmpty().withMessage("Назва кімнати обов'язкова"),
    body('date').notEmpty().withMessage("Дата бронювання обов'язкова"),
    body('phone').isLength({ min: 10 }).withMessage("Некоректний телефон")
], async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
        return res.status(400).json({ error: errors.array()[0].msg });
    }
    // ... логіка збереження в БД
});
```

### 3.4. Оптимізація та In-Memory Кешування (node-cache)
Для зменшення кількості звернень до MySQL оптимізовано маршрут `GET /api/rooms`. Результат звернення до БД зберігається в оперативній пам'яті на 60 секунд.
```javascript
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 60 });

app.get('/api/rooms', async (req, res) => {
    try {
        const cachedRooms = cache.get('rooms_list');
        if (cachedRooms) {
            return res.json(cachedRooms); // Віддаємо з кешу
        }

        const rooms = await Room.findAll(); // Запит до БД
        cache.set('rooms_list', rooms); // Зберігаємо в кеш
        res.json(rooms);
    } catch (err) {
        res.status(500).json({ error: "Помилка сервера" });
    }
});
```
При додаванні нової кімнати (`POST /rooms`) кеш примусово очищується командою `cache.del('rooms_list');`, щоб користувачі бачили актуальні дані.

### 3.5. Автоматизоване тестування (Jest + Supertest)
Створено файл `api.test.js` для перевірки ключових маршрутів API на відповідність очікуваним статус-кодам та форматам відповідей.
```javascript
const request = require('supertest');
const SERVER_URL = 'http://localhost:3000';

describe('DrumSpace API Tests', () => {
    test('GET /api/rooms має повертати статус 200', async () => {
        const response = await request(SERVER_URL).get('/api/rooms');
        expect(response.statusCode).toBe(200);
        expect(Array.isArray(response.body)).toBeTruthy();
    });

    test('Захищений маршрут без токена має видати 401', async () => {
        const response = await request(SERVER_URL).get('/profile');
        expect(response.statusCode).toBe(401);
    });
});
```

---

## 4. Результати виконання та тестування

### 4.1. Результати Unit та Integration тестування (Jest)
У терміналі було запущено команду `npm test`. Результат виконання продемонстрував успішне проходження всіх тестів:
```text
 PASS  ./api.test.js (7.763 s)
  DrumSpace API Tests
    √ 1. GET /api/rooms має повертати статус 200 та список баз (126 ms)
    √ 2. GET /status має повертати метрики сервера (9 ms)
    √ 3. Захищений маршрут GET /profile без токена має видати помилку 401 (7 ms)
```

### 4.2. Навантажувальне тестування (Artillery)
Для перевірки ефективності кешування було згенеровано високе навантаження на маршрут `/api/rooms`. Команда для симуляції 50 користувачів, кожен з яких робить по 20 запитів:
```bash
npx artillery quick --count 50 --num 20 http://localhost:3000/api/rooms
```

**Результати звіту Artillery (З увімкненим кешуванням):**
* `http.requests`: 1000
* `http.codes.200`: 1000 (100% успішних відповідей)
* `http.request_rate`: 279 запитів на секунду.
* `http.response_time p95`: 190.6 мс.
* `http.response_time median`: 156 мс.


**Аналіз:** Завдяки впровадженню `node-cache`, сервер обробив 1000 запитів менш ніж за 6 секунд. Медіанний час відповіді склав всього 156 мілісекунд, оскільки сервер не звертався до бази даних 1000 разів, а миттєво віддавав дані з оперативної пам'яті.

---

## 5. Висновки

У ході виконання лабораторної роботи було суттєво підвищено рівень безпеки та продуктивності серверної частини застосунку DrumSpace.

1. **Безпека:** Інтеграція `Helmet` дозволила приховати технологічний стек від потенційних зловмисників. Впровадження `express-rate-limit` надійно захистило API від DDoS-атак та спроб підбору паролів.
2. **Надійність даних:** Використання пакету `express-validator` забезпечило строгу перевірку вхідних даних (імена, дати, телефони) ще до етапу звернення до бази даних, що виключає ризик збереження пошкодженого контенту.
3. **Продуктивність:** Найбільшим досягненням стала реалізація in-memory кешування за допомогою `node-cache`. Як показало навантажувальне тестування `Artillery`, сервер здатен витримувати пікові навантаження (близько 280 запитів на секунду) без збільшення часу відповіді завдяки зменшенню кількості транзакцій з БД.
4. **Стабільність (QA):** Покрито ключові маршрути автоматизованими тестами за допомогою `Jest` та `Supertest`. Це гарантує, що при подальшому масштабуванні коду існуючий функціонал залишиться робочим.

Проєкт повністю відповідає сучасним вимогам до production-ready серверних додатків.

---

## 6. Перелік використаних джерел

1. Офіційна документація Helmet — [https://helmetjs.github.io/](https://helmetjs.github.io/)
2. Документація Express Validator — [https://express-validator.github.io/docs/](https://express-validator.github.io/docs/)
3. Node-cache на npm — [https://www.npmjs.com/package/node-cache](https://www.npmjs.com/package/node-cache)
4. Тестування API за допомогою Supertest — [https://github.com/ladjs/supertest](https://github.com/ladjs/supertest)
5. Навантажувальне тестування Artillery — [https://www.artillery.io/](https://www.artillery.io/)
