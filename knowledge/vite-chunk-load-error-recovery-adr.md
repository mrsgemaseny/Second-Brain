# ADR: Защита SPA от `Failed to fetch dynamically imported module` при обновлении деплоев (Vite + React Router)

**Статус:** Принято (Implemented in Production)
**Дата:** 2026-10-19
**Проекты:** MrDevCourses (и применимо ко всем SPA на React + Vite: JF-1C, MeDev, Valeur, Envie)

---

## 1. Контекст и суть проблемы

В современных SPA на базе **Vite + React** для ускорения первой загрузки используется **Code Splitting (Lazy Loading)** страниц через `React.lazy(() => import(...))`.
При каждой сборке (`vite build`) бандлер генерирует для каждого маршрута отдельный JS-файл (чанк) с уникальным хэшем содержимого в имени файла, например:
- Сборка №1: `dist/assets/ProjectsPage-D0BisIHN.js`
- Сборка №2 (после нового коммита): `dist/assets/ProjectsPage-BGyvhnsU.js`

При выкатке новой сборки на **Vercel / GitHub Pages / Nginx** старые файлы чанков (`ProjectsPage-D0BisIHN.js`) удаляются и заменяются новыми.

**Сценарий сбоя у пользователя:**
1. Пользователь открыл сайт во вкладке браузера до деплоя. В памяти его вкладки загружен старый `index.html` и старая таблица маршрутов, где записано, что страница `/projects` лежит по адресу `/assets/ProjectsPage-D0BisIHN.js`.
2. Пока вкладка открыта, разработчик делает `git push` -> происходит новый деплой.
3. Пользователь кликает в меню на `/projects`.
4. Браузер делает сетевой запрос за старым файлом `/assets/ProjectsPage-D0BisIHN.js`, которого на сервере больше нет (возвращается `404 Not Found` или HTML-фоллбэк с `text/html` вместо `application/javascript`).
5. Движок браузера выбрасывает `TypeError: Failed to fetch dynamically imported module`, и React Router роняет всё приложение с дефолтным экраном `Unexpected Application Error!`.

---

## 2. Архитектурное решение (3 уровня защиты)

Чтобы пользователь никогда не видел падения приложения при смене билдов на продакшне, внедряется трехуровневая защита:

### Уровень 1: Нативный перехватчик Vite (`vite:preloadError`) в `main.tsx`
Начиная с Vite 4.4+, при ошибке предзагрузки динамического чанка выбрасывается глобальное событие `vite:preloadError`. Его перехват позволяет мгновенно перезагрузить страницу и получить свежий `index.html`:

```typescript
// src/main.tsx
window.addEventListener('vite:preloadError', (event) => {
  event.preventDefault();
  window.location.reload();
});
```

### Уровень 2: Обертка `lazyWithRetry` с защитой от бесконечного цикла (`sessionStorage`)
Если ошибка произошла непосредственно внутри `React.lazy(() => import(...))`, обертка перехватывает сбой промиса и делает **ровно одну** автоматическую перезагрузку страницы в рамках сессии (флаг в `sessionStorage` гарантирует, что если сервер полностью недоступен или у пользователя пропал интернет, страница не уйдет в бесконечный цикл `reload()`):

```typescript
// src/app/router/index.tsx
function lazyWithRetry(
  importer: () => Promise<{ default: React.FC }>
): React.LazyExoticComponent<React.FC> {
  return React.lazy(async () => {
    try {
      const component = await importer();
      if (typeof window !== 'undefined') {
        window.sessionStorage?.removeItem('mrdev_chunk_reload');
      }
      return component;
    } catch (error) {
      const storageKey = 'mrdev_chunk_reload';
      const hasReloaded = typeof window !== 'undefined' && window.sessionStorage?.getItem(storageKey);
      if (typeof window !== 'undefined' && !hasReloaded) {
        window.sessionStorage?.setItem(storageKey, '1');
        window.location.reload();
      }
      throw error;
    }
  });
}
```

### Уровень 3: Кастомный `errorElement` (`RouteErrorFallback`) в React Router
На случай полного отсутствия сети у клиента (когда даже перезагрузка не может скачать чанк) на корневые маршруты (`createBrowserRouter`) вешается `errorElement: <RouteErrorFallback />` в дизайне приложения с кнопкой «Обновить страницу», полностью исключающий появление сырого дефолтного экрана React Router (`Hey developer`).

---

## 3. Последствия (Trade-offs)

- **Плюсы:**
  - Полное устранение `Unexpected Application Error: Failed to fetch dynamically imported module` при частых CI/CD деплоях.
  - Переход на новую версию сайта происходит прозрачно для пользователя прямо в момент клика по ссылке.
  - Защита от бесконечного релоада при реальном обрыве сети благодаря флагу в `sessionStorage`.
- **Минусы / Ограничения:**
  - В момент автоматического `window.location.reload()` сбрасывается несохраненный in-memory стейт открытой страницы (но так как `lazyWithRetry` срабатывает только в момент перехода на **новый** маршрут, пользователь еще ничего не вводил на целевой странице).
