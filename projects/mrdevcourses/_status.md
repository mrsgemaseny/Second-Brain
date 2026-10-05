# Статус проекта: MrDevCourses

**Текущая фаза:** Level 3 — Live Production / Боевой Запуск
**Дата обновления:** 2026-10-05
**Текущее состояние:**
- Бэкенд: 100% зелёные тесты JUnit 5 (251/251 тестов), развернут на Render Web Service (`https://mrdevcourses.onrender.com`). Внедрен Google Identity Services (GIS ID Token Flow), исправлен OriginValidationFilter и CORS для боевых доменов `courses.mrsgemaseny.com` и `*.vercel.app`.
- База данных: PostgreSQL 17 на Render (`dpg-db0g80k9v7es73ba8v90-a`), цепочка миграций Flyway V1..V67 успешно накатана.
- Фронтенд: 100% зелёные тесты Vitest (80/80 в 33 сьютах), production сборка без ошибок, развернут на Vercel с привязкой боевого домена (`https://courses.mrsgemaseny.com`). Настроен COOP `same-origin-allow-popups` для Google popup.
- Архитектура: Модульный монолит, Feature-Sliced Design (FSD), строгий монохромный дизайн (Black & White Only), dual layout RBAC (AdminLayout vs Client), cross-domain httpOnly JWT, DB-calculated drip-content, SOS-тикеты и Telegram Bot пульт ментора.
- Статус пилота: Завершен успешно. Платформа готова к приему 2-го коммерческого потока студентов.
