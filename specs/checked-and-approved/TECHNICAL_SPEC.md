# Mealora Technical Spec / Техническая спецификация

**Статус:** checked-and-approved

## 1. Назначение

Эта техническая спецификация описывает техническую структуру Mealora v1.

Документ покрывает:

- отправку публичной заявки;
- авторизацию клиента и личный кабинет;
- хранение заявок в базе данных;
- demo payment lifecycle;
- авторизацию администратора;
- защищенные admin routes;
- управление заявками в админ-зоне;
- базовые правила безопасности;
- технические крайние случаи.

Документ поддерживает:

- `BUSINESS_SPEC.md`
- `STYLE_UI_SPEC.md`
- `FUNCTIONAL_MAP_SPEC.md`

## 2. Текущий scope

Mealora v1 включает:

- публичные маркетинговые страницы и страницы заявки;
- русскую и английскую локализацию клиентского интерфейса;
- публичную форму заявки;
- login и registration клиента;
- dashboard личного кабинета клиента;
- список заявок клиента;
- детальную страницу заявки клиента;
- demo payment page после одобрения заявки;
- хранение заявок в базе данных;
- admin login;
- admin dashboard;
- список заявок в админ-зоне;
- детальную страницу заявки в админ-зоне;
- обновление статусов;
- внутренние admin notes.

Mealora v1 не включает:

- автоматическое списание подписки;
- автоматическое подтверждение заказа;
- управление меню;
- управление рецептами;
- tracking доставки;
- real payment processing внутри админ-зоны;
- CliQ payment processing;
- card payment processing;
- payment links;
- обработку реальных денежных платежей;
- повтор заказа в один клик;
- программу лояльности.

Админ-зона v1 не обрабатывает реальные платежи и не подключает payment provider. При этом admin может вручную обновлять `payment_status` и `admin_notes`, чтобы зафиксировать demo payment или вручную согласованную оплату вне сайта.

## 3. Техническое решение по stack

Использовать Supabase для:

- PostgreSQL database;
- customer authentication;
- admin authentication;
- row-level security;
- server-side хранения заявок.

Публичный сайт не должен раскрывать privileged database keys в браузере.

## 4. Маршруты

Клиентские маршруты должны поддерживать локали `ru` и `en`. Конкретная техническая схема локализованных URL может использовать locale prefix, middleware или эквивалентный механизм выбранного framework, но должна обеспечивать следующие обязательные правила:

- `ru` является локалью по умолчанию;
- переключение языка сохраняет текущую клиентскую страницу или ведет на ее локализованный эквивалент;
- локаль сохраняется при login/register и восстановлении незавершенной заявки;
- отсутствие или некорректное значение локали безопасно возвращает клиентский интерфейс к `ru`;
- admin routes не требуют английской локализации и используют русский интерфейс.

Публичные routes:

- `/`
- `/plans`
- `/order-request`
- `/order-request/success`

Customer routes:

- `/auth/login`
- `/auth/register`
- `/account`
- `/account/order-requests`
- `/account/order-requests/[id]`
- `/account/order-requests/[id]/payment`

Admin routes:

- `/admin/login`
- `/admin`
- `/admin/order-requests`
- `/admin/order-requests/[id]`

Server/API routes:

- `POST /api/order-requests`
- `GET /api/account/order-requests`
- `GET /api/account/order-requests/[id]`
- `POST /api/account/order-requests/[id]/demo-payment`
- `GET /api/admin/order-requests`
- `GET /api/admin/order-requests/[id]`
- `PATCH /api/admin/order-requests/[id]`

Если финальный framework использует server actions вместо API routes, те же access rules и data contracts остаются обязательными.

## 5. Авторизация

Customer и admin authentication используют Supabase Auth.

Правила:

- клиенты могут просматривать публичные страницы и начинать заполнение заявки без login;
- клиенты должны login или register перед финальной отправкой заявки;
- authenticated customers могут открывать `/account`;
- authenticated customers могут читать только свои заявки;
- только authenticated Mealora admin users могут открывать `/admin`;
- unauthenticated users на admin routes должны перенаправляться на `/admin/login`;
- unauthenticated users на account routes должны перенаправляться на `/auth/login`;
- public users не должны читать записи заявок напрямую из базы данных;
- admin users и customer users должны различаться через role или profile metadata;
- customer profile должен создаваться автоматически при регистрации customer;
- admin role не назначается через публичный UI и не может быть выбрана пользователем;
- admin role назначается только вручную владельцем проекта или Supabase admin;
- admin sessions должны проверяться server-side перед возвратом admin data.

