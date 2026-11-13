# Модель данных v0.1

## Таблицы
- people(id UUID PK, created_at, updated_at): глобальный технический идентификатор личности.
- verified_emails(id, person_id FK, normalized_email, verified_at, UNIQUE(normalized_email)): адрес после проверки; политику смены email и merge определить отдельно.
- projects(id UUID PK, key UNIQUE, name, brand_config JSONB, status, created_at).
- staff_users(id UUID PK, email UNIQUE, auth_provider, created_at): работники CRM.
- project_memberships(project_id FK, staff_user_id FK, role ENUM(project_admin,manager,viewer), PRIMARY KEY(project_id,staff_user_id)).
- project_contacts(id UUID PK, project_id FK, person_id FK, display_name, phone_entered, source, created_at, UNIQUE(project_id,person_id)).
- events(id UUID PK, project_id FK, slug, title, description, starts_at TIMESTAMPTZ, timezone IANA, status, UNIQUE(project_id,slug)); показывать локальную дату и время по timezone.
- registrations(id UUID PK, project_id FK, event_id FK, project_contact_id FK, email_verified_at, messenger_channel ENUM(telegram,max), messenger_verified_at, phone_verified_at NULL, status, created_at, UNIQUE(event_id,project_contact_id)). Проверять согласованность project_id во всех связанных строках.
- email_challenges(id UUID PK, event_id, normalized_email, code_hash, link_token_hash, expires_at, consumed_at, attempts, created_at).
- messenger_challenges(id UUID PK, registration_id, channel, token_hash, expires_at, consumed_at, created_at).
- messenger_accounts(id UUID PK, project_contact_id, channel, provider_user_id, verified_at, UNIQUE(channel,provider_user_id,project_contact_id)).
- contact_notes(id UUID PK, project_id, project_contact_id, author_staff_id, body, created_at).
- tasks(id UUID PK, project_id, project_contact_id NULL, assigned_staff_id, title, status, due_at).
- consent_records(id UUID PK, project_id, person_id, event_id NULL, consent_type, version, granted, captured_at, channel, evidence_reference).
- audit_log(id UUID PK, project_id NULL, actor_staff_id NULL, action, subject_type, subject_id, occurred_at, metadata JSONB).

## Инварианты
1. Любой запрос к участникам, событиям, регистрациям, задачам и заметкам фильтруется по разрешённым project_id.
2. event.project_id == registration.project_id == project_contact.project_id.
3. Один person может иметь несколько project_contacts.
4. Одна registration связывает ровно одно событие и одну карточку участника.
5. Статусы Email и мессенджера меняют только доверенные серверные обработчики.
6. Не подтверждать телефон только фактом отправки ссылки в Telegram или MAX.
7. На пользователя/мероприятие возможна повторная попытка подтверждения без создания дубликатов регистраций.
8. Контакт не может быть виден сотруднику без права доступа к его проекту.
9. Удаление/анонимизация данных учитывает ограничения хранения и ссылки на другие проекты, а не каскадно удаляет глобального person при удалении карточки одного проекта.

## API (черновик)
GET /api/projects — только доступные сотруднику проекты.
GET/POST /api/projects/{projectId}/events
GET /api/projects/{projectId}/contacts
GET /api/projects/{projectId}/events/{eventId}/registrations
POST /public/events/{slug}/email-challenges
POST /public/email-challenges/verify-code
GET /public/email-challenges/verify-link?token=...
POST /public/events/{slug}/registrations
POST /public/registrations/{id}/messenger-challenges
POST /webhooks/telegram
POST /webhooks/max
GET /public/registrations/{id}/status (только с защищённым одноразовым/session-токеном)

Важно: публичные slug должны быть уникальны глобально или URL должен включать ключ проекта. API выше — проектный набросок, не конечный контракт.
