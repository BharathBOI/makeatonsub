# SPEC.md: Subscription Management System (Brightloop)

> Single source of truth. Every agent prompt and every PR references this file.
> If code and spec disagree, fix one of them in the same PR. Items marked **TODO** need a team decision.

---

## 0. Team and ownership

| Owner | Modules | Backend folder | Frontend folder |
|---|---|---|---|
| A | Auth, RBAC, users, app shell, deploy | `backend/modules/auth` | `frontend/features/auth` |
| B | Products, variants, plans, quotation templates, subscriptions | `backend/modules/catalog`, `backend/modules/subscriptions` | `frontend/features/catalog`, `frontend/features/subscriptions` |
| C | Invoices, taxes, discounts, billing job | `backend/modules/billing` | `frontend/features/billing` |
| D | Payments, customer portal, dashboard, seed data | `backend/modules/payments`, `backend/modules/reports` | `frontend/features/payments`, `frontend/features/portal`, `frontend/features/dashboard` |

**Rule:** only edit files in your own modules. Shared files (schema/migrations, `package.json`, route index, app shell) go through owner A via PR.

---

## 1. Tech stack  (**TODO: confirm**)

- Frontend: React / Next.js + component library (shadcn/ui or MUI)
- Backend: Node (NestJS/Express) or Django/FastAPI
- DB: PostgreSQL
- Jobs/scheduler: **TODO** (BullMQ / Celery / node-cron)
- Email: **TODO** (Resend / SendGrid / Nodemailer)
- Auth: **TODO** (JWT or session cookie), password hashing with argon2 or bcrypt
- Deploy: **TODO**

---

## 2. Global conventions

1. **Money** is stored as integer minor units (paise). Never floats. Currency: **TODO** (INR?).
2. **Dates**: stored in UTC; billing dates are date-only. Month-end rule: **TODO** (e.g. a subscription started on the 31st bills on the last day of shorter months).
3. **IDs**: UUID or bigint, **TODO**.
4. **Error format** (all APIs):
   ```json
   { "error": { "code": "VALIDATION_ERROR", "message": "...", "fields": { "email": "already in use" } } }
   ```
5. **Pagination** on every list endpoint: `?page=1&pageSize=25`, response includes `total`.
6. **Timestamps** on every table: `created_at`, `updated_at`.
7. **Soft delete** vs hard delete: **TODO**.
8. All authorization is enforced **server-side**. The UI hiding a button is not security.
9. Never commit secrets. Use `.env` (see `.env.example`).

---

## 3. Roles and permissions

| Capability | Admin | Staff | Customer |
|---|---|---|---|
| Create staff users | Yes | No | No |
| Manage products, variants, plans | Yes | Read (**TODO**: edit?) | Read active plans |
| Manage taxes | Yes | No | No |
| Create/edit discount rules | Yes | No | No |
| Create quotations/subscriptions | Yes | Yes | No |
| Change subscription status | Yes | Yes (**TODO**: which transitions) | Own only (**TODO**: cancel/pause?) |
| Generate / confirm / cancel / send invoices | Yes | Yes | No |
| Record payments | Yes | Yes | No |
| View finance dashboard | Yes | Yes (**TODO**) | No |
| View own subscriptions and invoices | n/a | n/a | **Own only** |

**Hard rule:** a customer must never be able to read another customer's subscription, invoice or payment, even by changing an ID in the URL. Add an automated test for this.

---

## 4. Auth and users (Owner A)

### Rules
- Signup with email and password. **Email is unique** (case-insensitive).
- Password: **more than 8 characters** (so 9+), at least one uppercase, one lowercase, one special character. Validate on the server.
- Self-signup always creates a `customer` role user and a linked `Customer` record.
- Only an admin can create `staff` users.
- Forgot password: emailed reset link, single-use token, expiry **TODO** (e.g. 30 min). Do not reveal whether an email exists.
- Login rate limiting: **TODO**.

### Endpoints
| Method | Path | Access | Notes |
|---|---|---|---|
| POST | `/auth/signup` | public | |
| POST | `/auth/login` | public | |
| POST | `/auth/logout` | auth | |
| POST | `/auth/forgot-password` | public | sends reset email |
| POST | `/auth/reset-password` | public | token + new password |
| GET | `/auth/me` | auth | |
| POST | `/users/staff` | admin | create staff |
| GET | `/users` | admin | list |

---

## 5. Catalog: products, variants, plans (Owner B)