## 6. Модель базы данных

### 6.1 `profiles`

Хранит application-level profile и role data для Supabase Auth users.

Обязательные поля:

- `id` UUID primary key, ссылается на Supabase Auth user id;
- `created_at` timestamp;
- `updated_at` timestamp;
- `role` user role.

Допустимые значения `role`:

- `customer`
- `admin`

Customer users могут открывать только customer account pages. Admin users могут открывать admin pages.

Создание и назначение roles:

- при регистрации нового customer система автоматически создает запись `profiles` с role `customer`;
- если `profiles` запись не создана автоматически, server-side signup flow должен создать ее до финальной отправки заявки;
- role `admin` назначается только вручную владельцем проекта или Supabase admin;
- публичный register/login UI не должен давать пользователю выбор роли;
- customer не может изменить свою role через клиентский интерфейс или публичный API.

### 6.2 `order_requests`

Хранит отправленные публичные заявки.

Обязательные поля:

- `id` UUID primary key;
- `user_id` UUID, ссылается на customer profile / Supabase Auth user id;
- `created_at` timestamp;
- `updated_at` timestamp;
- `status` статус заявки;
- `payment_status` статус payment lifecycle;
- `plan_type`;
- `meal_format`;
- `people_count`;
- `cycle_price`;
- `customer_name`;
- `customer_whatsapp`;
- `delivery_city_or_zone`;
- `delivery_address`;
- `high_risk_answer`;
- `high_risk_confirmed_no`;

Необязательные поля:

- `food_preferences`;
- `excluded_foods`;
- `mild_intolerances`;
- `spice_level`;
- `additional_notes`;
- `preferred_start_date`;
- `admin_notes`;

### 6.3 `payment_attempts`

Хранит demo payment attempts для заявок, которые Mealora одобрила и перевела в `awaiting_payment`.

Обязательные поля:

- `id` UUID primary key;
- `order_request_id` UUID;
- `user_id` UUID;
- `created_at` timestamp;
- `updated_at` timestamp;
- `status`;
- `amount`;
- `currency`;
- `provider`;

Для v1 demo payment поле `provider` должно быть `demo`.

Manual agreed payment не создает запись в `payment_attempts`. Если оплата согласована вручную вне сайта, admin обновляет `order_requests.payment_status` на `paid` и фиксирует пояснение в `order_requests.admin_notes`.

Допустимые значения `status`:

- `processing`
- `paid`
- `failed`
- `cancelled`

### 6.4 `price_options`

Хранит цены для доступных комбинаций плана, формата питания, количества человек и стандартного недельного цикла.

Обязательные поля:

- `id` UUID primary key;
- `created_at` timestamp;
- `updated_at` timestamp;
- `plan_type`;
- `meal_format`;
- `people_count`;
- `cycle_days`;
- `amount`;
- `currency`;
- `active`;

Правила:

- `plan_type` должен соответствовать одному из активных планов v1;
- `meal_format` должен соответствовать одному из форматов питания v1;
- `people_count` должен быть от 1 до 5;
- `cycle_days` для v1 должен быть 5;
- `amount` должен быть больше 0;
- `currency` для Mealora v1 должна быть `JOD`;
- `active = true` означает, что цена доступна для публичного показа и отправки заявки;
- `active = false` означает, что комбинация временно недоступна и публичная форма должна блокировать отправку.

Заявка не должна хранить ссылку на цену как единственный источник правды. При отправке заявки server должен взять valid active price из `price_options` и сохранить snapshot суммы в `order_requests.cycle_price`.

### 6.5 Значения статуса заявки

Допустимые значения `status`:

- `submitted`
- `under_review`
- `awaiting_payment`
- `confirmed`
- `rejected`
- `cancelled`

Default status для успешно отправленной публичной формы: `submitted`.

Request status transition rules:

