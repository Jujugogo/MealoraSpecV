# Mealora Technical Spec

Status: draft

## 1. Purpose

This Technical Spec defines the technical structure for Mealora v1.

It covers:

- public order request submission;
- customer authentication and account area;
- database storage for order requests;
- demo payment lifecycle;
- admin authentication;
- protected admin routes;
- admin order request management;
- basic security rules;
- technical edge cases.

This document supports:

- `BUSINESS_SPEC.md`
- `STYLE_UI_SPEC.md`
- `FUNCTIONAL_MAP_SPEC.md`

## 2. Current Scope

Mealora v1 includes:

- public marketing and order request pages;
- public order request form;
- customer login and registration;
- customer account dashboard;
- customer order request list and detail pages;
- demo payment page after approval;
- database-backed order request storage;
- admin login;
- admin dashboard;
- admin order request list;
- admin order request detail page;
- status updates;
- internal admin notes.

Mealora v1 does not include:

- automatic subscription billing;
- automatic order confirmation;
- menu management;
- recipe management;
- delivery tracking;
- payment processing inside the admin zone;
- CliQ payment processing;
- card payment processing;
- payment links;
- real money payment processing;
- repeat order in one click;
- loyalty program.

## 3. Technical Stack Decision

Use Supabase for:

- PostgreSQL database;
- customer authentication;
- admin authentication;
- row-level security;
- server-side order request storage.

The public website must not expose privileged database keys to the browser.

## 4. Routes

Public routes:

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

If the final framework uses server actions instead of API routes, the same access rules and data contracts still apply.

## 5. Authentication

Customer and admin authentication use Supabase Auth.

Rules:

- customers may browse public pages and start filling the order request form without login;
- customers must login or register before final order request submission;
- authenticated customers may access `/account`;
- authenticated customers may read only their own order requests;
- only authenticated Mealora admin users may access `/admin`;
- unauthenticated users visiting admin routes must be redirected to `/admin/login`;
- unauthenticated users visiting account routes must be redirected to `/auth/login`;
- public users must never read order request records directly from the database;
- admin users and customer users must be distinguishable by role or profile metadata;
- admin sessions must be checked server-side before returning admin data.

## 6. Database Model

### 6.1 `profiles`

Stores application-level user profile and role data for Supabase Auth users.

Required fields:

- `id` UUID primary key, references Supabase Auth user id;
- `created_at` timestamp;
- `updated_at` timestamp;
- `role` user role.

Allowed `role` values:

- `customer`
- `admin`

Customer users may access only customer account pages. Admin users may access admin pages.

### 6.2 `order_requests`

Stores public order request submissions.

Required fields:

- `id` UUID primary key;
- `user_id` UUID, references customer profile / Supabase Auth user id;
- `created_at` timestamp;
- `updated_at` timestamp;
- `status` order request status;
- `payment_status` payment lifecycle status;
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

Optional fields:

- `food_preferences`;
- `excluded_foods`;
- `mild_intolerances`;
- `spice_level`;
- `additional_notes`;
- `preferred_start_date`;
- `admin_notes`;

### 6.3 `payment_attempts`

Stores demo payment attempts for order requests approved by Mealora and moved to `awaiting_payment`.

Required fields:

- `id` UUID primary key;
- `order_request_id` UUID;
- `user_id` UUID;
- `created_at` timestamp;
- `updated_at` timestamp;
- `status`;
- `amount`;
- `currency`;
- `provider`;

For v1 demo payment, `provider` must be `demo`.

Allowed `status` values:

- `processing`
- `paid`
- `failed`
- `cancelled`

### 6.4 Order Request Status Values

Allowed `status` values:

- `submitted`
- `under_review`
- `awaiting_payment`
- `confirmed`
- `rejected`
- `cancelled`

Default status for a successfully submitted public form: `submitted`.

### 6.5 Payment Status Values

Allowed `payment_status` values:

- `not_required`
- `awaiting_payment`
- `processing`
- `paid`
- `failed`
- `cancelled`

Default payment status for a successfully submitted public form: `not_required`.

### 6.6 Validation Rules

The server must reject an order request if:

- the user is not authenticated before final submission;
- required fields are missing;
- `people_count` is less than 1 or greater than 5;
- `high_risk_answer` is `yes`;
- `high_risk_confirmed_no` is not true;
- `cycle_price` is missing;
- selected plan, format, or people count is invalid;
- delivery zone is outside the supported Zarqa delivery scope.

If price cannot be calculated or found, the public form must not create an order request.

## 7. Public Submission Flow

1. Customer fills the order request form.
2. Client-side validation catches obvious missing fields.
3. If the customer is not authenticated, the app redirects to `/auth/login` or `/auth/register` before final submission.
4. After authentication, server-side validation repeats all critical checks.
5. Server calculates or verifies `cycle_price`.
6. Server inserts a new `order_requests` record with status `submitted`, payment status `not_required`, and the authenticated customer's `user_id`.
7. User is redirected to `/order-request/success`.
8. Customer can see the submitted request in `/account/order-requests`.
9. Admin can see the submitted request in `/admin/order-requests`.

The public form creates an order request, not a final order.

## 8. Customer Account And Demo Payment Flow