### Product
`id, name, type, sales_price, cost_price, notes, is_active`

### Variant
`id, product_id, attribute (e.g. "brand"), value (e.g. "Odoo"), extra_price`
Example: brand = Odoo adds 560 to the product price. Effective unit price = product sales price + variant extra price.

### Plan
`id, name, price, billing_period (daily|weekly|monthly|yearly), min_quantity, available_from, available_to, auto_close_at_end (bool), closable_early (bool), pausable (bool), renewable (bool), is_active`

Admin decides the four flags plan by plan.
- `auto_close_at_end`: the subscription closes by itself on its end date.
- `closable_early`: the subscription can be closed before the end date.
- `pausable`: the subscription can be paused and resumed. **TODO**: does billing stop while paused?
- `renewable`: the subscription can be renewed at the end.

### Quotation template
`id, name, validity_days, plan_id` + lines `(product_id, variant_id?, quantity)`

### Endpoints  (**TODO: fill request/response shapes in docs/api-contract.md**)
`/products`, `/products/:id/variants`, `/plans`, `/quotation-templates` with standard CRUD. Writes are admin only.

---

## 6. Subscriptions (Owner B)

### Fields
`id, number (unique, auto-generated e.g. SUB-000123), customer_id, plan_id, start_date, expiry_date, payment_terms, status, next_billing_date, created_by`

### Lines
`id, subscription_id, product_id, variant_id?, quantity, unit_price, tax_id?, amount`

### Creating from a quotation template
Copies the plan and lines from the template. The quotation is valid for `validity_days` from creation. Expired quotations: **TODO** (auto-cancel? block confirm?).

### State machine
```
Draft -> Quotation -> Confirmed -> Active -> Closed
```
Allowed transitions (everything else is rejected with a 409):

| From | To | Who | Notes |
|---|---|---|---|
| Draft | Quotation | Admin, Staff | |
| Quotation | Confirmed | Admin, Staff | check quotation not expired |
| Confirmed | Active | Admin, Staff / system | **TODO**: manual or automatic on start date? |
| Active | Closed | Admin, Staff / system | by plan flags (auto-close, early close) |
| Quotation | Closed | Admin, Staff | cancel before confirm (**TODO**) |

Paused/renewed behaviour: **TODO** (separate status vs flag).
Record every transition in `subscription_status_history (subscription_id, from, to, changed_by, changed_at)`.

---

## 7. Taxes (Owner C)

`Tax: id, name, percentage, applies_to (all | product category | product), is_active`

- Calculated automatically from admin-defined rules.
- **Snapshot the rate on the invoice line** at generation time, so later rule changes never alter old invoices.
- Calculation order: **TODO** (discount first, then tax on the discounted amount is the usual choice).
- Rounding: round per line to the nearest paisa, **TODO** confirm.

---

## 8. Discounts (Owner C)

`Discount: id, name, type (fixed|percentage), value, min_purchase_amount?, min_quantity?, start_date, end_date, usage_limit, usage_count, scope (product|subscription), product_ids[]?, is_active`

Rules:
- Only admin can create/edit.
- Applies to specific products or to whole subscriptions.
- Applied **automatically** on the next invoice if conditions are met: within dates, min amount/quantity met, `usage_count < usage_limit`.
- Stops when expired or exhausted.
- `usage_count` is incremented in the **same DB transaction** as invoice creation.
- Usage counts per invoice, not per subscription lifetime. **TODO**: confirm.
- Multiple discounts on one invoice: **TODO** (stack, or best one only).
- Fixed discount cannot make a line negative.

Implementation note: write the discount logic as a pure function, e.g. `applyDiscounts(subscription, lines, today) -> { lines, usedDiscountIds }`, so it is unit-testable.

---

## 9. Invoicing and billing job (Owner C)

### Invoice fields
`id, number (unique), subscription_id, customer_id, billing_period_start, billing_period_end, issue_date, due_date, status, subtotal, discount_total, tax_total, total, amount_paid`

### Invoice lines
`id, invoice_id, product_id, description, quantity, unit_price, discount_amount, tax_rate_snapshot, tax_amount, amount`

### State machine
```
Draft -> Confirmed -> Paid
```
Plus a `Cancelled` state reachable from Draft/Confirmed (**TODO**: confirm). Actions: confirm, cancel, send (email), print (PDF).