- `submitted` может перейти в `under_review`, `awaiting_payment`, `rejected` или `cancelled`;
- `under_review` может перейти в `awaiting_payment`, `rejected` или `cancelled`;
- `awaiting_payment` может перейти в `under_review`, `confirmed`, `rejected` или `cancelled` только с учетом payment rules ниже;
- переход `awaiting_payment -> under_review` разрешен только до успешной или вручную согласованной оплаты; при этом `payment_status` должен вернуться в `not_required`;
- переход `awaiting_payment -> confirmed` разрешен только при `payment_status = paid` после ручной проверки оплаты и деталей заявки администратором;
- переход `awaiting_payment -> rejected` или `awaiting_payment -> cancelled` разрешен стандартным workflow только до `payment_status = paid`;
- если `payment_status = paid`, стандартный следующий переход статуса заявки — только `confirmed`;
- исключительное отклонение или отмена после `payment_status = paid` не поддерживается стандартным workflow v1, не выполняется через обычный selector статуса и требует отдельного ручного решения владельца проекта с обязательной внутренней заметкой; автоматический refund в v1 отсутствует;
- `confirmed`, `rejected` и `cancelled` являются терминальными статусами и не могут переходить в другие статусы через стандартный workflow v1;
- изменение, продление или отмена после `confirmed` относится к следующему недельному циклу и не возвращает текущую заявку из `confirmed`;
- для повторной работы после `rejected`, `cancelled` или `confirmed` создается новая заявка;
- переход в тот же статус не считается изменением и не должен создавать ложное событие обновления;
- server-side endpoint обязан проверять допустимость каждого перехода; клиентский или admin UI не является источником истины для разрешения перехода.

### 6.6 Значения payment status

Допустимые значения `payment_status`:

- `not_required`
- `awaiting_payment`
- `processing`
- `paid`
- `failed`
- `cancelled`

Default payment status для успешно отправленной публичной формы: `not_required`.

Payment status transition rules:

- новая заявка создается с `payment_status = not_required`;
- когда admin переводит заявку в `awaiting_payment`, `payment_status` становится `awaiting_payment`;
- когда customer начинает demo payment, `payment_status` может временно стать `processing`;
- успешная demo payment переводит `payment_status` в `paid`;
- failed demo payment переводит `payment_status` в `failed`;
- после `failed` customer может повторить demo payment, если request status все еще `awaiting_payment`;
- если customer отменяет demo payment до завершения, `payment_status` становится `cancelled`;
- после `cancelled` customer может начать demo payment заново, если request status все еще `awaiting_payment`;
- если admin переводит заявку в `rejected` до оплаты, `payment_status` становится `not_required`;
- если admin переводит заявку в `cancelled` до оплаты, `payment_status` становится `cancelled`;
- если заявка уже имеет `payment_status = paid`, admin не должен переводить payment status назад в `not_required`, `failed` или `cancelled` без отдельного ручного решения и внутренней заметки.

### 6.7 Validation Rules

Server должен отклонить заявку, если:

- user не authenticated перед финальной отправкой;
- отсутствуют обязательные поля;
- `people_count` меньше 1 или больше 5;
- `high_risk_answer` равно `yes`;
- `high_risk_confirmed_no` не равно true;
- `cycle_price` отсутствует;
- active price не найден в `price_options`;
- выбранный plan, format или people count недействителен;
- delivery zone находится вне поддерживаемой зоны доставки Zarqa.

Если price невозможно рассчитать или найти, публичная форма не должна создавать заявку.

## 7. Flow публичной отправки заявки

1. Customer заполняет форму заявки.
2. Client-side validation ловит очевидно пропущенные поля.
3. Если customer не authenticated, приложение временно сохраняет введенные данные формы client-side.
4. После временного сохранения данных приложение перенаправляет customer на `/auth/login` или `/auth/register` перед финальной отправкой.
5. После successful authentication приложение возвращает customer к заявке и восстанавливает временно сохраненные данные формы.
6. Customer подтверждает финальную отправку восстановленной заявки.
7. После authentication server-side validation повторяет все critical checks.
8. Server рассчитывает или проверяет `cycle_price`.
9. Server создает новую запись `order_requests` со статусом `submitted`, payment status `not_required` и `user_id` authenticated customer.
10. После успешного создания заявки приложение очищает временно сохраненные client-side данные формы.
11. User перенаправляется на `/order-request/success`.
12. Customer видит отправленную заявку в `/account/order-requests`.
13. Admin видит отправленную заявку в `/admin/order-requests`.

