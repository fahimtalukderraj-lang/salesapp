# streamlit_app.py
"""
Daily Sales & Profit - Streamlit app
Paste into a file named streamlit_app.py and include requirements.txt in repo.
Streamlit Cloud will run it automatically.
"""

import json
import sqlite3
from datetime import date, datetime
from pathlib import Path
from typing import Any, Dict, List

import pandas as pd
import streamlit as st

# === Config / DB path ===
BASE_DIR = Path(__file__).parent
DB_PATH = BASE_DIR / "daily_sales.db"

# === Helpers ===
def safe_float(x: Any, default: float = 0.0) -> float:
    """Convert a value to float robustly."""
    try:
        # handle empty strings, None, etc.
        if x is None or (isinstance(x, str) and x.strip() == ""):
            return default
        return float(x)
    except Exception:
        return default

def get_conn():
    # Use string path for sqlite3
    return sqlite3.connect(str(DB_PATH), timeout=10)

# --- Database helpers -------------------------------------------------------
def init_db():
    conn = get_conn()
    c = conn.cursor()
    c.execute(
        """
        CREATE TABLE IF NOT EXISTS daily_sales (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            entry_date TEXT,
            data_json TEXT,
            saved_at TEXT
        )
        """
    )
    conn.commit()
    conn.close()

def save_entry(entry_date: str, data_json: str):
    conn = get_conn()
    c = conn.cursor()
    c.execute(
        "INSERT INTO daily_sales (entry_date, data_json, saved_at) VALUES (?, ?, ?)",
        (entry_date, data_json, datetime.utcnow().isoformat()),
    )
    conn.commit()
    conn.close()

def load_all_entries() -> pd.DataFrame:
    conn = get_conn()
    try:
        df = pd.read_sql_query("SELECT id, entry_date, data_json, saved_at FROM daily_sales ORDER BY entry_date DESC", conn)
    except Exception:
        # if table missing or DB corrupt, return empty
        df = pd.DataFrame(columns=["id", "entry_date", "data_json", "saved_at"])
    finally:
        conn.close()
    return df

# --- Calculation helpers ----------------------------------------------------
def calc_store_and_gas(data: Dict[str, Any]) -> Dict[str, float]:
    categories: List[Dict[str, Any]] = data.get("categories", [])
    for cat in categories:
        pct = safe_float(cat.get("profit_pct", 0.0))
        total = safe_float(cat.get("total_sales", 0.0))
        cat["profit"] = total * pct

    store_subtotal_profit = sum(safe_float(c.get("profit", 0.0)) for c in categories)

    # Gas
    gas: List[Dict[str, Any]] = data.get("gas", [])
    for g in gas:
        cost = safe_float(g.get("cost_per_gal", 0.0))
        sale = safe_float(g.get("sales_per_gal", 0.0))
        sale_gal = safe_float(g.get("sale_gal", 0.0))
        g["profit_per_gal"] = sale - cost
        g["profit"] = g["profit_per_gal"] * sale_gal
        # make sure numeric fields are stored as numbers
        g["cost_per_gal"] = cost
        g["sales_per_gal"] = sale
        g["sale_gal"] = sale_gal

    gas_total_profit = sum(safe_float(g.get("profit", 0.0)) for g in gas)

    payments = data.get("payments", {}) or {}
    payments = {k: safe_float(v, 0.0) for k, v in payments.items()}
    payment_total = sum(payments.values())

    lottery_payout = payments.get("lottery_p_o", 0.0)
    cash_paid = payments.get("cash_paid", 0.0)
    taxes = safe_float(data.get("taxes", 0.0))

    net_profit = store_subtotal_profit + gas_total_profit - cash_paid - lottery_payout - taxes

    total_sales_store = sum(safe_float(c.get("total_sales", 0.0)) for c in categories)
    # total gas sales revenue = sum(sales_per_gal * sale_gal)
    total_gas_sales_revenue = sum(safe_float(g.get("sales_per_gal", 0.0)) * safe_float(g.get("sale_gal", 0.0)) for g in gas)
    total_overall_sales = total_sales_store + total_gas_sales_revenue

    results = {
        "store_subtotal_profit": round(store_subtotal_profit, 2),
        "gas_total_profit": round(gas_total_profit, 2),
        "net_profit": round(net_profit, 2),
        "payment_total": round(payment_total, 2),
        "total_sales_store": round(total_sales_store, 2),
        "total_overall_sales": round(total_overall_sales, 2),
    }
    return results

