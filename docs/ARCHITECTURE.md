# Архитектура и контракты v0.1

## Компоненты
- Web: публичный сайт, формы, калькулятор, портфолио.
- API: Python FastAPI; серверная валидация, авторизация, business logic.
- PostgreSQL: первичное хранилище; Alembic миграции.
- Worker: асинхронная отправка уведомлений через transactional outbox; старт — DB-backed polling, без обязательного Redis.
- Telegram Bot API: уведомления и защищённые ссылки на CRM (без чувствительных подробностей).
- Admin/CRM: отдельный интерфейс с сессиями и ролями.
- Media storage: объектное хранилище для кейсов; подписанные URL для приватных файлов.
- Monitoring: structured logs без PII, health endpoints, error reporting, backups.

## Данные (минимальная схема)
clients(id UUID PK, name, email nullable, phone nullable, telegram nullable, created_at, updated_at)
leads(id UUID PK, client_id FK, status, source, description, assigned_to FK nullable, created_at, updated_at)
estimates(id UUID PK, lead_id FK, pricing_version, currency, items JSONB, total_minor, created_at)
services(id UUID PK, slug UNIQUE, title, active, sort_order)
pricing_versions(id UUID PK, version UNIQUE, published_at, rules JSONB, created_by FK)
users(id UUID PK, email UNIQUE, password_hash, role, active, created_at)
lead_events(id UUID PK, lead_id FK, actor_id nullable, event_type, payload JSONB, created_at)
notes(id UUID PK, lead_id FK, author_id FK, body, created_at)
conversations(id UUID PK, lead_id FK, channel, status, created_at)
messages(id UUID PK, conversation_id FK, direction, body, external_id nullable, created_at)
outbox_events(id UUID PK, topic, payload JSONB, attempts, next_attempt_at, delivered_at nullable, created_at)
consents(id UUID PK, client_id FK, policy_version, purpose, granted_at, source, revoked_at nullable)
audit_events(id UUID PK, actor_id nullable, action, entity_type, entity_id, created_at, metadata JSONB)

Договорённости: timestamps UTC; UUID; индексы leads(status,created_at), leads(client_id), outbox_events(delivered_at,next_attempt_at), messages(conversation_id,created_at). Retention/PII encryption — после юридического решения.

## API (предварительный)
GET /api/v1/public/pricing → опубликованная версия и правила
POST /api/v1/public/estimates/preview → серверная смета без сохранения (не доверять сумме из браузера)
POST /api/v1/public/leads → создать лид и смету; Idempotency-Key; rate limit; captcha/honeypot; consent
GET /api/v1/admin/leads?status=&cursor= → пагинация, RBAC
GET /api/v1/admin/leads/{id} → карточка, смета, история
PATCH /api/v1/admin/leads/{id} → статус, исполнитель, optimistic concurrency
POST /api/v1/admin/leads/{id}/notes → заметка
GET /api/v1/admin/pricing → версии прайса
POST /api/v1/admin/pricing → создать черновик
POST /api/v1/admin/pricing/{id}/publish → публикация
GET /health/live → liveness
GET /health/ready → DB readiness

## Контракт создания лида (пример, не финальная спецификация)
POST /api/v1/public/leads
Headers: Idempotency-Key: UUID
Body: { "name":"Анна", "contact":{"email":"anna@example.org"}, "description":"Нужен сайт", "selection":{"type":"landing","addons":["metrika"]}, "consent":{"accepted":true,"policy_version":"v1"} }
Response 201: { "lead_id":"UUID", "estimate_id":"UUID", "estimated_total_minor":4900000, "currency":"RUB", "status":"new" }
Ошибки: 400 invalid input, 409 conflicting idempotency, 422 validation, 429 rate limit, 503 temporary failure.

## Поток
Browser → API (validation, idempotency) → DB transaction (client, lead, estimate, consent, outbox) → 201.
Worker → outbox → Telegram → delivery status/retry. CRM reads DB, not Telegram.
Обновление цены после создания лида не меняет сохранённую смету.

## Security baseline
HTTPS, secure HttpOnly SameSite cookies, CSRF protection for session-auth writes, Argon2id password hashing, MFA admin, role checks on API, audit trail, secret manager/env, strict CORS, CSP, input limits, server-side pricing, rate limiting, backups, dependency scanning. Не хранить секреты в репозитории.