Публичная форма создает заявку, а не финальный заказ.

Техническое правило создания заявки:

- заявка создается только server-side через server endpoint или server action;
- `user_id` берется из authenticated session, а не из client-submitted payload;
- client не может передать или подменить `user_id`;
- service role key не используется в браузере и не передается клиенту;
- если используется service role key на server-side, endpoint обязан сначала проверить authenticated session и ownership data.

Техническое правило временного хранения формы до login/register:

- незавершенная заявка до authentication хранится только client-side, например в `sessionStorage`;
- временно сохраненные данные формы не считаются доверенными данными;
- после login/register восстановленные данные должны снова пройти client-side validation и обязательную server-side validation;
- `cycle_price`, `user_id`, `status`, `payment_status` и admin-managed fields не берутся из временного client-side хранения как источник правды;
- после успешного создания заявки временно сохраненные данные должны быть очищены;
- если временно сохраненные данные повреждены, устарели или не проходят validation, customer должен вернуться к форме и исправить данные.

## 8. Flow личного кабинета клиента и demo payment

1. Customer входит через `/auth/login` или регистрируется через `/auth/register`.
2. Customer открывает `/account`.
3. Customer открывает `/account/order-requests`.
4. Customer открывает детальную страницу одной заявки.
5. Customer видит status, payment status, отправленные данные, price и next step.
6. Если Mealora еще не перевела заявку в `awaiting_payment`, payment недоступен.
7. Если admin одобряет заявку, admin устанавливает request status `awaiting_payment`, а payment status становится `awaiting_payment`.
8. Customer открывает `/account/order-requests/[id]/payment`.
9. Customer выполняет demo payment.
10. Server создает запись `payment_attempts` с provider `demo`.
11. Во время обработки demo payment payment status может стать `processing`.
12. Успешная demo payment устанавливает payment status `paid`.
13. Failed demo payment устанавливает payment status `failed` и дает customer возможность повторить demo payment, если заявка все еще `awaiting_payment`.
14. Cancelled demo payment устанавливает payment status `cancelled` и дает customer возможность начать demo payment заново, если заявка все еще `awaiting_payment`.
15. Успешная demo payment не меняет request status автоматически.
16. Заявка остается в `awaiting_payment`, пока admin не проверит payment и детали заявки.
17. Admin вручную меняет request status на `confirmed`, когда финальный заказ принят.

Demo payment не должен списывать реальные деньги.

## 9. Flow админ-зоны

1. Admin входит через `/admin/login`.
2. Admin открывает `/admin`.
3. Admin видит короткий dashboard summary.
4. Admin открывает `/admin/order-requests`.
5. Admin открывает детальную страницу заявки.
6. Admin проверяет customer details, plan, delivery data, price и preferences.
7. Admin меняет status при необходимости: `under_review`, `awaiting_payment`, `confirmed`, `rejected` или `cancelled`.
8. Admin добавляет или обновляет internal notes.
9. Admin может вручную связаться с customer через WhatsApp вне сайта.
10. Если оплата выполнена через demo payment, server фиксирует payment attempt и устанавливает `payment_status = paid`.
11. Если оплата вручную согласована вне сайта, admin устанавливает `payment_status = paid` без создания `payment_attempts` и добавляет пояснение в internal notes.
12. Admin может перевести заявку в `confirmed` только после approval Mealora и `payment_status = paid`.

Обновление `payment_status` в админ-зоне не считается real payment processing. Это только ручная фиксация состояния оплаты для demo payment или manual agreed payment.

Internal notes никогда не видны customers.

## 10. Правила авторизации

Public users могут:

- просматривать публичные страницы;
- начинать заполнение формы заявки до login.

Public users не могут:

- финально отправлять заявку без login;
- просматривать список заявок;
- читать отдельные записи заявок;
- обновлять заявки;
- читать admin notes;
- менять statuses.

Authenticated customer users могут:

- отправлять valid заявки для себя;
- просматривать только свои заявки;
- читать только детали своих заявок;
- выполнять demo payment только для своих заявок в статусе `awaiting_payment`.

Authenticated customer users не могут:

