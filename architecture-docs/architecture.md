# Expanse Tracker — API Architecture

> Location: `expanse-tracker-api/architecture-docs/architecture.md`
> Companion: `expanse-tracker-web/architecture-docs/architecture.md` (Frontend)

---

## 1. Project Overview

### Purpose

Backend REST API for the **Expense Tracker** application. It exposes domain resources (expenses, categories) consumed by the Angular frontend and persists data in MongoDB. The current workstream is **API foundation only** — a bootstrapped Express server that is progressively built out.

### Technology Stack

| Concern | Choice | Notes |
| --- | --- | --- |
| Runtime | Node.js (CommonJS) | `"type": "commonjs"` |
| Web framework | Express 5 | `express` ^5.2.1 |
| Configuration | `dotenv` | Env vars loaded via `.env` |
| Dev runner | `nodemon` | Auto-restart in development |
| Persistence | MongoDB Atlas (cloud cluster) | Via Mongoose (to be installed); DB `expense_tracker_db` |
| Language | JavaScript | No TypeScript in this repo |

### Installed Dependencies

Runtime:

- `dotenv`, `express`

Dev:

- `nodemon`

---

## 2. Current Implementation Status

| Area | Status |
| --- | --- |
| Server bootstrap | **DONE** (`index.js`) |
| `.env` + port config | **DONE** (default `8000`) |
| Health/root route | **DONE** (`GET /`) |
| MongoDB connection | **Configured** (Atlas cluster URI in `.env`; Mongoose driver planned) |
| Route definitions/controllers | Planned |
| Validation & error handling | Planned |
| Auth | Planned |

The API currently returns a single welcome message from the root route and does not yet connect to any database.

---

## 3. Folder Architecture

Current:

```
expanse-tracker-api/
├── index.js                 # Server bootstrap + app entry point
├── package.json
├── package-lock.json
├── .env                     # Local config (PORT, MONGODB_URI, secrets)
├── architecture-docs/
│   └── architecture.md     # This document
└── mcp-server-docs/         # OpenCode ⇄ MongoDB read-only MCP plan
```

Target (as modules are added):

```
expanse-tracker-api/
├── index.js                       # Entry point: dotenv → connect DB → listen
├── src/
│   ├── app.js                     # Express app assembly (middleware + routes)
│   ├── config/
│   │   └── db.js                  # MongoDB/Mongoose connection
│   ├── models/                    # Mongoose schemas
│   │   ├── Expense.js
│   │   └── Category.js
│   ├── routes/                    # Express Routers per resource
│   │   ├── expense.routes.js
│   │   └── category.routes.js
│   ├── controllers/               # Request handlers (thin, no DB logic)
│   ├── services/                  # Business logic / persistence access
│   ├── middleware/                # Validation, error handler, auth
│   └── utils/                     # Helpers, constants
└── architecture-docs/
    └── architecture.md
```

`index.js` stays the composition root: load config, connect to MongoDB, mount the app, and listen on `PORT`.

---

## 4. Application Startup Flow

```
              index.js
                 │  require('dotenv').config()
                 │
                 ▼
        PORT = process.env.PORT || 8000
                 │
                 ▼
        express()  →  app
                 │
                 ▼
   Middleware: express.json(), express.urlencoded({extended:true})
                 │
                 ▼
   Routes mounted (e.g. GET / → { message: 'Expense Tracker API' })
                 │
                 ▼
   app.listen(PORT)  →  console.log('Server running on http://localhost:8000')
```

Planned insert: MongoDB connection (async) between config load and `app.listen` — connect via Mongoose to the Atlas URI in `.env` (DB `expense_tracker_db`), so the server only accepts requests once the DB is ready.

---

## 5. Routing and API Endpoints

Defined in `index.js` today; moves to `src/routes/` as resources are added. All payloads are JSON.

### Current

| Method | Path | Handler |
| --- | --- | --- |
| GET | `/` | Welcome message (`{ message: 'Expense Tracker API' }`) |

### Planned (mirrors the frontend models)

**Expenses**

| Method | Path | Description |
| --- | --- | --- |
| GET | `/api/expenses` | List expenses (filter/paginate) |
| GET | `/api/expenses/:id` | Get one expense |
| POST | `/api/expenses` | Create expense |
| PUT | `/api/expenses/:id` | Update expense |
| DELETE | `/api/expenses/:id` | Delete expense |

