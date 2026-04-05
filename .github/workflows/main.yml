#!/usr/bin/env python3
"""
🥭 Grocery Price Tracker
========================
Tracks mango prices across BigBasket, Amazon, Blinkit, Zepto, and MangoPoint.
Saves a record whenever a price changes, and always saves once at end-of-day (23:59 IST).

Setup:
    pip install -r requirements.txt
    playwright install chromium

Run manually:
    python price_tracker.py

Environment Variables:
    SAVE_MODE   : 'on_change' (default) | 'force' (always save, used at 23:59)
    TZ          : Set to 'Asia/Kolkata' — GitHub Actions workflow handles this
    PINCODE     : Delivery pincode for Blinkit/Zepto (default: 560001 = Bengaluru)

Output:
    price_history.csv — one row per price event (change, initial, or end-of-day)
"""

import csv
import os
import re
import sys
from datetime import datetime
from pathlib import Path
from typing import Optional

try:
    from playwright.sync_api import sync_playwright, TimeoutError as PlaywrightTimeoutError
except ImportError:
    print("ERROR: playwright not installed.")
    print("Run:   pip install playwright && playwright install chromium")
    sys.exit(1)


# ─── CONFIGURATION ────────────────────────────────────────────────────────────

DATA_FILE = "price_history.csv"
CSV_HEADERS = ["timestamp", "product_id", "site", "product_name", "quantity", "price_inr", "change_type"]

# Delivery pincode for location-gated sites (Blinkit, Zepto)
# Change this to your local pincode if prices differ by region
PINCODE = os.environ.get("PINCODE", "560001")

PRODUCTS = [
    {
        "id": "bigbasket_banganapalli_1kg",
        "site": "BigBasket",
        "url": "https://www.bigbasket.com/pd/10000298/fresho-banganapalli-mango-1-kg/",
    },
    {
        "id": "amazon_banganapalli_2pcs",
        "site": "Amazon",
        "url": "https://www.amazon.in/Fresh-Mango-Banganapalli-2-Pieces/dp/B07B8SQ3SG/",
    },
    {
        "id": "amazon_thothapuri_12pcs",
        "site": "Amazon",
        "url": "https://www.amazon.in/Fresh-Mango-Thothapuri-12-Pieces/dp/B07B8PQ3G9/",
    },
    {
        "id": "amazon_alphonso_ratnagiri",
        "site": "Amazon",
        "url": "https://www.amazon.in/Fresh-Mango-Alphonso-Ratnagiri-Approx-350-400/dp/B09TBKX3CF/",
    },
    {
        "id": "blinkit_banganapalli",
        "site": "Blinkit",
        "url": "https://blinkit.com/prn/banganapalli-mango/prid/267649",
    },
    {
        "id": "zepto_banganapalli",
        "site": "Zepto",
        "url": "https://www.zepto.com/pn/mango-banganapalli/pvid/a616d8b7-b335-4038-8e96-73300f6c8cd6",
    },
    {
        "id": "mangopoint_banganapalli",
        "site": "MangoPoint",
        "url": "https://www.mangopoint.in/fresh-mangoes/banganapalli-mangoes-1kg-3kg-5kg-10kg-box-naturally-ripened-and-carbide-free/",
    },
]


# ─── DATA STORAGE ─────────────────────────────────────────────────────────────

def ensure_data_file() -> None:
    """Create CSV with headers if it doesn't exist yet."""
    if not Path(DATA_FILE).exists():
        with open(DATA_FILE, "w", newline="", encoding="utf-8") as f:
            csv.writer(f).writerow(CSV_HEADERS)


def load_last_prices() -> dict:
    """Return dict of product_id → last saved row (most recent wins)."""
    ensure_data_file()
    last: dict = {}
    try:
        with open(DATA_FILE, "r", newline="", encoding="utf-8") as f:
            for row in csv.DictReader(f):
                last[row["product_id"]] = row
    except Exception:
        pass
    return last


def append_price(product_id: str, site: str, name: str, qty: str,
                 price: float, change_type: str) -> None:
    """Append one price record to the CSV."""
    ensure_data_file()
    ts = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    with open(DATA_FILE, "a", newline="", encoding="utf-8") as f:
        csv.writer(f).writerow([ts, product_id, site, name, qty, f"{price:.2f}", change_type])
    print(f"    ✓  {name}  |  {qty}  |  ₹{price:.2f}  |  [{change_type}]")


# ─── SCRAPING HELPERS ─────────────────────────────────────────────────────────

def first_text(page, selectors: list) -> Optional[str]:
    """Try CSS selectors in order; return first non-empty inner text."""
    for sel in selectors:
        try:
            el = page.query_selector(sel)
            if el:
                text = el.inner_text().strip()
                if text:
                    return text
        except Exception:
            continue
    return None


