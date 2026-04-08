## 1. Тема, Мета, Посилання

### 1.1 Тема
**Тема:** «РОЗРОБКА ФУНКЦІОНАЛЬНОГО REST API. РЕЄСТРАЦІЯ ТА АВТОРИЗАЦІЯ КОРИСТУВАЧІВ. ВАЛІДАЦІЯ ДАНИХ І ОБРОБКА ПОМИЛОК»

### 1.2 Мета
**Мета:** розширити серверну частину веб-застосунку DrumSpace безпечним RESTful API. Реалізувати процеси реєстрації та авторизації з використанням криптографічного хешування паролів (bcryptjs). Впровадити систему автентифікації на базі JSON Web Tokens (JWT) із розподілом на Access та Refresh токени. Захистити приватні маршрути за допомогою Middleware. Налаштувати авторизацію через сторонні сервіси (Google OAuth 2.0) та реалізувати додаткові механізми безпеки (лімітування запитів, логування помилок).

### 1.3 Посилання

* Репозиторій власного веб-застосунку (GitHub): [посилання](https://github.com/wh0smarg0/IO-35_webLAB-SakhnoMargaryta-FIOT-2026.git)
* Репозиторій звітного HTML-документа (GitHub): [посилання](https://github.com/wh0smarg0/IO-35_appRECORD-SakhnoMargaryta-FIOT-2026.git)
* Звітний HTML-документ (Жива сторінка): [посилання](https://wh0smarg0.github.io/IO-35_appRECORD-SakhnoMargaryta-FIOT-2026/lab/lab-1)

---

## 2. Короткі теоретичні відомості

### 2.1 RESTful API
REST (Representational State Transfer) — це архітектурний стиль побудови веб-сервісів, що базується на HTTP-протоколі. Основний принцип полягає в тому, що сервер працює з "ресурсами" (наприклад, `/users`, `/bookings`), а взаємодія відбувається через стандартні методи: `GET`, `POST`, `PUT`, `DELETE`. Архітектура є stateless — сервер не зберігає стан сесії клієнта, тому кожен запит має містити всі необхідні дані для авторизації.

### 2.2 Хешування паролів (Bcrypt)
Для зберігання паролів у базі даних використано бібліотеку `bcryptjs`. Вона перетворює введений пароль на незворотній криптографічний хеш із додаванням унікальної "солі" (salt). Це гарантує, що навіть у разі витоку бази даних зловмисники не зможуть дізнатися оригінальні паролі користувачів.

### 2.3 JWT (JSON Web Tokens)
JWT — це відкритий стандарт (RFC 7519) для безпечної передачі інформації у вигляді JSON-об'єкта. У проєкті реалізовано дворівневу систему:
* Access Token (короткостроковий, 15 хв) — використовується для доступу до захищених маршрутів.
* Refresh Token (довгостроковий, 7 днів) — використовується для безпечного отримання нового Access Token без необхідності повторного введення логіна та пароля.

### 2.4 Google OAuth 2.0
OAuth 2.0 — це протокол авторизації, який дозволяє застосунку отримати обмежений доступ до даних користувача на іншому сервісі (Google) без передачі пароля. У проєкті використано бібліотеку  `google-auth-library` для валідації ідентифікаційних токенів (ID Token), отриманих від Google, та автоматичної реєстрації/авторизації користувачів у системі DrumSpace.

---

## 3. Проєктування бази даних
Для підтримки нової логіки авторизації модель User була розширена новими полями. Sequelize автоматично синхронізував ці зміни з MySQL за допомогою опції `{ alter: true }`.

Оновлена модель User:
```bash
const User = sequelize.define('User', {
    name: { type: DataTypes.STRING, allowNull: false },
    email: { type: DataTypes.STRING, allowNull: false, unique: true },
    password: { type: DataTypes.STRING, allowNull: false }, // Зберігає bcrypt-хеш
    role: { type: DataTypes.STRING, defaultValue: 'user' }, // Для перевірки прав доступу
    isEmailConfirmed: { type: DataTypes.BOOLEAN, defaultValue: false } // Для підтвердження пошти
});
```
---

## 4. Реалізація backend-частини

### 4.1. Встановити необхідні бібліотеки
Для реалізації функціоналу було встановлено низку npm-пакетів: `express` (маршрутизація), `bcryptjs` (хешування), `jsonwebtoken` (JWT), `sequelize` та `mysql2` (база даних), `express-rate-limit` (захист від спаму), `google-auth-library` (OAuth). Підключення в коді:

```javascript
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const { OAuth2Client } = require('google-auth-library');
const rateLimit = require('express-rate-limit');
const sequelize = require('./config/database');
const fs = require('fs');
const path = require('path');
```

### 4.2. Реалізувати реєстрацію та авторизацію користувача
Створено базові маршрути `POST /register` та `POST /login` для створення облікових записів та перевірки хешованих паролів за допомогою `bcryptjs`.

```javascript
// Хешування при реєстрації
const salt = await bcrypt.genSalt(10);
const hashedPassword = await bcrypt.hash(password, salt);

// Перевірка при логіні
const validPassword = await bcrypt.compare(req.body.password, user.password);
if (!validPassword) {
    return res.status(401).json({ error: "Невірний пароль" });
}
```

### 4.3. Додати валідацію даних, обробку помилок
В усі маршрути додано перевірку на наявність обов'язкових полів та мінімальну довжину пароля. Увесь код загорнуто в блоки `try...catch` для уникнення падіння сервера.

```javascript
if (!name || !email || !password || !passwordConfirm) {
    return res.status(400).json({ message: "Всі поля є обов'язковими" });
}
if (password.length < 6) {
    return res.status(400).json({ message: "Пароль має бути не менше 6 символів" });
}
// ...
catch (err) {
    logError(err, req);
    res.status(500).json({ error: err.message });
}
```

### 4.4. Реалізувати захищений маршрут
Створено ендпоінт для отримання профілю, який вимагає наявності JWT-токена у заголовках запиту. Без нього доступ заборонено.

```javascript
app.get('/profile', authenticateToken, async (req, res) => {
    const user = await User.findByPk(req.user.id, {
        attributes: { exclude: ['password'] } 
    });
    res.json({ message: "Доступ дозволено", user: user });
});
```

### 4.5. Протестувати API через Postman або curl
Тестування всіх створених маршрутів (від реєстрації до OAuth) було успішно проведено у середовищі Postman. Скріншоти успішних та помилкових HTTP-запитів наведено у Розділі 6 даного звіту.

### 4.6. Проаналізувати отримані результати
Аналіз відповідей сервера показав коректну обробку всіх передбачених статус-кодів: `200 OK` (успішні запити), `201 Created` (створення сутності), `400 Bad Request` (помилки валідації), `401 Unauthorized` (помилки авторизації), `403 Forbidden` (недостатньо прав) та `404 Not Found`.

### 4.7. Додати підтвердження пароля при реєстрації
У маршрут `/register` додано порівняння основного пароля з його підтвердженням на стороні бекенду.

```javascript
const { password, passwordConfirm } = req.body;
if (password !== passwordConfirm) {
    return res.status(400).json({ message: "Паролі не співпадають!" });
}
```

### 4.8. Додати роль користувача (admin/user)
Модель користувача розширено полем `role`. Це поле зашивається у JWT-токен і перевіряється при спробі доступу до ресурсів, наприклад, щоб захистити систему від IDOR (Insecure Direct Object Reference).

```javascript
// В моделі User.js
role: {
    type: DataTypes.STRING,
    defaultValue: 'user'
}

// Перевірка прав доступу в маршруті
if (req.user.id !== parseInt(req.params.id) && req.user.role !== 'admin') {
    return res.status(403).json({ error: "Доступ заборонено!" });
}
```

### 4.9. Реалізувати logout
У stateless архітектурі (на базі JWT) сервер просто підтверджує вихід, а фактичне видалення токенів реалізовано на клієнтській стороні у `localStorage`.

```javascript
app.post('/logout', authenticateToken, (req, res) => {
    res.json({
        message: "Ви успішно вийшли з системи.",
        instruction: "Frontend має видалити JWT-токен із localStorage."
    });
});
```

### 4.10. Додати оновлення профілю
Реалізовано маршрут для редагування особистих даних (імені та пошти).

```javascript
app.put('/profile', authenticateToken, async (req, res) => {
    const user = await User.findByPk(req.user.id);
    if (req.body.name) user.name = req.body.name;
    if (req.body.email) user.email = req.body.email;
    await user.save();
    res.json({ message: "Профіль успішно оновлено" });
});
```

### 4.11. Зберігати користувачів у базі
Всі дані користувачів надійно зберігаються у базі даних MySQL за допомогою моделі Sequelize з автоматичною синхронізацією схеми.

```javascript
const User = sequelize.define('User', {
    name: { type: DataTypes.STRING, allowNull: false },
    email: { type: DataTypes.STRING, allowNull: false, unique: true },
    password: { type: DataTypes.STRING, allowNull: false },
    role: { type: DataTypes.STRING, defaultValue: 'user' },
    isEmailConfirmed: { type: DataTypes.BOOLEAN, defaultValue: false }
});
```

### 4.12. Реалізувати refresh token
Реалізовано систему безпеки з двома токенами. У разі "смерті" Access токена (через 15 хвилин), Refresh токен дозволяє отримати новий без повторного вводу пароля.

```javascript
app.post('/refresh', (req, res) => {
    const { refreshToken } = req.body;
    jwt.verify(refreshToken, REFRESH_SECRET_KEY, (err, user) => {
        if (err) return res.status(403).json({ error: "Недійсний Refresh токен" });
        const newAccessToken = jwt.sign(
            { id: user.id, email: user.email, role: user.role },
            SECRET_KEY, { expiresIn: '15m' }
        );
        res.json({ accessToken: newAccessToken });
    });
});
```

### 4.13. Додати логування помилок
Усі винятки `catch` автоматично передаються у спеціальну функцію, яка записує дату, метод і текст помилки у фізичний файл `error.log`.

```javascript
const logError = (err, req) => {
    const timestamp = new Date().toISOString();
    const logMessage = `[${timestamp}] ${req.method} ${req.url} - Помилка: ${err.message}\n`;
    fs.appendFile(path.join(__dirname, 'error.log'), logMessage, () => {});
};
```

### 4.14. Обмежити кількість спроб входу
Для захисту системи від перебору паролів (Brute-force) підключено Middleware `express-rate-limit`.

```javascript
const loginLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 хвилин
    max: 5, // Максимум 5 спроб
    message: { error: "Забагато спроб входу. Спробуйте пізніше." }
});
app.post('/login', loginLimiter, async (req, res) => { /* ... */ });
```

### 4.15. Додати middleware для перевірки токена
Функція-охоронець `authenticateToken` перевіряє HTTP-заголовок `Authorization` перед тим, як пустити користувача до приватного маршруту.

```javascript
const authenticateToken = (req, res, next) => {
    const token = req.headers['authorization']?.split(' ')[1];
    if (!token) return res.status(401).json({ message: "Немає токена." });

    try {
        req.user = jwt.verify(token, SECRET_KEY);
        next();
    } catch (error) {
        res.status(403).json({ message: "Невірний токен." });
    }
};
```

### 4.16. Реалізувати зміну пароля
Захищений маршрут для зміни пароля, який вимагає підтвердження старого пароля перед збереженням нового хешу.

```javascript
app.put('/profile/password', authenticateToken, async (req, res) => {
    const { oldPassword, newPassword } = req.body;
    const user = await User.findByPk(req.user.id);
    
    if (!(await bcrypt.compare(oldPassword, user.password))) {
        return res.status(400).json({ error: "Невірний поточний пароль" });
    }
    
    user.password = await bcrypt.hash(newPassword, await bcrypt.genSalt(10));
    await user.save();
    res.json({ message: "Пароль змінено!" });
});
```

### 4.17. Реалізувати видалення користувача
Маршрут для повного видалення облікового запису з бази даних MySQL. Завдяки зовнішнім ключам Sequelize, пов'язані з юзером бронювання обробляються автоматично.

```javascript
app.delete('/profile', authenticateToken, async (req, res) => {
    const result = await User.destroy({ where: { id: req.user.id } });
    if (result) res.json({ message: "Акаунт успішно видалено" });
    else res.status(404).json({ error: "Не знайдено" });
});
```

### 4.18. Реалізувати відновлення пароля
Реалізовано генерацію одноразового безпечного JWT-токена (з прив'язкою до старого пароля), який діє лише 15 хвилин для скидання забутого пароля.

```javascript
app.post('/forgot-password', async (req, res) => {
    // ...
    const secret = SECRET_KEY + user.password;
    const resetToken = jwt.sign({ email: user.email, id: user.id }, secret, { expiresIn: '15m' });
    res.json({ resetToken: resetToken, message: "Посилання відправлено" });
});
```

### 4.19. Додати підтвердження email
Після реєстрації користувачу генерується спеціальне посилання з токеном. Перехід за цим посиланням змінює статус `isEmailConfirmed` на `true`.

```javascript
app.get('/confirm-email/:token', async (req, res) => {
    const decoded = jwt.verify(req.params.token, SECRET_KEY);
    const user = await User.findByPk(decoded.id);
    
    user.isEmailConfirmed = true;
    await user.save();
    res.send(`<h1>✅ Ваш Email успішно підтверджено!</h1>`);
});
```

### 4.20. Реалізувати OAuth (Google login)
Інтегровано авторизацію через обліковий запис Google. Сервер перевіряє Google ID Token та автоматично створює користувача в базі, якщо він входить вперше, генеруючи йому власні JWT-токени системи.

```javascript
app.post('/auth/google', async (req, res) => {
    const ticket = await googleClient.verifyIdToken({
        idToken: req.body.token,
        audience: GOOGLE_CLIENT_ID
    });
    const { email, name } = ticket.getPayload();

    let user = await User.findOne({ where: { email } });
    if (!user) {
        user = await User.create({
            name, email,
            password: await bcrypt.hash(Math.random().toString(), 10),
            isEmailConfirmed: true
        });
    }

    const accessToken = jwt.sign({ id: user.id, role: user.role }, SECRET_KEY, { expiresIn: '15m' });
    res.json({ accessToken, user: { id: user.id, name: user.name } });
});
```

---

## 5. Інтеграція з Frontend-частиною
Клієнтська частина (HTML/JS) була адаптована для роботи зі stateless-архітектурою:

### 5.1 Збереження токенів: 
Після успішного логіну (через стандартну форму або Google) accessToken та refreshToken зберігаються у localStorage браузера.

### 5.2 Передача заголовків: 
Усі Fetch-запити до захищених маршрутів (отримання бронювань, редагування, видалення) модифіковані для передачі заголовка headers: { 'Authorization': 'Bearer ' + token }.

### 5.3 Обробка розкладу (Захист від подвійних бронювань):
ntgthСтворено публічний маршрут GET /bookings/schedule, який повертає лише час зайнятих кімнат. Фронтенд перевіряє ці дані перед відправкою форми, миттєво блокуючи спроби забронювати зайнятий час.

---

## 6. Результати виконання

### БЛОК 1: Реєстрація та Вхід (Пункти 2, 3, 7, 14)

#### 1. Реєстрація нового користувача (з валідацією та підтвердженням пароля)
* **Метод:** `POST`
* **URL:** `http://localhost:3000/register`
* **Body (raw -> JSON):**
    ```json
    {
        "name": "Марго",
        "email": "margo.test@gmail.com",
        "password": "password123",
        "passwordConfirm": "password123"
    }
    ```
![1](/assets/labs/lab-3/1.png)
* *Очікуваний результат:* Статус `201 Created`. У відповіді буде `userId` та посилання для підтвердження email.

![2](/assets/labs/lab-3/2.png)
* *Як перевірити помилку (Пункт 3):* Зробити `passwordConfirm`: "інший_пароль" і отримати статус `400 Bad Request`.

#### 2. Авторизація / Логін
* **Метод:** `POST`
* **URL:** `http://localhost:3000/login`
* **Body (raw -> JSON):**
    ```json
    {
        "email": "margo.test@gmail.com",
        "password": "password123"
    }
    ```
![3](/assets/labs/lab-3/3.png)
* *Очікуваний результат:* Статус `200 OK`. Отриаємо `accessToken` та `refreshToken`. **Обов'язково скопіювати `accessToken` для наступних запитів!**

![4](/assets/labs/lab-3/4.png)
* *Як перевірити ліміт (Пункт 14):* Натисни кнопку *Send* 6 разів поспіль. На шостий раз отримаєш статус `429 Too Many Requests` ("Забагато спроб входу").

### БЛОК 2: Захищені маршрути та Профіль (Пункти 4, 10, 15, 16)

*⚠️ УВАГА: Для всіх запитів у цьому блоці потрібно додати токен. У Postman перейти у вкладку **Headers**, додати ключ `Authorization`, а в значення встав `Bearer ТВІЙ_СКОПІЙОВАНИЙ_ACCESS_TOKEN`.*

#### 3. Отримання профілю (Перевірка Middleware та захисту)
* **Метод:** `GET`
* **URL:** `http://localhost:3000/profile`
* **Headers:** `Authorization: Bearer <твій_токен>`

![5](/assets/labs/lab-3/5.png)
* *Очікуваний результат:* Статус `200 OK` та дані твого профілю (без пароля).

![6](/assets/labs/lab-3/6.png)
* *Як перевірити захист (Пункт 15):* Вимкни галочку біля заголовка `Authorization` і натисни *Send*. Отримаєш `401 Unauthorized` ("Немає токена").

#### 4. Оновлення профілю (Пункт 10)
* **Метод:** `PUT`
* **URL:** `http://localhost:3000/profile`
* **Headers:** `Authorization: Bearer <твій_токен>`
* **Body (raw -> JSON):**
    ```json
    {
        "name": "Маргарита"
    }
    ```
![7](/assets/labs/lab-3/7.png)
* *Очікуваний результат:* Статус `200 OK` та оновлене ім'я.

#### 5. Зміна пароля (Пункт 16)
* **Метод:** `PUT`
* **URL:** `http://localhost:3000/profile/password`
* **Headers:** `Authorization: Bearer <твій_токен>`
* **Body (raw -> JSON):**
    ```json
    {
        "oldPassword": "password123",
        "newPassword": "new_super_password"
    }
    ```
![8](/assets/labs/lab-3/8.png)
* *Очікуваний результат:* Статус `200 OK` ("Пароль змінено!").

### БЛОК 3: Безпека та Ролі (Пункти 8, 9, 12, 17)

#### 6. Перевірка IDOR (Пункт 8 - Ролі та чужі дані)
* **Метод:** `GET`
* **URL:** `http://localhost:3000/users/9/bookings` *(намагаємося прочитати бронювання юзера з ID 9)*
* **Headers:** `Authorization: Bearer <твій_токен>`

![9](/assets/labs/lab-3/9.png)
* *Очікуваний результат:* Статус `403 Forbidden` ("Доступ заборонено! Ви можете переглядати лише власні записи").

#### 7. Оновлення токена (Пункт 12 - Refresh Token)
* **Метод:** `POST`
* **URL:** `http://localhost:3000/refresh`
* **Body (raw -> JSON):**
    ```json
    {
        "refreshToken": "ТВІЙ_ДОВГИЙ_REFRESH_TOKEN_З_ЛОГІНУ"
    }
    ```
![10](/assets/labs/lab-3/10.png)
* *Очікуваний результат:* Статус `200 OK` і новий свіжий `accessToken`.

#### 8. Logout (Пункт 9)
* **Метод:** `POST`
* **URL:** `http://localhost:3000/logout`
* **Headers:** `Authorization: Bearer <твій_токен>`

![11](/assets/labs/lab-3/11.png)
* *Очікуваний результат:* Статус `200 OK` ("Ви успішно вийшли").

#### 9. Видалення акаунту (Пункт 17)
* **Метод:** `DELETE`
* **URL:** `http://localhost:3000/profile`
* **Headers:** `Authorization: Bearer <твій_токен>`

![12](/assets/labs/lab-3/12.png)
* *Очікуваний результат:* Статус `200 OK` ("Ваш акаунт успішно видалено").

### БЛОК 4: Додаткові фічі (Пункти 18, 19, 20)

#### 10. Відновлення пароля (Пункт 18) - Крок 1
* **Метод:** `POST`
* **URL:** `http://localhost:3000/forgot-password`
* **Body (raw -> JSON):**
    ```json
    {
        "email": "margo.test@gmail.com"
    }
    ```
![13](/assets/labs/lab-3/13.png)
* *Очікуваний результат:* Статус `200 OK`. Сервер видасть спеціальний `resetToken`. Скопіюй його!

#### 11. Відновлення пароля (Пункт 18) - Крок 2
* **Метод:** `POST`
* **URL:** `http://localhost:3000/reset-password`
* **Body (raw -> JSON):**
    ```json
    {
        "id": 1, 
        "token": "СКОПІЙОВАНИЙ_RESET_TOKEN",
        "newPassword": "recovered_password"
    }
    ```
![14](/assets/labs/lab-3/14.png)
*(Заміни `id: 1` на свій реальний ID).*

#### 12. Підтвердження Email (Пункт 19)
Цей запит найпростіше робити прямо в браузері (Chrome), але можна і в Postman.
* **Метод:** `GET`
* **URL:** `http://localhost:3000/confirm-email/ТОКЕН_З_ВІДПОВІДІ_РЕЄСТРАЦІЇ`

![15](/assets/labs/lab-3/15.png)

![16](/assets/labs/lab-3/16.png)
* *Очікуваний результат:* Статус `200 OK` та HTML-код із зеленою галочкою.

#### 13. Google OAuth (Пункт 20)
* **Метод:** `POST`
* **URL:** `http://localhost:3000/auth/google`
* **Body (raw -> JSON):**
    ```json
    {
        "token": "fake_or_real_google_token"
    }
    ```
![17](/assets/labs/lab-3/17.png)
* *Очікуваний результат:* Оскільки в Postman важко отримати реальний токен від Google, ти отримаєш статуc `401 Unauthorized` ("Помилка авторизації через Google"). Це нормально і доводить, що захист працює!

![18](/assets/labs/lab-3/18.png)

### БЛОК 5: (Пункти 11, 13)

#### 14. Збереження користувачів у базі (Пункт 11)

![19](/assets/labs/lab-3/19.png)
Postman підтверджує це побічно. Коли робимо запит POST /register, і сервер повертає статус 201 Created разом із userId (наприклад, "userId": 5) — це означає, що база даних успішно прийняла запис і видала йому цей ідентифікатор. А коли робимо POST /login, бекенд читає ці дані з бази.

#### 15. Логування помилок (Пункт 13)

Postman тут виступає в ролі "шкідника", який має навмисно зламати запит, щоб сервер записав це у файл.
У нашому коді функція logError спрацьовує тоді, коли код потрапляє в блок catch (тобто стається внутрішня помилка сервера — статус 500, або помилка бази даних).

*Як це спровокувати через Postman:*
Зробимо запит, який не пройде валідацію бази даних (наприклад, спробуємо створити бронювання взагалі без даних, щоб база обурилася):
* **Метод:** `POST`
* **URL:** `http://localhost:3000/bookings`
* **Body (raw -> JSON):**
    ```json
    { }
    ```
    (Відправляємо абсолютно пустий об'єкт)

*Що відбудеться:*
* Postman покаже помилку (скоріш за все 400 Bad Request або 500 Internal Server Error, бо roomName не може бути null).
* Наш бекенд спіймає цю помилку в блок catch і викличе logError(err, req).
* Бекенд автоматично створить файл error.log (або допише в існуючий) новий рядок.

![20](/assets/labs/lab-3/20.png)
![21](/assets/labs/lab-3/21.png)

-----

## 9. Висновки

У ході виконання лабораторної роботи було суттєво розширено та вдосконалено серверну частину веб-застосунку DrumSpace. Основною метою було створення безпечного RESTful API з повноцінною системою авторизації, автентифікації та валідації даних користувачів.

У межах роботи було успішно реалізовано:
1.  **Захист конфіденційних даних:** Паролі користувачів більше не зберігаються у відкритому вигляді. Впроваджено криптографічне хешування за допомогою алгоритму `bcryptjs` із використанням унікальної "солі" для кожного запису.
2.  **JWT-Автентифікацію:** Розроблено механізм stateless-авторизації. При вході сервер генерує `Access Token` (для доступу до приватних маршрутів) та `Refresh Token` (для безпечного подовження сесії без повторного введення пароля).
3.  **Middleware-захист:** Створено проміжне програмне забезпечення (`authenticateToken`) для захисту приватних ендпоінтів (перегляд профілю, управління бронюваннями). Успішно реалізовано захист від вразливості IDOR (Insecure Direct Object Reference) — користувач має доступ виключно до власних записів.
4.  **Інтеграцію OAuth 2.0:** Впроваджено можливість безпечного входу в систему за допомогою облікового запису Google з використанням офіційної бібліотеки `google-auth-library`.
5.  **Підвищення надійності API:** Додано обмеження кількості запитів (`express-rate-limit`) для захисту від brute-force атак, а також реалізовано автоматичне логування критичних помилок сервера у файл `error.log`.
6.  **Додатковий функціонал:** Розроблено API-маршрути для імітації підтвердження електронної пошти при реєстрації та безпечного відновлення забутого пароля за допомогою генерації тимчасових токенів.

Усі створені API-маршрути були успішно інтегровані з клієнтською частиною (frontend) за допомогою Fetch API. Отримана система відповідає сучасним стандартам веб-розробки (REST-архітектура), відрізняється високою продуктивністю та гарантує надійний захист даних і сесій користувачів.

-----

## 10. Перелік використаних джерел

1.  Офіційна документація платформи Node.js — *[https://nodejs.org/en/docs/](https://nodejs.org/en/docs/)*
2.  Документація фреймворку Express.js — *[https://expressjs.com/](https://expressjs.com/)*
3.  Офіційна документація Sequelize ORM — *[https://sequelize.org/](https://sequelize.org/)*
4.  Введення в JSON Web Tokens (JWT) та стандарт RFC 7519 — *[https://jwt.io/introduction](https://jwt.io/introduction)*
5.  Документація Google Identity Services (OAuth 2.0 для веб-застосунків) — *[https://developers.google.com/identity/gsi/web/guides/overview](https://developers.google.com/identity/gsi/web/guides/overview)*
6.  Документація пакету bcryptjs для криптографічного хешування — *[https://www.npmjs.com/package/bcryptjs](https://www.npmjs.com/package/bcryptjs)*
7.  Рекомендації OWASP (Open Web Application Security Project) щодо безпеки REST API — *[https://owasp.org/www-project-api-security/](https://owasp.org/www-project-api-security/)*
8.  MDN Web Docs: Використання Fetch API та передача HTTP-заголовків авторизації — *[https://developer.mozilla.org/uk/docs/Web/API/Fetch\_API/Using\_Fetch](https://www.google.com/search?q=https://developer.mozilla.org/uk/docs/Web/API/Fetch_API/Using_Fetch)*
