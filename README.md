# 🥭 Grocery Price Tracker

Automatically tracks mango prices across **BigBasket, Amazon, Blinkit, Zepto, and MangoPoint**.
Runs hourly via **GitHub Actions** (free) — no server needed.

---

## What it does

| Scenario | Action |
|---|---|
| Price changed from last check | Saves new price immediately with a `price_change ↑/↓₹X` label |
| No price change during the day | Saves one `end_of_day_no_change` record at 23:59 IST |
| First time seeing a product | Saves an `initial` record |

All records go into **`price_history.csv`** — committed back to the repo automatically.

---

## Setup (5 minutes)

### Step 1 — Create a GitHub repository

1. Go to [github.com/new](https://github.com/new)
2. Create a **new public or private repo** (e.g. `mango-price-tracker`)
3. Clone it to your computer:
   ```bash
   git clone https://github.com/YOUR_USERNAME/mango-price-tracker.git
   cd mango-price-tracker
   ```

### Step 2 — Copy these files into the repo

Copy the entire contents of this folder into your cloned repo:
```
price_tracker.py
requirements.txt
price_history.csv
.gitignore
.github/
  workflows/
    price_tracker.yml
```

### Step 3 — Push to GitHub

```bash
git add .
git commit -m "Initial commit"
git push origin main
```

### Step 4 — Enable GitHub Actions

1. Open your repo on GitHub
2. Click the **Actions** tab
3. Click **"I understand my workflows, go ahead and enable them"** if prompted
4. Done! The workflow will now run on the schedule automatically.

### Step 5 — Test it manually

1. Go to **Actions → 🥭 Grocery Price Tracker**
2. Click **"Run workflow"** → **"Run workflow"**
3. Watch the run complete (takes ~5–10 minutes)
4. Check `price_history.csv` in your repo for new entries

---

## Customize

### Change the delivery pincode

Blinkit and Zepto are location-aware. Edit the `PINCODE` line in the workflow file:

```yaml
# .github/workflows/price_tracker.yml
env:
  PINCODE: "560001"   # ← change to your city's pincode
```

Or set it in the Python script directly:
```python
PINCODE = os.environ.get("PINCODE", "560001")
```

### Add or remove products

Edit the `PRODUCTS` list in `price_tracker.py`:
```python
PRODUCTS = [
    {
        "id": "my_product_id",       # unique key (used in CSV)
        "site": "BigBasket",         # display name
        "url": "https://...",        # product page URL
    },
    # ... more products
]
```

### Change the check frequency

Edit the cron schedule in `.github/workflows/price_tracker.yml`.
Default is **every hour from 6 AM to 11 PM IST**.

```yaml
- cron: "30 0-17 * * *"   # Hourly  6:00 AM – 11:30 PM IST
- cron: "29 18 * * *"     # End-of-day  23:59 IST
```

Use [crontab.guru](https://crontab.guru) to build a custom schedule.

---

## Output: price_history.csv

| Column | Example | Description |
|---|---|---|
| `timestamp` | `2026-04-05 08:30:12` | When the record was saved (IST) |
| `product_id` | `bigbasket_banganapalli_1kg` | Unique product key |
| `site` | `BigBasket` | Store name |
| `product_name` | `Fresho Banganapalli Mango` | Scraped product name |
| `quantity` | `1 kg` | Scraped pack size |
| `price_inr` | `89.00` | Price in ₹ |
| `change_type` | `price_change ↓₹10.00` | Why this row was saved |

---

## GitHub Actions free tier

| Plan | Free minutes/month |
|---|---|
| Public repo | Unlimited |
| Private repo (free plan) | 2,000 min |

At ~3 min/run × 18 runs/day × 30 days ≈ **1,620 min/month** for a private repo — just within the free tier.
For a public repo it's completely free.

---

## Troubleshooting

**Price shows as blank / not found**
Some sites (especially Blinkit and Zepto) heavily use JavaScript and may require a valid pincode or block headless browsers. Try running locally first to debug:
```bash
pip install -r requirements.txt
playwright install chromium
python price_tracker.py
```

**Amazon shows "Sign in" instead of price**
Amazon occasionally gates prices behind login. The script handles this gracefully by skipping the product.

**Blinkit/Zepto asks for location**
The script automatically tries to fill in the pincode. If it still fails, the site may have changed its UI. Open an issue or update the selector in `scrape_blinkit()` / `scrape_zepto()`.