### Billing job (the critical part)
- Runs on a schedule (**TODO**: daily at what time) and via a manual "Run billing now" admin button.
- Selects subscriptions with `status = Active AND next_billing_date <= today`.
- For each, in **one transaction**: create the invoice with lines, tax and discounts, then advance `next_billing_date` by the plan's period.
- **Idempotency:** unique constraint on `(subscription_id, billing_period_start)`. Running the job twice must create zero duplicates.
- A failure on one subscription must not block the others. Log failures and retry.
- **Catch-up:** if the job missed days, bill every missed period, none skipped. **TODO**: confirm behaviour.
- `due_date = issue_date + payment_terms` (**TODO**: define payment terms values, e.g. Immediate, Net 15, Net 30).

### Acceptance tests
- [ ] Run billing twice, get exactly one invoice per due subscription.
- [ ] Simulate a crash mid-run, re-run and get no duplicates and no skipped subscriptions.
- [ ] Discount usage count increments exactly once per use.

---

## 10. Payments (Owner D)

`Payment: id, invoice_id, method (cash|card|bank_transfer|upi|other **TODO**), amount, payment_date, note, recorded_by`

- Recorded against an invoice.
- Partial payments allowed (**TODO** confirm). Sum of payments per invoice drives `amount_paid`.
- When `amount_paid >= total`, the invoice moves to **Paid** automatically.
- Overpayment: **TODO**.
- **Overdue** = `due_date < today AND status != Paid` (computed, not stored).
- Payments are immutable once recorded; corrections are made via a reversal record. **TODO**.

---

## 11. Customer portal (Owner D)

Customer-facing pages (own data only):
- My subscriptions (status, plan, next billing date)
- My invoices (status, download PDF)
- Payment history
- Actions customers may take: **TODO** (cancel, pause, upgrade?)

---

## 12. Finance dashboard (Owner D)

Shows:
- Active subscription count
- Revenue (collected) for a selected period
- Outstanding amount
- Overdue invoices list
- Payments list
- MRR (**stretch**)

Filters: date range, customer, plan, status.

**Performance:** every page responds in **under 2 seconds** with thousands of subscriptions.
- Use SQL aggregates (`GROUP BY`, `SUM`, `COUNT`), never loop in app code.
- Index: `subscriptions(status, next_billing_date, customer_id)`, `invoices(status, due_date, customer_id)`, `payments(invoice_id, payment_date)`.
- Paginate all lists.
- Seed script creates 5,000+ subscriptions with invoices and payments; record page timings in `docs/perf.md`.

---

## 13. Database schema

> Maintained as migrations in `backend/migrations/`. Never edit a merged migration; add a new one. Schema PRs are announced in the group chat and merged by owner A in order.

Tables: `users, customers, products, product_variants, plans, quotation_templates, quotation_template_lines, subscriptions, subscription_lines, subscription_status_history, taxes, discounts, discount_products, invoices, invoice_lines, payments, password_reset_tokens`

(**TODO**: add ER diagram or paste final `schema.sql` here.)

---

## 14. Seed data

`npm run seed` (or equivalent) creates:
- 1 admin, 2 staff, 50+ customers (with portal logins)
- 5 products with variants, 6 plans across all four billing periods
- 3 taxes, 3 discounts (one expired, one exhausted, one active)
- 5,000+ subscriptions in mixed statuses
- Invoices in Draft, Confirmed, Paid, and overdue states, with partial and full payments

---

## 15. Demo script

1. Sign up as a customer, log out.
2. Admin creates a product with variants, a plan, a tax and a discount.
3. Staff creates a quotation from a template, then confirms it and it goes Active.
4. Admin clicks **Run billing**. An invoice appears with tax and discount applied.
5. Click **Run billing** again. Nothing new is created (idempotency).
6. Staff records a payment, and the invoice flips to Paid.
7. Dashboard updates: active subscriptions, revenue, outstanding, overdue.
8. Customer logs in and sees only their own subscription and invoice. Try another customer's invoice ID and get a 403/404.

---

## 16. Open questions log

| # | Question | Decision | Decided by / date |
|---|---|---|---|
| 1 | | | |

---

## 17. Definition of done (per PR)

- [ ] Matches this spec (or the spec is updated in the same PR)
- [ ] Server-side validation and role checks
- [ ] Money as integers
- [ ] Tests for the business rules touched
- [ ] Only files in my own modules changed
- [ ] No secrets committed
- [ ] Page/endpoint responds well within the 2 s budget on seeded data