def extract_price_value(text: Optional[str]) -> Optional[float]:
    """Parse currency strings like '₹89', 'Rs. 89.50', '₹ 1,299' → float."""
    if not text:
        return None
    # Strip known currency markers then find the first number
    cleaned = re.sub(r"[₹$€£RsSr.,\s]", "", text)
    match = re.search(r"\d+\.?\d*", cleaned)
    return float(match.group()) if match else None


def qty_from_name(name: Optional[str]) -> Optional[str]:
    """Extract weight/count hint from product name, e.g. '1 kg', '12 Pieces'."""
    if not name:
        return None
    match = re.search(
        r"\b(\d+\.?\d*\s*(?:kg|g|gm|grams?|pieces?|pcs|pc|pack|nos|count|ct|box|doz))\b",
        name, re.IGNORECASE
    )
    return match.group(1).strip() if match else None


def stealth_page(browser):
    """Create a browser context that looks like a real Chrome user."""
    ctx = browser.new_context(
        user_agent=(
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
            "AppleWebKit/537.36 (KHTML, like Gecko) "
            "Chrome/124.0.0.0 Safari/537.36"
        ),
        viewport={"width": 1366, "height": 768},
        locale="en-IN",
        timezone_id="Asia/Kolkata",
        geolocation={"latitude": 12.9716, "longitude": 77.5946},  # Bengaluru
        permissions=["geolocation"],
        extra_http_headers={"Accept-Language": "en-IN,en;q=0.9,hi;q=0.8"},
    )
    # Hide the automation flag that sites detect
    ctx.add_init_script(
        "Object.defineProperty(navigator, 'webdriver', {get: () => undefined})"
    )
    return ctx.new_page()


def try_fill_pincode(page) -> None:
    """If a pincode/location modal is shown, fill it and dismiss."""
    try:
        pin_input = page.query_selector(
            "input[placeholder*='pincode' i], "
            "input[placeholder*='Pincode' i], "
            "input[placeholder*='Enter your' i], "
            "input[placeholder*='location' i]"
        )
        if pin_input:
            pin_input.fill(PINCODE)
            page.keyboard.press("Enter")
            page.wait_for_timeout(3_000)
    except Exception:
        pass


# ─── SITE-SPECIFIC SCRAPERS ───────────────────────────────────────────────────

def scrape_bigbasket(browser, url: str) -> dict:
    page = stealth_page(browser)
    try:
        page.goto(url, wait_until="domcontentloaded", timeout=30_000)
        page.wait_for_timeout(4_000)

        name = first_text(page, [
            "h1[class*='prod-name']",
            "h1[class*='ProdName']",
            "h1[class*='name']",
            "h1",
        ])
        price_text = first_text(page, [
            "span[class*='DiscountedPrice']",
            "span[class*='discnt-price']",
            "[class*='Price--size-md']",
            "span[class*='Price']",
            "[class*='price'] span",
        ])
        qty = first_text(page, [
            "[class*='PackSize']",
            "[class*='pack-size']",
            "[class*='PackChooser']",
            "[class*='qty']",
        ]) or qty_from_name(name)

        return {"name": name, "qty": qty, "price": extract_price_value(price_text)}
    finally:
        page.context.close()


def scrape_amazon(browser, url: str) -> dict:
    page = stealth_page(browser)
    try:
        page.goto(url, wait_until="domcontentloaded", timeout=30_000)
        page.wait_for_timeout(3_000)

        name = first_text(page, ["#productTitle", "span#productTitle"])

        # Amazon often splits price into whole + fraction; .a-offscreen has full value
        price_text = first_text(page, [
            ".a-price .a-offscreen",
            "#priceblock_ourprice",
            "#priceblock_dealprice",
            "#price_inside_buybox",
            ".apexPriceToPay .a-offscreen",
            ".a-price-whole",
        ])

        qty = (
            first_text(page, ["#selected-size-name", ".selection", "#native_dropdown_selected_size_name"])
            or qty_from_name(name)
        )

        return {"name": name, "qty": qty, "price": extract_price_value(price_text)}
    finally:
        page.context.close()


def scrape_blinkit(browser, url: str) -> dict:
    page = stealth_page(browser)
    try:
        page.goto(url, wait_until="networkidle", timeout=45_000)
        page.wait_for_timeout(5_000)
        try_fill_pincode(page)

        name = first_text(page, [
            "h1[class*='ProductName']",
            "h1[class*='product-name']",
            "[class*='product__name']",
            "h1",
        ])
        price_text = first_text(page, [
            "[class*='Price__StyledPrice']",
            "[class*='product-price']",
            "[class*='ProductPrice']",
            "[class*='price']",
        ])
        qty = (
            first_text(page, [
                "[class*='Quantity__']",
                "[class*='weight']",
                "[class*='size']",
                "[class*='variant']",
            ])
            or qty_from_name(name)
        )

        return {"name": name, "qty": qty, "price": extract_price_value(price_text)}
    finally:
        page.context.close()