- читать заявки других customers;
- читать admin notes;
- обновлять admin-managed fields;
- самостоятельно редактировать отправленные заявки или confirmed orders;
- напрямую менять request status;
- выполнять demo payment до approval.

Admin users могут:

- просматривать список заявок;
- читать детали заявки;
- обновлять status;
- обновлять payment status при необходимости;
- обновлять internal notes.

Запросы на изменения, продление или отмену обрабатываются вручную через WhatsApp вне сайта. Admins фиксируют результат через request status и internal notes.

В v1 выполненная demo payment и вручную согласованная оплата обе фиксируются технически как `payment_status = paid`. Отдельного payment status для manual agreed payment в v1 нет. Manual agreed payment не создает `payment_attempts`; admin обязан зафиксировать пояснение в internal notes.

Admin users не могут:

- обходить server-side validation;
- раскрывать private data на публичных страницах;
- делать заказ финальным без approval Mealora и `payment_status = paid`.

## 11. Row-Level Security

Supabase RLS должен быть включен для application tables:

- `profiles`;
- `order_requests`;
- `payment_attempts`;
- `price_options`.

Required policy intent for `profiles`:

- public users не могут читать profiles;
- authenticated customers могут читать только свой profile;
- authenticated customers не могут менять свой `role`;
- authenticated admins могут читать profiles, если это нужно для admin workflow;
- назначение или изменение `role = admin` выполняется только владельцем проекта, Supabase admin или controlled server-side operation;
- public register/login UI не может назначать admin role.

Required policy intent for `order_requests`:

- public users не могут напрямую select, update или delete заявки;
- вставка заявки должна происходить только через controlled server endpoint или server action со строгой validation;
- `user_id` для новой заявки должен браться из authenticated session;
- client-submitted `user_id` должен игнорироваться или отклоняться;
- authenticated customers могут select только заявки, где `user_id` совпадает с authenticated user id;
- authenticated customers не могут обновлять admin-managed fields: `status`, `payment_status` или `admin_notes`;
- authenticated admin users могут select заявки;
- authenticated admin users могут обновлять admin-managed fields: `status`, `payment_status` и `admin_notes`;
- customers не могут delete заявки.

Required policy intent for `payment_attempts`:

- public users не могут читать, создавать, обновлять или удалять payment attempts;
- authenticated customers могут читать только свои payment attempts;
- demo payment attempt создается только через controlled server endpoint или server action для заявки этого customer в статусе `awaiting_payment`;
- `user_id` и `order_request_id` для payment attempt должны проверяться server-side;
- authenticated customers не могут создавать payment attempts для чужих заявок;
- authenticated customers не могут вручную менять payment attempt status;
- authenticated admins могут читать payment attempts для admin workflow;
- authenticated admins не должны использовать `payment_attempts` для manual agreed payment; manual agreed payment фиксируется через `order_requests.payment_status` и `admin_notes`.

Required policy intent for `price_options`:

- public users и authenticated customers могут читать только active prices, которые нужны для публичного отображения цены;
- public users и authenticated customers не могут создавать, обновлять или удалять price options;
- authenticated admins могут читать price options;
- изменение price options выполняется только через admin-controlled или owner-controlled operation;
- inactive price options не должны использоваться для создания заявки;
- server-side validation должна проверять active price перед созданием `order_requests`.

Global RLS/security rules:

- service role keys должны использоваться только server-side.
- client-side код не должен получать privileged database keys.
- server endpoints или server actions не должны обходить ownership checks даже при использовании service role key.

## 12. Работа с ценами

Prices хранятся в таблице `price_options` и отображаются на публичном сайте.

Публичная форма должна блокировать отправку, если valid price не найден для:

- plan type;
- meal format;
- people count;
- standard 5-day cycle.

Valid price означает активную запись `price_options` с `active = true`, которая совпадает с выбранными `plan_type`, `meal_format`, `people_count`, `cycle_days = 5` и `currency = JOD`.

Server должен рассчитывать или проверять `cycle_price` только на основе `price_options`. Client-submitted price нельзя принимать как источник правды.

Database record в `order_requests` должен хранить snapshot отправленного `cycle_price`, чтобы customer и admin видели точную цену, зафиксированную при отправке.

## 13. Состояния ошибок

