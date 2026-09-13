# Редизайн UI/UX Launch Lab 21 — план реализации

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Перевести интерфейс на бренд Launch Lab 21, добавить стартовый экран с выбором роли и экран модуля с живым превью one-pager, не меняя логику и требования PDF.

**Architecture:** Один файл `index.html`. CSS переписывается целиком на тёмную тему с CSS-переменными. Разметка экранов переписывается внутри существующих view-функций; one-pager выносится в общий рендер `renderSheet`, который используют и страница one-pager, и превью в модуле. Данные, права доступа, роутер и защита черновика сохраняются.

**Tech Stack:** HTML, CSS, vanilla JS, localStorage. Без сборки и внешних ресурсов.

**Отклонение от шаблона навыка:** по просьбе пользователя (мало времени) план не дублирует полный код; код пишется сразу в `index.html`, проверка — браузерным регрессионным скриптом после реализации.

## Global Constraints

- Один файл `index.html`, запуск двойным кликом, без интернета и CDN.
- Ключи хранилища: `ll21.state.v2`, `ll21.session.v1` (в сессию добавляется только `onboarded`).
- Цвета: `--bg #141925`, `--surface #1e2433`, `--surface-2 #262d3f`, `--line #2f3749`, `--ink #eef0f5`, `--muted #9aa3b5`, `--violet #8b3cf7`, `--green #3ee89a`, `--amber #f5b544`, `--red #ff6b6b`.
- Основной текст 17px, подписи не меньше 14px; `prefers-reduced-motion` отключает анимации.
- One-pager — светлый лист (`#ffffff` / `#1b1d22`), печатается без тёмного фона.
- Сохраняются селекторы, на которые опираются тесты: `.journey li.active|done`, `#new-project`, `.pick`, `#profile-form`, `#save-answer`, `#form-error`, `.module-item[data-status]`, `#remaining`, `.onepager`, `.empty-section`, `#saved-banner`, `.just-saved`, `#section-<id>`, `.comment-form[data-module]`, `.comment`, `table.teams tr[data-id]`, `.dot`, `.stat-num`, `#switch-team`, `.role-switch button[data-role]`, `#reset`.

---

### Task 1: Визуальная система и шапка

**Files:** Modify: `index.html` (блок `<style>`, `<header>`)

**Produces:** CSS-классы `.btn(.primary|.ghost|.small|.danger)`, `.card`, `.badge(.done|.partial|.empty|.current)`, `.notice`, `.success`, `.journey`, `.steps/.step`, `.sheet`, `.fade-in`; ссылка логотипа `#/start`.

- [ ] Переписать `:root` на переменные из Global Constraints, тёмный фон, светлый текст, поля ввода на тёмной подложке, видимый фокус `--violet`.
- [ ] Главная кнопка — `background: linear-gradient(90deg,#8b3cf7,#3ee89a)`, тёмный текст.
- [ ] Шапка: логотип «Launch Lab 21» с градиентной «21» → `#/start`, крошки, сегментированный переключатель ролей.
- [ ] Стили one-pager вынести в `.sheet` со своими светлыми цветами; `@media print` оставляет только лист на белом фоне.
- [ ] `@media (prefers-reduced-motion: reduce)` — `animation: none; transition: none`.

**Проверка:** страница открывается без ошибок консоли, текст контрастный, фокус виден при Tab.

### Task 2: Стартовый экран и маршрут онбординга

**Files:** Modify: `index.html` (`viewStart`, `render`, обработчик переключателя ролей)

**Consumes:** `session`, `saveSession()`, `go(hash)`.
**Produces:** `viewStart()`, маршрут `#/start`, кнопки `#start-participant`, `#start-curator`, флаг `session.onboarded`.

- [ ] `viewStart()`: hero «От сырой идеи до one-pager», три шага (Модули → Ответы → One-pager), две карточки ролей.
- [ ] Клик по карточке: `session.role = ...; session.onboarded = true; saveSession(); go(роль === 'curator' ? '#/curator' : '#/')`.
- [ ] В `render()`: `if (parts[0] === 'start') return viewStart();` и если `!session.onboarded` и маршрут `#/` или `#/curator` → `go('#/start')`.
- [ ] Переключатель ролей в шапке тоже ставит `session.onboarded = true`.

