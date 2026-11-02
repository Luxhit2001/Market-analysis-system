# Marketly

A Marketing & Sales dashboard built with Django. It stores orders, products, customers and footfall in one SQLite database, summarises them on an interactive dashboard, and recommends products to each customer with a transparent, rule-based score.

Charts are drawn in the browser with plain JavaScript and SVG, and product images are generated line drawings, so there is no chart library and there are no image files to manage.

## Features

- **Dashboard** with four stat cards (sales, orders, customers, interactions), each compared with the equal-length previous period.
- **Period filter:** this month, last month, last 30 days, this quarter, this year.
- **Channel performance** (website, marketplace, store) shown as a gauge, with units, revenue and change.
- **Average sales:** monthly revenue against monthly target, filterable by year, product and category.
- **Top 3 products** by units sold (daily, 7-day or 30-day range).
- **Visitors:** footfall by channel plus a weekday × time-slot busy-hours heatmap (last 28 days).
- **Stock alerts:** products with fewer than 40 units are flagged, with a suggested reorder quantity.
- **For you** page: per-customer recommendations with a reason on every card, search, category chips, sorting, purchase-frequency and category-mix charts, and saved (heart) products.
- **Tables** for products, orders and customers with search.
- **CSV export** of order lines for the selected period.
- **Django admin** for all models.

## Requirements

- Python 3.10 or newer
- Django 5.0 or newer (see `requirements.txt`)
- A browser. Internet is only needed for the Inter web font; a system font is used otherwise.

## Quick start

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

python manage.py makemigrations dashboard
python manage.py migrate
python manage.py seed            # demo data; use --reset to rebuild
python manage.py runserver
```

Open http://127.0.0.1:8000/.

To use the admin site, create a login first:

```bash
python manage.py createsuperuser
```

> The `dashboard/migrations/` folder ships empty, so `makemigrations dashboard` is required before `migrate`. The bundled `db.sqlite3` does not contain the app's tables yet.

## Pages

| URL | Purpose |
|---|---|
| `/` | Main analytics dashboard |
| `/for-you/` | Personalised recommendations (`?customer=<id>`) |
| `/products/` | Product table; `?stock=low` shows low stock only |
| `/orders/` | Latest 60 orders |
| `/customers/` | Customers ranked by spend |
| `/export/orders.csv` | CSV download (`?period=month\|last_month\|30d\|quarter\|year`) |
| `/admin/` | Django admin |

Query parameters are validated against allowed values; anything invalid falls back to the default.

## Data model

| Model | Notes |
|---|---|
| `Category` | Unique name |
| `Product` | Category, kind (drives the line art), price, MRP, stock, rating, tile hue; `discount` is computed |
| `Customer` | Unique email, comma-separated interests, joined date |
| `Order` | Customer, channel, status (pending / shipped / delivered / returned), indexed `created_at` |
| `OrderItem` | Product, quantity and the unit price at purchase time |
| `MonthlyTarget` | Revenue target per year and month |
| `Visit` | Footfall per date, channel and slot (09-12, 12-15, 15-18) |
| `Interaction` | Daily count of chats, calls and reviews |

## How recommendations work

Every product gets a score for the selected customer:

1. Base: `rating / 5 + discount% / 100`
2. `+3` if the product's category is in the customer's interests
3. `+1.5 + min(times bought in category, 10) / 10` if they have bought from that category
4. `-0.8` if they have already bought that exact product

The top three become "Handpicked for you", and the top three from categories outside their interests become "Discover more". Each card shows its reason, such as "Because you like Outdoor", "You often buy Footwear", "Top rated" or "Popular right now".

## Demo data

`python manage.py seed` creates 6 categories, 14 products, 12 customers, orders from 1 January of the current year to today, 120 days of visits and interactions, and monthly targets. It uses a fixed random seed, so the data is repeatable. Order status depends on age: today is pending, under 3 days is shipped, older is delivered, with about 3% returned. Run it again with `--reset` to delete and rebuild the demo data.

## Project layout

```
market/
├── manage.py
├── requirements.txt
├── db.sqlite3
├── market_project/          settings, urls, wsgi, asgi
└── dashboard/
    ├── models.py            database schema
    ├── views.py             dashboard, recommendations, tables, CSV export
    ├── urls.py
    ├── fmt.py               ₹ amounts (Indian grouping), K/L/Cr, % change, "time ago"
    ├── admin.py
    ├── templatetags/ui.py   icon and product-art template tags
    ├── management/commands/seed.py
    ├── templates/dashboard/ base, dashboard, recommend, table, _card, _icons
    └── static/dashboard/    app.css, app.js (SVG charts, filters, saved items)
```

## Using the app in an existing project

Keep your own `settings.py` and make these changes:

- Add `"dashboard"` to `INSTALLED_APPS`.
- Set `TIME_ZONE = "Asia/Kolkata"` (or your zone) and keep `STATIC_URL`.
- Include `dashboard.urls` in your root `urls.py`.

## Before deploying

The bundled settings are for development only. Change these first:

- Set `DEBUG = False` and load `SECRET_KEY` from the environment.
- Restrict `ALLOWED_HOSTS` (currently `["*"]`).
- Add authentication; dashboard pages are currently open to anyone who can reach the server.
- Configure static file serving and consider a production database.

## Known limitations

- Data is demo data; there is no live store, cart or payment integration.
- Recommendations use fixed weights and do not learn from behaviour.
- Tables show the first 60 orders or 100 products/customers, without pagination.
- Saved (heart) products are stored in the browser's local storage only.
- The dashboard search box is disabled; use the Products, Orders and Customers pages to search.
- There are no automated tests.
