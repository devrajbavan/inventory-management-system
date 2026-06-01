# InventoryPro — Inventory & Order Management System

A production-ready, fully containerized full-stack application for managing **products, customers, orders, and inventory**.

| Layer | Stack |
| --- | --- |
| Frontend | React + TypeScript + Vite, Tailwind CSS, TanStack Query, React Router |
| Backend | Python, FastAPI, SQLAlchemy 2.0, Pydantic v2 |
| Database | PostgreSQL 16 |
| Infra | Docker, Docker Compose, Nginx |

---

## 🚀 Live Deployment

| Service | URL |
| --- | --- |
| **Frontend** | https://inventory-system-gamma-eight.vercel.app/ |
| **Backend API** | https://inventory-management-system-1-52hr.onrender.com |
| **API docs (Swagger)** | https://inventory-management-system-1-52hr.onrender.com/docs |

**Login credentials:** `admin@inventorypro.com` / `admin123`

---

## Quick start (Docker Compose)

```bash
cp .env.example .env          # adjust secrets if you like
docker compose up --build
```

| Service | URL |
| --- | --- |
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:8000 |
| API docs (Swagger) | http://localhost:8000/docs |

**Login:** `admin@inventorypro.com` / `admin123` (credentials are pre-filled on the login screen and configurable via env vars). The database is auto-created and seeded with demo products/customers.

---

## Local development (without Docker)

**Backend**
```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
# point DATABASE_URL at a running Postgres, then:
uvicorn app.main:app --reload
```

**Frontend**
```bash
cd frontend
npm install
echo "VITE_API_URL=http://localhost:8000" > .env
npm run dev      # http://localhost:5173
```

---

## API overview

| Method | Endpoint | Description |
| --- | --- | --- |
| POST | `/auth/login` | Authenticate, returns a JWT |
| GET/POST | `/products` | List / create products |
| GET/PUT/DELETE | `/products/{id}` | Read / update / delete a product |
| GET/POST | `/customers` | List / create customers |
| GET/DELETE | `/customers/{id}` | Read / delete a customer |
| GET/POST | `/orders` | List / create orders |
| GET/DELETE | `/orders/{id}` | Read / cancel an order |
| GET | `/dashboard/summary` | Dashboard stats, recent orders, low stock, top products |
| GET | `/health` | Health check |

Full interactive documentation is available at `/docs`.

---

## Business rules (enforced by the backend)

- **Unique product SKU** and **unique customer email** — enforced with DB constraints + friendly `409` responses.
- **Stock can never go negative** — DB `CHECK` constraint + validation.
- **Orders are rejected when inventory is insufficient** (`409`) — checked *before* any mutation.
- **Creating an order automatically decrements stock**; deleting an order **restocks** it.
- **Order totals are computed server-side** from a snapshot of each product's price at order time (multi-item orders supported).
- Concurrent orders cannot oversell — product rows are locked with `SELECT ... FOR UPDATE` inside the order transaction.
- All endpoints validate input (Pydantic) and return appropriate HTTP status codes.

---

## Database schema

```
users        (id, email[unique], full_name, hashed_password, created_at)
products     (id, name, sku[unique], description, category, price,
              quantity_in_stock, low_stock_threshold, created_at, updated_at)
customers    (id, full_name, email[unique], phone, company, created_at, updated_at)
orders       (id, order_number[unique], customer_id→customers, status,
              total_amount, notes, created_at, updated_at)
order_items  (id, order_id→orders[cascade], product_id→products,
              product_name, quantity, unit_price, line_total)
```

**Query optimisation / indexing**

- Unique B-tree indexes on `products.sku`, `customers.email`, `orders.order_number`.
- `ix_products_low_stock (quantity_in_stock, low_stock_threshold)` to back the dashboard low-stock query.
- Composite `ix_orders_customer_created (customer_id, created_at)` + index on `orders.created_at` for "recent orders" sorting.
- Indexes on `order_items.order_id` and `order_items.product_id` for joins/aggregations.
- The orders list and dashboard use **single grouped queries** (join + `COUNT`) to avoid N+1 lookups; order details use `selectinload` for eager relationship loading.

---

## Docker notes

- Backend uses a **multi-stage** build on `python:3.12-slim`, runs as a **non-root** user, and ships a `HEALTHCHECK`.
- Frontend builds with Node and is served by **Nginx** (`nginx:alpine`) with SPA routing fallback.
- PostgreSQL data persists in the named volume `inventorypro_postgres_data`.
- **No credentials are hardcoded** — everything is supplied via environment variables.

---

## Deployment

- **Backend** → Render / Railway / Fly.io. Build from `backend/Dockerfile`; set `DATABASE_URL`, `SECRET_KEY`, `CORS_ORIGINS` (include the frontend's URL).
- **Frontend** → Vercel / Netlify. Build command `npm run build`, output `dist`, and set `VITE_API_URL` to the deployed backend URL **at build time**.

> The backend Docker image can be pushed to Docker Hub:
> ```bash
> docker build -t <user>/inventorypro-backend:latest ./backend
> docker push <user>/inventorypro-backend:latest
> ```

---

## Project structure

```
.
├── backend/
│   ├── app/
│   │   ├── core/        # config, database, security
│   │   ├── models/      # SQLAlchemy ORM models
│   │   ├── schemas/     # Pydantic request/response models
│   │   ├── routers/     # API route handlers
│   │   ├── seed.py      # admin + demo data
│   │   └── main.py      # app factory, CORS, error handling
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── api/         # axios client + endpoint wrappers
│   │   ├── components/  # reusable UI
│   │   ├── context/     # auth + toast providers
│   │   ├── layouts/     # app shell (sidebar + topbar)
│   │   ├── pages/       # Login, Dashboard, Products, Customers, Orders
│   │   └── lib/         # formatting helpers
│   ├── Dockerfile
│   └── nginx.conf
└── docker-compose.yml
```