**Проверка:** чистый localStorage → открывается старт; «Я из команды» → экран выбора проекта; «Я куратор» → таблица команд; прямая ссылка `#/p/...` старт не показывает.

### Task 3: Страница проекта (кольцо прогресса + дорожка модулей)

**Files:** Modify: `index.html` (`viewProject`, новая `progressRing(pr)`)

**Produces:** `progressRing(pr) → string` (SVG, r=52, `stroke-dasharray` = 326.73, `stroke-dashoffset` = 326.73 × (1 − pct/100), подпись «N из M»).

- [ ] Двухколоночная сетка: слева карточка с кольцом, `#remaining`, «Продолжить: <модуль> →», «Профиль», `#switch-team`, «Открыть one-pager»; справа дорожка `.track` с `.module-item[data-status]`.
- [ ] Узлы: ✓ пройден (зелёный), номер текущего с фиолетовым свечением, номер «впереди» приглушённый; бейджи статуса и 💬 N.
- [ ] Меньше 900px — одна колонка.

**Проверка:** пустой проект → статусы `current,empty,empty`, кольцо 0%; RoomFinder → 3 из 3, кольцо заполнено.

### Task 4: Общий рендер one-pager и экран модуля с живым превью

**Files:** Modify: `index.html` (`renderSheet`, `viewModule`, `viewOnePager`)

**Produces:** `renderSheet(project, { draftModuleId, draft, savedId, preview }) → string`:
- `draft` — объект `{ fieldId: value }` для модуля `draftModuleId`, подменяет сохранённые ответы только при рендере;
- `preview: true` — без комментариев, без id у секций (нет дублей `#section-*`), секция черновика с классом `is-drafting`;
- `savedId` — секция с `.just-saved`.

- [ ] Вынести разметку листа из `viewOnePager` в `renderSheet`; статусы разделов считать через `moduleStatus` на объединённых данных.
- [ ] `viewModule`: сетка `.split` — слева форма (номер модуля, поля, счётчик `#filled-count` «Заполнено X из Y», комментарии, закреплённая панель действий), справа `.preview` с бейджем `#draft-badge` и `#preview-sheet`.
- [ ] На `input`: собрать черновик, `preview-sheet.innerHTML = renderSheet(project, { draftModuleId, draft, preview: true })`, обновить счётчик, бейдж → «Черновик · не сохранено», шаг сценария → 3. В localStorage ничего не писать.
- [ ] Сохранение, ошибка пустой формы, защита черновика — без изменений.
- [ ] Куратор: поля `readonly`, превью сохранённых данных, без `#save-answer`.
- [ ] Меньше 1000px — превью под формой.

**Проверка:** ввод в поле сразу виден в превью; до сохранения `localStorage` не содержит текста; после сохранения one-pager содержит фразу, раздел подсвечен.

### Task 5: One-pager, выбор проекта, профиль и режим куратора в новом стиле

**Files:** Modify: `index.html` (`viewOnePager`, `viewChooseProject`, `viewProfile`, `viewCurator`, `viewNoAccess`, `viewNotFound`)

- [ ] One-pager: панель действий, баннер `#saved-banner`, лист через `renderSheet(project, { savedId })`, обложка с полосой готовности, комментарии под разделами.
- [ ] Выбор проекта и профиль: карточки в новом стиле, селекторы сохранены.
- [ ] Куратор: карточки статистики с градиентными числами, тёмная таблица, клик по строке открывает one-pager.

**Проверка:** три пустых раздела у пустого проекта без поломки вёрстки; куратор видит таблицу из всех команд и оставляет комментарий.

### Task 6: Регрессия, скриншоты, коммит

**Files:** Modify: `README.md` (раздел «Сценарий демо» — стартовый экран и живое превью)

- [ ] Прогнать браузерный регрессионный скрипт: 39 прежних проверок (со стартом через `#start-participant`) + новые: старт и роли, живое превью, отсутствие записи черновика в localStorage, бейдж черновика, кольцо прогресса.
- [ ] Три живые проверки PDF.
- [ ] Скриншоты: старт, модуль, one-pager, куратор; ширина 1280px и 375px; консоль без ошибок.
- [ ] Очистить тестовые данные, закоммитить.