Public form:

- пропущено обязательное поле;
- unauthenticated final submission;
- invalid people count;
- high-risk answer `yes`;
- unsupported delivery zone;
- price unavailable;
- server validation failure;
- database insert failure.

Все клиентские validation, error, success, empty и refusal сообщения должны возвращаться или отображаться на активной локали `ru` или `en`. Сервер не должен отдавать клиенту только внутренний технический текст ошибки. Для известных error codes клиентский слой должен иметь локализованные сообщения; неизвестная ошибка должна использовать локализованный безопасный fallback.

Customer account:

- unauthenticated access;
- expired session;
- order request not found;
- попытка открыть заявку другого customer;
- payment not yet available;
- demo payment failure.

Admin zone:

- unauthenticated access;
- expired session;
- request not found;
- status update failure;
- admin notes save failure;
- empty request list.

Все errors должны использовать спокойный, non-alarming copy.

## 14. Требования безопасности

- Admin routes должны быть защищены server-side.
- Account routes должны быть защищены server-side.
- Admin data не должны попадать в public pages bundle.
- Internal notes никогда не должны возвращаться public clients.
- Public form submissions должны быть rate-limited или защищены от очевидного spam перед production.
- Secrets и Supabase service keys должны храниться в environment variables.
- Client-side validation недостаточно; server-side validation обязательна.
- Demo payment не должен использовать real payment credentials или списывать реальные деньги.

## 15. Критерии приемки

Technical implementation считается acceptable, если:

- клиентский интерфейс полностью доступен на `ru` и `en`, при этом `ru` используется по умолчанию;
- выбор клиентской локали сохраняется при навигации, login/register и восстановлении формы;
- клиентские validation, error, success, empty и refusal состояния отображаются на выбранной локали;
- неизвестная серверная ошибка отображается через локализованный безопасный fallback без раскрытия внутренних деталей;
- админ-зона v1 использует русский язык и не требует английской локализации;
- unauthenticated users могут просматривать публичные страницы, но не могут финально отправить заявку;
- authenticated customers могут отправить valid заявку;
- valid заявка создает одну database record, связанную с customer `user_id`;
- invalid или high-risk submissions не создают records;
- missing price блокирует отправку;
- valid price берется из active `price_options` с `currency = JOD`;
- client-submitted price не используется как источник правды;
- `order_requests.cycle_price` сохраняет snapshot цены на момент отправки заявки;
- если unauthenticated customer начал заявку, данные формы временно сохраняются client-side и восстанавливаются после login/register;
- временно сохраненные client-side данные не обходят server-side validation;
- после успешного создания заявки временно сохраненные данные формы очищаются;
- отправленные заявки появляются в customer account и admin request list со статусом `submitted`;
- customers могут читать только свои заявки;
- customers не могут читать admin notes;
- demo payment доступна только после admin approval;
- demo payment создает `payment_attempts` только для заявки authenticated customer в статусе `awaiting_payment`;
- customers не могут создавать payment attempts для чужих заявок;
- успешная demo payment обновляет payment status на `paid`;
- успешная demo payment не устанавливает request status `confirmed` автоматически;
- manual agreed payment не создает `payment_attempts`;
- manual agreed payment фиксируется через `payment_status = paid` и пояснение в `admin_notes`;
- unauthenticated users не могут открывать admin pages;
- unauthenticated users не могут открывать account pages;
- admin может открыть детали заявки;
- admin может изменить status;
- admin может выбрать только допустимый следующий status согласно матрице переходов из раздела 6.5;
- admin может вручную перевести eligible заявку с `payment_status = paid` в `confirmed`;
- admin может сохранять internal notes;
- customers не видят internal notes;
- RLS включен для `profiles`, `order_requests`, `payment_attempts` и `price_options`;
- customers могут читать только свои `profiles`, `order_requests` и `payment_attempts`;
- customers не могут менять role, admin-managed fields или price options;
- public users и customers могут читать только active price options для публичного отображения цены;
- service role keys используются только server-side и не обходят ownership checks;
- final order confirmation требует approval Mealora, `payment_status = paid` и ручного перевода заявки в `confirmed`.

## 16. Статус одобрения

Technical Spec проверена владельцем проекта и имеет статус `checked-and-approved`.