**Categories**

| Method | Path | Description |
| --- | --- | --- |
| GET | `/api/categories` | List categories |
| GET | `/api/categories/:id` | Get one category |
| POST | `/api/categories` | Create category |
| PUT | `/api/categories/:id` | Update category |
| DELETE | `/api/categories/:id` | Delete category |

Planned conveniences (beyond CRUD, for the dashboard/reports):

| Method | Path | Description |
| --- | --- | --- |
| GET | `/api/expenses/summary` | Totals, current-month totals, category totals |

Routing convention: `routes/xxx.routes.js` export an `express.Router();` the `index.js`/`app.js` mounts them under `/api`.

---

## 6. Middleware

Registered in order in `index.js`:

- `express.json()` — parse JSON bodies
- `express.urlencoded({ extended: true })` — parse URL-encoded bodies
- Route handlers
- Planned: centralized error handler (last), request logging, cross-origin/CORS for the Angular dev server, and validation middleware per body/schema.

---

## 7. Data Layer (Planned)

### Persistence

- MongoDB Atlas cluster **Cluster0** (`cluster0.cdftphh.mongodb.net`), application database **`expense_tracker_db`**.
- Driver: Mongoose (not yet installed — next step).
- Connection string from `process.env.MONGODB_URI`; never hard-coded in source.
- Connection centralized in `src/config/db.js`; exported function called during startup before `app.listen`.
- Atlas user: `shomerituparnacyberswift_db_user` (SRV scheme, db user with read/write on `expense_tracker_db`). Password lives only in `.env`.

### Models

Mirror the frontend contracts in `expanse-tracker-web/architecture-docs/architecture.md`:

```javascript
// src/models/Expense.js
const expenseSchema = new Schema({
  title: { type: String, required: true },
  amount: { type: Number, required: true, min: 0 },
  categoryId: { type: String, required: true },
  date: { type: Date, required: true },
  paymentMethod: {
    type: String,
    enum: ['cash', 'upi', 'card', 'bank_transfer', 'other'],
    required: true
  },
  notes: String,
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date
}, { timestamps: true });

// src/models/Category.js
const categorySchema = new Schema({
  name: { type: String, required: true },
  color: { type: String, required: true },
  icon: String,
  isActive: { type: Boolean, default: true }
}, { timestamps: true });
```

---

## 8. Configuration and Environment

`.env` (git-ignored) drives all environment-specific settings. Example:

```
PORT=8000
MONGODB_URI=mongodb+srv://<db_user>:<password>@cluster0.cdftphh.mongodb.net/expense_tracker_db
```

Never commit `.env` or real credentials. Confirm `.gitignore` excludes:

```
.env
.env.*
```

---

## 9. Conventions and Constraints

- **CommonJS** modules throughout (`require` / `module.exports`) unless converted deliberately.
- `index.js` = composition root only; no business logic inline.
- Controllers are thin; services own business logic; models own persistence.
- Consistent REST style: plural resource names, JSON bodies, semantic HTTP status codes.
- Validation at the boundary: reject invalid/missing fields with `400`; return `404` for unknown resources.
- Seeding/init data for categories matches the frontend defaults.

---

## 10. Build and Verification

### Scripts (`package.json`)

```
npm start        node index.js
npm run dev      nodemon index.js    (hot reload)
npm test         no test specified yet
```

### Verification

- Start with `npm run dev`.
- `GET http://localhost:8000/` → `{ "message": "Expense Tracker API" }`.
- Add a test script (e.g. `node --test` / supertest) once routes exist.

---

## 11. System Boundary

```
Angular Frontend ──HTTP /api──▶ Node.js/Express API ──Mongoose──▶ MongoDB
```

The frontend service layer is designed so only shared contracts change when this boundary is wired up (see the web architecture doc, section 11). Authentication and CORS are the main concerns to add before first integration.

---

## 12. Reference

- `expanse-tracker-web/architecture-docs/architecture.md` — frontend architecture and shared data contracts.
- `mcp-server-docs/mongodb_mcp_server_plan.md` — read-only MongoDB access for OpenCode; the application DB itself is managed by this API.