# === UI ===
st.set_page_config(page_title="Daily Sales & Profit", layout="centered")
init_db()

st.title("Daily Sales & Profit — Gas Station / Convenience Store")
st.markdown("Enter daily numbers and get automatic calculations, save to DB, and view monthly/annual reports.")

menu = st.sidebar.selectbox("Menu", ["Enter Daily", "Reports", "Data Export", "Settings"])

try:
    if menu == "Enter Daily":
        st.header("Enter Daily Sales")
        with st.form("daily_form"):
            entry_date = st.date_input("Date", value=date.today())

            st.subheader("Store Categories (enter Profit% as decimal: 0.15 = 15%)")
            default_cats = ["CIGARETTE", "TOBACCO", "GROCERY", "BEER", "DELI", "OTHERS", "INSIDE SALES", "LOTTERY"]
            categories = []
            for cat_name in default_cats:
                col1, col2, col3 = st.columns([3, 2, 2])
                with col1:
                    st.markdown(f"**{cat_name}**")
                with col2:
                    pct = st.text_input(f"{cat_name} profit % (decimal)", value="0.10", key=f"pct_{cat_name}")
                with col3:
                    total = st.text_input(f"{cat_name} total sales", value="0", key=f"total_{cat_name}")
                categories.append({"name": cat_name, "profit_pct": pct, "total_sales": total})

            st.subheader("Other Adjustments")
            taxes = st.number_input("Taxes / Other deductions", value=0.0, format="%.2f")
            st.markdown("---")

            st.subheader("Gas Sales Detail")
            gas_grades = ["REGULAR", "MIDGRADE", "PREMIUM", "DIESEL"]
            gas = []
            for g in gas_grades:
                cols = st.columns([2, 2, 2, 2])
                with cols[0]:
                    st.markdown(f"**{g}**")
                with cols[1]:
                    cost = st.text_input(f"{g} - Cost / gal", value="0.00", key=f"cost_{g}")
                with cols[2]:
                    sale = st.text_input(f"{g} - Sale / gal", value="0.00", key=f"sale_{g}")
                with cols[3]:
                    sale_gal = st.text_input(f"{g} - Sale gal (quantity)", value="0", key=f"gal_{g}")
                gas.append({"grade": g, "cost_per_gal": cost, "sales_per_gal": sale, "sale_gal": sale_gal})

            st.markdown("---")
            st.subheader("Payment / Collections")
            colp1, colp2, colp3 = st.columns(3)
            with colp1:
                cash = st.number_input("CASH", value=0.0, format="%.2f", key="cash")
                credit = st.number_input("CREDIT CARD", value=0.0, format="%.2f", key="credit")
            with colp2:
                debit = st.number_input("DEBIT CARD", value=0.0, format="%.2f", key="debit")
                ebt = st.number_input("EBT", value=0.0, format="%.2f", key="ebt")
            with colp3:
                lottery_p_o = st.number_input("LOTTERY P/O (payout)", value=0.0, format="%.2f", key="lottery_p_o")
                cash_paid = st.number_input("CASH PAID (expenses)", value=0.0, format="%.2f", key="cash_paid")

            payments = {
                "cash": cash, "credit": credit, "debit": debit, "ebt": ebt,
                "lottery_p_o": lottery_p_o, "cash_paid": cash_paid
            }

            submitted = st.form_submit_button("Calculate & Save")
            if submitted:
                data = {"categories": categories, "taxes": taxes, "gas": gas, "payments": payments}
                results = calc_store_and_gas(data)

                st.success("Calculated!")
                st.metric("Store Subtotal Profit", f"${results['store_subtotal_profit']:.2f}")
                st.metric("Gas Total Profit", f"${results['gas_total_profit']:.2f}")
                st.metric("Net Profit (store+gas - payouts/taxes)", f"${results['net_profit']:.2f}")
                st.write("Payment Total:", f"${results['payment_total']:.2f}")
                st.write("Total Sales (store only):", f"${results['total_sales_store']:.2f}")
                st.write("Total Overall Sales (store + gas estimated):", f"${results['total_overall_sales']:.2f}")

                # Save to DB
                save_entry(entry_date.isoformat(), json.dumps({"data": data, "results": results}))
                st.info("Saved to local database (daily_sales.db)")

    elif menu == "Reports":
        st.header("Reports")
        df = load_all_entries()
        if df.empty:
            st.info("No entries yet. Go to 'Enter Daily' to add one.")
        else:
            rows = []
            for _, row in df.iterrows():
                payload = json.loads(row["data_json"])
                results = payload.get("results", {})
                rows.append({
                    "id": row["id"],
                    "date": row["entry_date"],
                    "store_profit": results.get("store_subtotal_profit", 0),
                    "gas_profit": results.get("gas_total_profit", 0),
                    "net_profit": results.get("net_profit", 0),
                    "payment_total": results.get("payment_total", 0),
                    "total_sales_store": results.get("total_sales_store", 0),
                    "total_overall_sales": results.get("total_overall_sales", 0),
                    "raw": row["data_json"]
                })
            rpt_df = pd.DataFrame(rows)
            rpt_df["date"] = pd.to_datetime(rpt_df["date"]).dt.date

            st.subheader("Recent entries")
            st.dataframe(rpt_df[["id", "date", "total_sales_store", "store_profit", "gas_profit", "net_profit", "payment_total"]].sort_values("date", ascending=False))

            st.subheader("Summary")
            col_a, col_b = st.columns(2)
            min_date = rpt_df["date"].min()
            max_date = rpt_df["date"].max()
            with col_a:
                start = st.date_input("Start date", min_value=min_date, value=min_date)
            with col_b:
                end = st.date_input("End date", min_value=min_date, value=max_date)

            filtered = rpt_df[(rpt_df["date"] >= start) & (rpt_df["date"] <= end)]
            if filtered.empty:
                st.warning("No entries in selected date range.")
            else:
                total_store_sales = filtered["total_sales_store"].sum()
                total_overall_sales = filtered["total_overall_sales"].sum()
                total_store_profit = filtered["store_profit"].sum()
                total_gas_profit = filtered["gas_profit"].sum()
                total_net_profit = filtered["net_profit"].sum()
                st.metric("Total Store Sales", f"${total_store_sales:.2f}")
                st.metric("Total Overall Sales", f"${total_overall_sales:.2f}")
                st.metric("Total Store Profit", f"${total_store_profit:.2f}")
                st.metric("Total Gas Profit", f"${total_gas_profit:.2f}")
                st.metric("Total Net Profit", f"${total_net_profit:.2f}")

                st.subheader("Monthly Aggregation")
                agg = filtered.copy()
                agg["month"] = pd.to_datetime(agg["date"]).dt.to_period("M")
                monthly = agg.groupby("month").agg({
                    "total_sales_store": "sum",
                    "store_profit": "sum",
                    "gas_profit": "sum",
                    "net_profit": "sum"
                }).reset_index()
                monthly["month"] = monthly["month"].astype(str)
                st.dataframe(monthly)

                st.subheader("Annual Aggregation")
                agg["year"] = pd.to_datetime(agg["date"]).dt.year
                annual = agg.groupby("year").agg({
                    "total_sales_store": "sum",
                    "store_profit": "sum",
                    "gas_profit": "sum",
                    "net_profit": "sum"
                }).reset_index()
                st.dataframe(annual)

    elif menu == "Data Export":
        st.header("Data Export")
        df = load_all_entries()
        if df.empty:
            st.info("No data to export.")
        else:
            rows = []
            for _, row in df.iterrows():
                payload = json.loads(row["data_json"])
                results = payload.get("results", {})
                rows.append({
                    "id": row["id"],
                    "date": row["entry_date"],
                    "total_store_sales": results.get("total_sales_store", 0),
                    "store_subtotal_profit": results.get("store_subtotal_profit", 0),
                    "gas_total_profit": results.get("gas_total_profit", 0),
                    "net_profit": results.get("net_profit", 0),
                    "raw_json": row["data_json"]
                })
            export_df = pd.DataFrame(rows)
            export_csv = export_df.to_csv(index=False).encode("utf-8")
            st.download_button("Download CSV", data=export_csv, file_name="daily_sales_export.csv", mime="text/csv")
            st.dataframe(export_df)

    elif menu == "Settings":
        st.header("Settings")
        st.markdown("Database file: `daily_sales.db` (in the app folder).")
        st.markdown("To reset database, delete the file (or use the button below).")
        if st.button("Reset / Delete local DB"):
            if DB_PATH.exists():
                DB_PATH.unlink()
                init_db()
                st.success("Deleted and reinitialized database.")
            else:
                st.info("No database file found - already clean.")

    st.sidebar.markdown("---")
    st.sidebar.write("App stores data locally in `daily_sales.db`.")
    st.sidebar.write("Back up your DB regularly using Data Export or by downloading the DB from the host.")

except Exception as exc:
    st.error("An unexpected error occurred. See details below.")
    st.exception(exc)
