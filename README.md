# Daily Stock Report — Streamlit App

A single-file Streamlit application that builds a daily stock report from five
uploaded Excel files. It merges store stock, DC stock, sell-out, pending purchase
orders, product master, and supplier lead time, calculates stock coverage
metrics, and returns a downloadable multi-sheet workbook.

The batch version of the same logic lives in
[`../DAILY_STOCK_REPORT`](../DAILY_STOCK_REPORT) as a notebook pipeline; this
repository keeps only one interactive app.

---

## Input files

Upload all five through the browser:

| # | Upload | Sheet | Key columns |
|---|---|---|---|
| 1 | Weekly CJ stock report (`.xlsx`) | `Sahamit Report` (header row 2) | `CJ_Item`, `Division`, product name/category/brand, store stock |
| 2 | Daily DC stock (`.xlsx`) | `Sheet1` | `Material`, `Plant` (maps to DC), stock qty/value |
| 3 | Sell-out past 30 days (`.xlsx`) | first sheet | `Calendar Day`, `CJ_Item`, `Total_SellOut_Qty`, `Total_Sellout_Amt` |
| 4 | Access extract (`.xlsx`) | `Master_Product`, `Pending_All_Div` | `CJ_Item`, `SHM_Item`, supplier, order qty/unit, delivery date, DC |
| 5 | Master lead time (`.xlsx`) | `All_Product` | `CJ_Item`, `SHM_Item`, `OwnerSCM`, base lead time |

No data is bundled. Bring your own files; the app never writes them to disk.

---

## Processing

1. Reads the five workbooks and normalises column names.
2. Cleans CJ stock (removes excluded divisions) and DC stock (maps plant codes,
   floors negative stock at zero, aggregates per item × DC).
3. Aggregates sell-out into 7-day and 30-day windows.
4. Standardises DC names, converts PO quantities to pieces, and aggregates PO
   quantity + minimum delivery date per item × DC.
5. Joins stock, sell-out, PO, product master, and lead time into one table.
6. Calculates:
   - total remaining stock across DCs and per DC
   - average sales for 90 / 30 / 7 days
   - DC sales-allocation ratios
   - DOH (Days on Hand), including DOH with pending PO
   - stock cover dates that account for PO arrival dates
7. Handles duplicate `CJ_Item` rows with explicit rules to avoid double counting.
8. Runs a final SQL shaping step through `pandasql`.
9. Produces quantity and carton views plus PO analysis tables.
10. Streams the result to a download button as a multi-sheet `.xlsx`.

---

## Output

`Sahamit_Daily_Stock_Report_<date>.xlsx`

| Sheet | Description |
|---|---|
| `Data by Qty` | Stock report in pieces |
| `Data by Cartons` | Same report in cartons |
| `All PO Pending` | Pending PO detail |
| `MIN ETA` | Earliest delivery date per SKU per DC |

---

## Run locally

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
streamlit run streamlit_daily_stock.py
```

Then open <http://localhost:8501> and upload the five files.

The included Dev Container (`.devcontainer/`) starts the same command and
forwards port 8501 if you prefer a container.

---

## Notes

- DOH is based on 90-day average daily sales unless stated otherwise.
- Pending PO quantities only extend cover when the delivery date is earlier than
  the projected stock-out date.
- The report is generated in memory; nothing is persisted server-side.
- No company data, credentials, or internal paths are included in this
  repository. All sample/screenshot data should be synthetic.
