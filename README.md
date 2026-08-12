# PayFlow — Digital Banking & Payments Platform

FastAPI + MySQL backend, vanilla HTML/JS frontend. UPI/bill/recharge/wallet
payments, AI fraud scoring, KYC, operator/admin panel, and a self-service
**Add Money** flow.

## 1. Backend setup

```bash
cd backend
cp .env.example .env      # then edit MYSQL_URL credentials
pip install -r requirements.txt --break-system-packages
python ml/train_fraud_model.py     # generates the fraud detection model
uvicorn app.main:app --reload --port 8000
```

- API docs: http://localhost:8000/docs
- The frontend is also served directly at http://localhost:8000/app (see
  `StaticFiles` mount in `app/main.py`) — you don't have to run a separate
  static server if you don't want to.

MySQL must be running and reachable at whatever `MYSQL_URL` you set in
`.env`. Tables are auto-created on startup (`Base.metadata.create_all`) —
no manual migration step needed.

## 2. Frontend setup

If you're serving the frontend separately (e.g. a static file server on a
different port instead of using the `/app` mount above), point it at your
backend in `frontend/js/config.js`:

```js
window.PAYFLOW_API_BASE = "http://localhost:8000";
```

**This must match whatever port you actually run uvicorn on.** If it
doesn't match, every API call from the UI will silently fail (buttons open
modals fine since that's pure DOM, but nothing that talks to the backend —
login, balances, history, Add Money, etc. — will work). This was previously
mismatched (defaulted to `8001` while the documented run command uses
`8000`) — verified fixed in this build.

## 3. First-time login

There's no public sign-up for admin/operator roles. On first run:
1. Open the app, go to the login screen — a **"Set up the admin account"**
   link appears automatically when no admin/operator exists yet
   (`GET /api/auth/setup-status`).
2. Fill it in once via `POST /api/auth/setup-admin`. This endpoint 403s on
   every call after the first admin exists, so it's safe to leave in place.
3. Customers self-register normally via **Create an account**.

## 4. Add Money (self-service, admin-approved)

A customer can request funds be added to their own account; nothing is
credited until an operator/admin approves it.

**Customer side** (`dashboard.html` / `js/app.js`):
- "Add Money" in the sidebar, Quick Actions, and topbar all open a modal
  → `POST /api/payments/balance-requests` with `{ amount, note }`.
- A "My Add Money Requests" table on the dashboard shows each request's
  status (`pending` / `approved` / `rejected`) and any admin remarks
  → `GET /api/payments/balance-requests`.

**Admin side** (`admin.html` / `js/admin.js`):
- "Balance Requests" tab lists everything pending
  → `GET /api/admin/balance-requests/pending`.
- Approve/Reject buttons → `PATCH /api/admin/balance-requests/{id}/decision`.
  Approving credits `Account.balance` and logs a `Transaction` (ref
  prefixed `ADDMNY-`); rejecting just closes the request out.

This was **verified end-to-end** against a live instance of this exact
backend: register → login → submit request → admin sees it in the pending
queue → admin approves → customer's balance updates from ₹0 to the
requested amount → transaction shows up in history with an `ADDMNY-` ref.

There's also a separate, pre-existing **instant** admin action — "Add
Balance" on the Users tab (`POST /api/admin/users/{user_id}/balance`) —
that credits/debits immediately with no approval step, for manual
corrections/refunds. Both flows coexist; use whichever fits.

## Project layout

```
backend/
  app/
    core/       auth deps, password/JWT helpers
    models/     SQLAlchemy models (User, Account, Transaction, BalanceRequest, ...)
    routers/    FastAPI routers (one per microservice-style concern)
    schemas/    Pydantic request/response models
    services/   business logic (payment processing, fraud scoring, cache, notifications)
  ml/           trained fraud-detection model artifacts + training script
frontend/
  index.html      login / register / one-time admin setup
  dashboard.html  customer dashboard (Send Money, Add Money, KYC, history)
  admin.html      operator/admin panel (users, KYC, balance requests, fraud, disputes, reports)
  js/             config.js (API base URL), api.js (fetch wrapper), app.js, admin.js
  css/style.css
```