1. Customer signs in through `/auth/login` or registers through `/auth/register`.
2. Customer opens `/account`.
3. Customer opens `/account/order-requests`.
4. Customer opens one order request detail page.
5. Customer can see status, payment status, submitted data, price, and next step.
6. If Mealora has not moved the request to `awaiting_payment`, payment is not available.
7. If admin approves the request, admin sets request status to `awaiting_payment` and payment status becomes `awaiting_payment`.
8. Customer opens `/account/order-requests/[id]/payment`.
9. Customer runs demo payment.
10. Server creates a `payment_attempts` record with provider `demo`.
11. Successful demo payment sets payment status to `paid`.
12. Successful demo payment does not automatically change the request status.
13. The request remains in `awaiting_payment` until an admin verifies payment and request details.
14. Admin manually changes request status to `confirmed` when the final order is accepted.

Demo payment must not charge real money.

## 9. Admin Flow

1. Admin signs in through `/admin/login`.
2. Admin opens `/admin`.
3. Admin sees a short dashboard summary.
4. Admin opens `/admin/order-requests`.
5. Admin opens a request detail page.
6. Admin reviews customer details, plan, delivery data, price, and preferences.
7. Admin changes status when needed: `under_review`, `awaiting_payment`, `confirmed`, `rejected`, or `cancelled`.
8. Admin adds or updates internal notes.
9. Admin may contact the customer manually through WhatsApp outside the site.
10. Admin can confirm the final order only after approval and completed or agreed payment.

Internal notes are never visible to customers.

## 10. Authorization Rules

Public users may:

- browse public pages;
- start filling the order request form before login.

Public users may not:

- submit the final order request without login;
- list order requests;
- read individual order request records;
- update order requests;
- read admin notes;
- change statuses.

Authenticated customer users may:

- submit valid order requests for themselves;
- list only their own order requests;
- read only their own order request details;
- run demo payment only for their own requests in `awaiting_payment` status.

Authenticated customer users may not:

- read other customers' requests;
- read admin notes;
- update admin-managed fields;
- self-edit submitted requests or confirmed orders;
- change request status directly;
- run demo payment before approval.

Admin users may:

- list order requests;
- read request details;
- update status;
- update payment status when needed;
- update internal notes.

Change, extension, or cancellation requests are handled manually through WhatsApp outside the site. Admins record the result through request status and internal notes.

Admin users may not:

- bypass server-side validation;
- expose private data to public pages;
- make an order final without Mealora approval and completed or agreed payment.

## 11. Row-Level Security

Supabase RLS must be enabled for order request tables.

Required policy intent:

- public users cannot directly select, update, or delete order requests;
- order request insertion must happen through a controlled server endpoint or server action with strict validation;
- authenticated customers can select only order requests where `user_id` matches their authenticated user id;
- authenticated customers cannot update admin-managed fields such as `status`, `payment_status`, or `admin_notes`;
- authenticated customers can create demo payment attempts only for their own requests in `awaiting_payment` status;
- authenticated customers can read only their own payment attempts;
- authenticated admin users can select order requests;
- authenticated admin users can update admin-managed fields such as `status`, `payment_status`, and `admin_notes`;
- service role keys must only be used server-side.

## 12. Price Handling

Prices are stored in the database and displayed on the public site.

The public form must block submission if a valid price cannot be found for:

- plan type;
- meal format;
- people count;
- standard 5-day cycle.

The database record must store the submitted `cycle_price` snapshot so the customer and admin see the exact price captured at submission.

## 13. Error States

Public form:

- missing required field;
- unauthenticated final submission;
- invalid people count;
- high-risk answer `yes`;
- unsupported delivery zone;
- price unavailable;
- server validation failure;
- database insert failure.

Customer account:

- unauthenticated access;
- expired session;
- order request not found;
- attempt to access another customer's request;
- payment not yet available;
- demo payment failure.

Admin zone:

- unauthenticated access;
- expired session;
- request not found;
- status update failure;
- admin notes save failure;
- empty request list.

All errors must use calm, non-alarming copy.

## 14. Security Requirements

- Admin routes must be protected server-side.
- Account routes must be protected server-side.
- Admin data must not be bundled into public pages.
- Internal notes must never be returned to public clients.
- Public form submissions must be rate-limited or protected against obvious spam before production.
- Secrets and Supabase service keys must stay in environment variables.
- Client-side validation is not enough; server-side validation is required.
- Demo payment must not use real payment credentials or charge real money.

## 15. Acceptance Criteria

Technical implementation is acceptable when:

- unauthenticated users can browse public pages but cannot final-submit a request;
- authenticated customers can submit a valid order request;
- a valid order request creates one database record linked to the customer `user_id`;
- invalid or high-risk submissions do not create records;
- missing price blocks submission;
- submitted requests appear in the customer account and admin request list with status `submitted`;
- customers can read only their own requests;
- customers cannot read admin notes;
- demo payment is available only after admin approval;
- successful demo payment updates payment status to `paid`;
- successful demo payment does not automatically set request status to `confirmed`;
- unauthenticated users cannot access admin pages;
- unauthenticated users cannot access account pages;
- admin can open request details;
- admin can change status;
- admin can manually set an eligible request with `payment_status = paid` to `confirmed`;
- admin can save internal notes;
- customers cannot see internal notes;
- final order confirmation requires Mealora approval and completed or agreed payment.

## 16. Approval Status

This Technical Spec is a draft until the project owner reviews and approves it.
