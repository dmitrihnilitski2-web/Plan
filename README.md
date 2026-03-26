Комплексний план рефакторингу та оптимізації Fitness Hub Pro (FBot_V_2.0)
Цей документ є детальним технічним керівництвом, яке описує кожен крок, необхідний для безпечного розбиття поточного проекту на підтримувані, масштабовані та безпечні модулі. Усі зміни слід впроваджувати поетапно, використовуючи шаблон Strangler Fig (поступова заміна частин моноліту), щоб проект продовжував працювати під час оновлень.

Етап 1: Конфігурація та Безпека (Основа)
Ціль: Видалити хардкод, налаштувати централізоване управління змінними середовища та закрити вразливості DDOS і XSS.

Створення 
config.py
 (оновлено) та .env файлу:

Створити файл .env у корені проекту. Записати туди:
env
BOT_TOKEN=ваш_токен
ADMIN_ID=1100202114
WEBAPP_URL=https://ваш_ngrok_url
DB_URL=sqlite+aiosqlite:///fitness_bot.db
Оновити 
config.py
, щоб він читав ці змінні через бібліотеку python-dotenv або pydantic-settings.
Дія: Замінити всі входження "1100202114" у 
server.py
 та 
bot.py
 на config.ADMIN_ID.
Захист від XSS на Frontend (
client.js
):

У функції window.formatMarkdown додається генерація HTML. Якщо туди потрапляють дані оброблені ШІ, це потенційний XSS.
Дія: Підключити бібліотеку DOMPurify (через CDN в 
index.html
) і завжди обертати вивід у DOMPurify.sanitize(...).
Приклад: document.getElementById('plan-explanation').innerHTML = DOMPurify.sanitize(window.formatMarkdown(userData.workout_plan.explanation));
Захист від DoS/Spam на Backend (
server.py
):

Ендпоінт /api/log_client_error можна викликати нескінченно, забиваючи логи сервера об'ємними стектрейсами.
Дія: Додати slowapi (FastAPI Rate Limiter) на цей маршрут (наприклад, 5 запитів на хвилину від одного IP або ID користувача).
Етап 2: Розбиття Бази Даних (Data Access Layer)
Ціль: Перейти від монолітного 
database.py
 (1500+ рядків) до системи репозиторіїв та правильних міграцій.

Нова структура папок бази даних: Створити директорію app/db/ з такою структурою:

app/
  db/
    __init__.py
    connection.py     # логіка aiosqlite.connect (генератор сесій)
    models.py         # SQLAlchemy моделі (переписані з CREATE TABLE)
    repositories/
      __init__.py
      user_repo.py    # функції get_user, save_user
      workout_repo.py # get_today_completed_sets, log_workout_set
      social_repo.py  # друзі, дуелі, соцмережі
Перехід на SQLAlchemy ORM (Рекомендовано) або структурування aiosqlite:

Варіант А (Простий): Залишити aiosqlite, але рознести SQL-запити з 
database.py
 по файлах repositories/*. Усі функції, що зараз є в 
database.py
, просто імпортуватимуться в 
server.py
 з нових файлів репозиторіїв (напр. from app.db.repositories.user_repo import get_user). Це найшвидший шлях, що не зламає код.
Міграції: Видалити блок try...except pass із 
init_db
. Замість нього використовувати систему міграцій Alembic (якщо застосовано SQLAlchemy) або хоча б створити надійний скрипт migrations.py, який перевірятиме схему PRAGMA table_info перед ALTER TABLE.
Перенесення логіки ініціалізації: Після розбиття, await database.init_db() в 
server.py
 зміниться на імпорт і виклик ініціалізатора з app/db/connection.py.

Етап 3: Рефакторинг Backend API (
server.py
)
Ціль: Розвантажити файл на понад 2100 рядків, перетворивши його на чистий MVC (Model-View-Controller) за допомогою APIRouter FastAPI.

Створення директорії app/api/: Створити маршрутизатори (Routers) за доменами та перенести відповідні ендпоінти з 
server.py
:

app/api/auth.py (TG Middleware, 
verify_tg_init_data
, 
get_current_user
).
app/api/users.py (/api/profile, /api/user/{user_id}, /api/edit_profile, вага).
app/api/workouts.py (/api/generate_plan, /api/log_set, /api/checkin, адаптація).
app/api/nutrition.py (/api/user/food_prefs, фото їжі, улюблені страви).
app/api/social.py (дуелі, квести, друзі).
app/api/admin.py (/api/admin/stats, /api/admin/users).
Винесення Pydantic Моделей: Всі класи типу 
UserData
, 
SetLog
, 
CheckinData
, що зараз на початку 
server.py
, вирізати і вставити у новий файл app/schemas.py.

Ізоляція бізнес-логіки (Services): У роутах (наприклад /api/checkin) міститься і валідація, і логіка перевірки тарифу (PAYWALL), і виклик моделі ШІ.

Ця логіка має лежати в app/services/workout_service.py.
Відповідно ендпоінт матиме вигляд усього з 3-5 рядків, делегуючи генерацію плану до сервісного рівня.
Збірка нового main.py (Заміна 
server.py
): Створити легкий main.py:

python
from fastapi import FastAPI
from app.api import users, workouts, social, admin ...
app = FastAPI(lifespan=lifespan)
app.include_router(users.router)
app.include_router(workouts.router)
# ... інше
Етап 4: Перебудова Telegram Бота (
bot.py
)
Ціль: Стабілізувати роботу планувальника, відділивши бекграунд-задачі від обробки повідомлень.

Виділення задач (Scheduler):

Функції 
cancel_expired_pending_duels
, 
judge_active_duels
, 
run_notifications
 та 
scheduler_task
 перенести у файл app/tasks/scheduler.py.
Замість циклу while True: await asyncio.sleep(30) підключити бібліотеку APScheduler (або Celery для серйознішого навантаження). Це інтегрується безпосередньо в main.py на старті додатку (FastAPI 
lifespan
), а не в процес бота.
Розбиття Хендлерів AI-бота:

Створити папку bot/handlers/.
start.py -> перенести логіку /start та Deep Linking (запрошення, реферали).
photo.py -> перенести обробку @dp.message(F.photo) (їжа).
chat.py -> перенести обробку простого тексту (чат з ШІ-тренером).
Зібрати всі ці роутери в 
bot.py
 (як диспетчер).
Етап 5: Модернізація Frontend-Архітектури
Ціль: Організувати файли так, щоб мінімізувати розмір 
index.html
 і розбити JavaScript на модулі. Ми не будемо впроваджувати React/Vue негайно, але використаємо ES Modules.

Розбиття 
index.html
 (152KB):

Кожна "view" (напр., <div id="profile-view">, <div id="nutrition-view">) переноситься в окремі файли в папці templates/partials/.
В 
index.html
 використовуємо синтаксис Jinja2:
html
{% include "partials/profile.html" %}
{% include "partials/dashboard.html" %}
Це збереже швидкість рендерингу і зменшить розмір файлу розробника.
Розбиття JavaScript Моноліту (
client.js
):

Додати type="module" в тезі скрипта <script src="/static/js/main.js" type="module"></script>.
Створити js/api/ папку. Усі виклики типу 
fetch('/api/user/...')
 зібрати у модулі api.js.
Розбити глобальну змінну userData:
Створити store.js для управління станом (State Management).
export const state = { user: null, progress: null };
Створити ui/dashboard.js, ui/profile.js, ui/nutrition.js і там помістити відповідні функції рендерингу (напр. 
renderApp
, 
submitProfile
).
Оновлення CSS:

Перевірити 
style.css
 (45KB) на наявність дублювання (особливо .bento-card стилі). Зменшити розмір, використовуючи загальні утилітні класи (utility-first підхід, подібний до Tailwind, або CSS variables).
План виконання (Action Items для розробника)
Якщо ви готові розпочати, порядок дій буде наступним:

[Крок 1]: Зміна логіки конфігурації (створення .env та налаштування безпеки). Ми можемо зробити це одразу без простоїв.
[Крок 2]: Розділення 
database.py
 на репозиторії. Ми створимо app/db/repositories/ і акуратно перенесемо методи, виправивши імпорти.
[Крок 3]: Розділення 
server.py
. Ми створимо app/api/ та зареєструємо роутери FastAPI.
[Крок 4]: Винесення 
scheduler_task
 з 
bot.py
 у правильний планувальник.
[Крок 5]: Розділення 
index.html
 та 
client.js
.
TIP

Під час усього процесу застосунок НЕ БУДЕ зламаний. Всі зміни вноситимуться так, що API відповіді повністю співпадатимуть зі старими версіями, а бази даних не загублять жодного байта. Напишіть, якщо погоджуєтесь із таким об'ємним планом і хочете стартувати з Етапу 1 та Етапу 2.