def scrape_zepto(browser, url: str) -> dict:
    page = stealth_page(browser)
    try:
        page.goto(url, wait_until="networkidle", timeout=45_000)
        page.wait_for_timeout(5_000)
        try_fill_pincode(page)

        name = first_text(page, [
            "h1",
            "[class*='product-name']",
            "[class*='productName']",
            "[class*='item-name']",
        ])
        price_text = first_text(page, [
            "[class*='product-price']",
            "[class*='ProductPrice']",
            "[class*='price']",
            "[class*='Price']",
        ])
        qty = (
            first_text(page, [
                "[class*='quantity']",
                "[class*='Quantity']",
                "[class*='variant']",
                "[class*='weight']",
            ])
            or qty_from_name(name)
        )

        return {"name": name, "qty": qty, "price": extract_price_value(price_text)}
    finally:
        page.context.close()


def scrape_mangopoint(browser, url: str) -> dict:
    page = stealth_page(browser)
    try:
        page.goto(url, wait_until="domcontentloaded", timeout=30_000)
        page.wait_for_timeout(2_000)

        name = first_text(page, [
            "h1.product_title",
            "h1.entry-title",
            "h1",
        ])
        # WooCommerce price structure: sale price is inside <ins>
        price_text = first_text(page, [
            ".price ins .woocommerce-Price-amount bdi",
            ".price ins .woocommerce-Price-amount",
            ".price .woocommerce-Price-amount bdi",
            ".price .woocommerce-Price-amount",
            ".price ins .amount",
            ".price .amount",
            ".woocommerce-Price-amount",
        ])
        qty = (
            first_text(page, [
                ".quantity",
                "[class*='variant']",
                "[class*='weight']",
                "[class*='size']",
            ])
            or qty_from_name(name)
        )

        return {"name": name, "qty": qty, "price": extract_price_value(price_text)}
    finally:
        page.context.close()


def scrape(browser, product: dict) -> dict:
    """Dispatch to the right scraper, return name/qty/price dict."""
    site = product["site"]
    url  = product["url"]
    try:
        if site == "BigBasket":
            return scrape_bigbasket(browser, url)
        elif site == "Amazon":
            return scrape_amazon(browser, url)
        elif site == "Blinkit":
            return scrape_blinkit(browser, url)
        elif site == "Zepto":
            return scrape_zepto(browser, url)
        elif site == "MangoPoint":
            return scrape_mangopoint(browser, url)
    except PlaywrightTimeoutError:
        print(f"    ✗  Timeout on {site}")
    except Exception as exc:
        print(f"    ✗  Error on {site}: {exc}")
    return {"name": None, "qty": None, "price": None}


# ─── MAIN ─────────────────────────────────────────────────────────────────────

def main() -> None:
    now = datetime.now()
    save_mode = os.environ.get("SAVE_MODE", "on_change").lower()

    # Force-save at end of day: either via env var OR if script runs at 23:59
    is_force = save_mode == "force" or (now.hour == 23 and now.minute >= 59)

    print(f"\n{'═' * 62}")
    print(f"  🥭  Price Tracker  ·  {now.strftime('%Y-%m-%d  %H:%M:%S')}")
    print(f"  Mode : {'FORCE SAVE — End-of-Day' if is_force else 'Save on Change Only'}")
    print(f"  Pincode : {PINCODE}")
    print(f"{'═' * 62}")

    last_prices = load_last_prices()
    saved_count = 0

    with sync_playwright() as pw:
        browser = pw.chromium.launch(
            headless=True,
            args=[
                "--no-sandbox",
                "--disable-setuid-sandbox",
                "--disable-dev-shm-usage",
                "--disable-blink-features=AutomationControlled",
                "--disable-infobars",
                "--window-size=1366,768",
            ],
        )

        for product in PRODUCTS:
            pid  = product["id"]
            site = product["site"]
            print(f"\n  → [{site}]  {pid}")

            result = scrape(browser, product)
            name  = result.get("name")  or f"Unknown — {pid}"
            qty   = result.get("qty")   or "N/A"
            price = result.get("price")

            if price is None:
                print("    ✗  Could not extract price — skipping")
                continue

            last       = last_prices.get(pid)
            last_price = float(last["price_inr"]) if last and last.get("price_inr") else None

            if last_price is None:
                # Very first record for this product
                append_price(pid, site, name, qty, price, "initial")
                saved_count += 1

            elif abs(price - last_price) > 0.01:
                # Price has changed
                arrow = "↑" if price > last_price else "↓"
                diff  = abs(price - last_price)
                append_price(pid, site, name, qty, price,
                             f"price_change {arrow}₹{diff:.2f}")
                saved_count += 1

            elif is_force:
                # End-of-day checkpoint — price unchanged but we still log it
                append_price(pid, site, name, qty, price, "end_of_day_no_change")
                saved_count += 1

            else:
                print(f"    —  No change  ₹{price:.2f}  (was ₹{last_price:.2f})")

        browser.close()

    print(f"\n{'═' * 62}")
    print(f"  ✅  Run complete — {saved_count} new entr{'y' if saved_count == 1 else 'ies'} saved")
    print(f"{'═' * 62}\n")


if __name__ == "__main__":
    main()
