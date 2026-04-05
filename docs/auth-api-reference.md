# Auth Service API Reference

Документация по транспортному слою микросервиса аутентификации. Включает публичный REST API (Web/Mobile клиенты) и внутренний gRPC API (межсервисное взаимодействие).

---

## REST API (Управление сессиями)

Базовый путь: `/auth`

> **Важно:** Все эндпоинты, возвращающие токены (`signup`, `signin`, `refresh`), устанавливают **HttpOnly** Cookie с ключом `refreshToken`. Браузер будет автоматически отправлять эту куку на эндпоинт `/auth/refresh`.

### 1. Регистрация нового пользователя
Создает нового пользователя, возвращает `accessToken` и устанавливает `refreshToken` в куки.

**Эндпоинт:** `POST /auth/signup`

**Тело запроса (JSON):**
| Параметр | Тип | Описание |
| :--- | :--- | :--- |
| `email` | `string` | Валидный email адрес пользователя. |
| `password` | `string` | Пароль (будет захэширован через Argon2). Минимальная длина и сложность валидируются на уровне DTO. |

**cURL пример:**
```bash
curl -X POST http://localhost:3000/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "SecurePassword123!"}'
```

**Успешный ответ (201 Created):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5c...",
  "user": {
    "id": "uuid-string",
    "email": "user@example.com",
    "role": "USER"
  }
}
```
*Заголовки ответа:* Устанавливается заголовок `Set-Cookie: refreshToken=...; HttpOnly; Secure; SameSite=Strict`

---

### 2. Авторизация (Вход)
Аутентифицирует пользователя и создает новую сессию (с привязкой к IP и User-Agent).

**Эндпоинт:** `POST /auth/signin`

**Тело запроса (JSON):**
| Параметр | Тип | Описание |
| :--- | :--- | :--- |
| `email` | `string` | Email пользователя |
| `password` | `string` | Пароль |

**cURL пример:**
```bash
curl -X POST http://localhost:3000/auth/signin \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "SecurePassword123!"}'
```

**Успешный ответ (201 Created):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5c...",
  "user": {
    "id": "uuid-string",
    "email": "user@example.com",
    "role": "USER"
  }
}
```

---

### 3. Обновление токена (Refresh Token Rotation)
Выдает новый `accessToken` и обновляет `refreshToken` в куках (старый RT инвалидируется). При попытке использовать старый `refreshToken` (Reuse Detection) все сессии пользователя уничтожаются.

**Эндпоинт:** `POST /auth/refresh`

**Заголовки / Куки:**
Обязательно наличие `HttpOnly` Cookie `refreshToken`.

**cURL пример:**
```bash
curl -X POST http://localhost:3000/auth/refresh \
  -b "refreshToken=eyJhbGciOiJIUzI1NiIsInR5c..." \
  -H "Content-Type: application/json"
```

**Успешный ответ (201 Created):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5c..."
}
```

---

### 4. Выход (Завершение текущей сессии)
Уничтожает текущую сессию пользователя в базе данных, заносит `accessToken` в Redis Blacklist и очищает `HttpOnly` Cookie.

**Эндпоинт:** `POST /auth/logout`

**Заголовки:**
- `Authorization: Bearer <accessToken>` (Обязателен)

**cURL пример:**
```bash
curl -X POST http://localhost:3000/auth/logout \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5c..." \
  -H "Content-Type: application/json"
```

**Успешный ответ (201 Created):**
```json
{
  "success": true
}
```

---

### 5. Выход со всех устройств (Terminate All Sessions)
Удаляет все активные сессии пользователя (например, при подозрительной активности).

**Эндпоинт:** `POST /auth/logout-all`

**Заголовки:**
- `Authorization: Bearer <accessToken>` (Обязателен)

**cURL пример:**
```bash
curl -X POST http://localhost:3000/auth/logout-all \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5c..." \
  -H "Content-Type: application/json"
```

**Успешный ответ (201 Created):**
```json
{
  "success": true
}
```

---

## gRPC API (Внутренние микросервисы)

Используется исключительно сервисами внутреннего контура для высокоскоростной аутентификации HTTP-запросов (например, API Gateway).

### Проверка токена (ValidateToken)
Проверяет валидность `accessToken` (алгоритм подписи) и его отсутствие в **Redis Blacklist**.

**Сервис и Метод:** `AuthService.ValidateToken`

**RPC Запрос (ValidateTokenRequest):**
| Поле | Тип | Описание |
| :--- | :--- | :--- |
| `token` | `string` | `accessToken`, извлеченный другим микросервисом из запроса клиента |

**gRPCurl пример:**
```bash
grpcurl -plaintext -d '{"token": "eyJhbG..."}' localhost:50051 auth.AuthService/ValidateToken
```

**RPC Ответ (ValidateTokenResponse):**
| Поле | Тип | Описание |
| :--- | :--- | :--- |
| `isValid` | `boolean` | `true`, если токен подлинный и не в блэклисте |
| `userId` | `string` | Идентификатор пользователя (из payload `sub`) |
| `role` | `string` | Роль пользователя (например, `USER`, `ADMIN`) |

*Примечание: Если `isValid` возвращает `false`, сервис-потребитель должен отклонить запрос со статусом `401 Unauthorized`.*
