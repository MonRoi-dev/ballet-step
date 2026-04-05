# AGENT_PLAN: Auth-Service (NestJS + gRPC)

## Этап 1. Планирование и список вопросов
**Контекст:** Реализация сервиса авторизации (`apps/auth-service`) на NestJS с gRPC по контракту `packages/grpc/auth.proto`. База данных: PostgreSQL (`auth_db`), ORM: Prisma.

### Вопросы/Утверждения для архитектуры:
1. **Prisma:** Используем локальную генерацию (`apps/auth-service/prisma/schema.prisma`), так как сервис является Source of Truth для паролей.
2. **Шифрование Email:** Email зашифрован через AES-256-GCM. Для детерминированного поиска при входе добавляется колонка `emailHash` (HMAC).
3. **gRPC маппинг:** Ручное описание TypeScript интерфейсов по DTO, чтобы не усложнять пайплайн сборкой через `ts-proto`.

---

## Этап 2. Реализация логики
1. **Prisma Setup:** Инициализация `schema.prisma` внутри `apps/auth-service`, генерация клиента. Написание `PrismaService`.
2. **Crypto Module:** `CryptoService` для Argon2 хеширования, AES-шифрования и HMAC-хеширования.
3. **Auth Module:** `AuthService` с функциями `register(email, password)`, `login(email, password)` и `validateToken(token)`.
4. **gRPC Контроллер:** `AuthController`, реализующий `AuthService` (protobuf), подключение `GlobalExceptionFilter`.
5. **JWT Конфигурация:** Подключение `@nestjs/jwt`.

---

## Этап 3. Тестирование
- Unit-тесты для `CryptoService` и `AuthService`.
- E2E-тесты инициализации gRPC сервера.

---

## Этап 4. Документация
- Генерация Markdown-описания модуля авторизации и swagger-like контракта.
