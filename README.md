# Hospitality-Analysis
Hospitality Analysis Dashboard using Python, Streamlit and Plotly. Author: ShrutiDhumal.
# Hospitality Analysis Dashboard
# Author: ShrutiDhumal

import streamlit as st
import pandas as pd
import plotly.express as px

st.set_page_config(page_title="Hospitality Analysis | ShrutiDhumal", page_icon="🏨", layout="wide")

st.title("🏨 Hospitality Analysis Dashboard")
st.caption("Author: ShrutiDhumal | Hotel bookings, revenue, occupancy and cancellation insights")

sample_path = "data/sample_hotel_data.csv"
uploaded = st.sidebar.file_uploader("Upload your hotel dataset (CSV or Excel)", type=["csv", "xlsx"])
use_sample = st.sidebar.checkbox("Use included sample data", value=True)

def load_data(file):
    if file is None:
        return pd.read_csv(sample_path)
    if file.name.lower().endswith(".csv"):
        return pd.read_csv(file)
    return pd.read_excel(file)

try:
    if uploaded is not None:
        df = load_data(uploaded)
    elif use_sample:
        df = load_data(None)
    else:
        st.info("Upload a CSV/Excel file or select 'Use included sample data'.")
        st.stop()
except Exception as e:
    st.error(f"Could not load data: {e}")
    st.stop()

# Standardise common column names for flexible use
aliases = {
    "revenue": ["revenue", "total_revenue", "booking_revenue", "amount"],
    "occupancy": ["occupancy", "occupancy_rate", "occupancy_percent", "occ_pct"],
    "booking_date": ["booking_date", "date", "stay_date", "arrival_date"],
    "hotel": ["hotel", "hotel_name", "property", "property_name"],
    "city": ["city", "hotel_city"],
    "cancelled": ["is_canceled", "is_cancelled", "cancelled", "canceled", "cancellation"],
    "adr": ["adr", "average_daily_rate", "daily_rate"],
}
lower_map = {str(c).strip().lower(): c for c in df.columns}
def find_col(key):
    for candidate in aliases[key]:
        if candidate in lower_map:
            return lower_map[candidate]
    return None

revenue_col = find_col("revenue")
occupancy_col = find_col("occupancy")
date_col = find_col("booking_date")
hotel_col = find_col("hotel")
city_col = find_col("city")
cancel_col = find_col("cancelled")
adr_col = find_col("adr")

if date_col:
    df[date_col] = pd.to_datetime(df[date_col], errors="coerce")
if revenue_col:
    df[revenue_col] = pd.to_numeric(df[revenue_col], errors="coerce")
if occupancy_col:
    df[occupancy_col] = pd.to_numeric(df[occupancy_col], errors="coerce")
if adr_col:
    df[adr_col] = pd.to_numeric(df[adr_col], errors="coerce")

st.sidebar.subheader("Filters")
filtered = df.copy()
if hotel_col:
    choices = sorted(filtered[hotel_col].dropna().astype(str).unique())
    selected = st.sidebar.multiselect("Hotel / Property", choices, default=choices)
    if selected:
        filtered = filtered[filtered[hotel_col].astype(str).isin(selected)]
if city_col:
    choices = sorted(filtered[city_col].dropna().astype(str).unique())
    selected = st.sidebar.multiselect("City", choices, default=choices)
    if selected:
        filtered = filtered[filtered[city_col].astype(str).isin(selected)]

if filtered.empty:
    st.warning("No records match the selected filters.")
    st.stop()

total_bookings = len(filtered)
total_revenue = filtered[revenue_col].sum() if revenue_col else None
avg_occupancy = filtered[occupancy_col].mean() if occupancy_col else None
cancel_rate = None
if cancel_col:
    c = filtered[cancel_col]
    if c.dtype == bool:
        cancel_rate = c.mean() * 100
    else:
        vals = c.astype(str).str.lower().str.strip()
        yes = vals.isin(["1", "true", "yes", "cancelled", "canceled"])
        cancel_rate = yes.mean() * 100

k1, k2, k3, k4 = st.columns(4)
k1.metric("Total Bookings", f"{total_bookings:,}")
k2.metric("Total Revenue", f"₹{total_revenue:,.0f}" if total_revenue is not None else "N/A")
k3.metric("Average Occupancy", f"{avg_occupancy:.1f}%" if avg_occupancy is not None else "N/A")
k4.metric("Cancellation Rate", f"{cancel_rate:.1f}%" if cancel_rate is not None else "N/A")

st.divider()
left, right = st.columns(2)
with left:
    st.subheader("Revenue Trend")
    if date_col and revenue_col:
        trend = filtered.dropna(subset=[date_col]).groupby(filtered[date_col].dt.to_period("M").astype(str))[revenue_col].sum().reset_index()
        trend.columns = ["Month", "Revenue"]
        if not trend.empty:
            st.plotly_chart(px.line(trend, x="Month", y="Revenue", markers=True, title="Monthly Revenue"), use_container_width=True)
        else:
            st.info("No valid dates available for the revenue trend.")
    elif revenue_col and hotel_col:
        chart = filtered.groupby(hotel_col, as_index=False)[revenue_col].sum()
        st.plotly_chart(px.bar(chart, x=hotel_col, y=revenue_col, title="Revenue by Hotel"), use_container_width=True)
    else:
        st.info("Add a date and revenue column (or hotel and revenue columns) to show revenue trends.")

with right:
    st.subheader("Bookings by Hotel / City")
    group_col = hotel_col or city_col
    if group_col:
        counts = filtered[group_col].astype(str).value_counts().head(12).rename_axis(group_col).reset_index(name="Bookings")
        st.plotly_chart(px.bar(counts, x=group_col, y="Bookings", title=f"Bookings by {group_col}"), use_container_width=True)
    else:
        st.info("Add a Hotel or City column to compare bookings.")

left2, right2 = st.columns(2)
with left2:
    st.subheader("Occupancy Analysis")
    if occupancy_col:
        if hotel_col:
            occ = filtered.groupby(hotel_col, as_index=False)[occupancy_col].mean()
            st.plotly_chart(px.bar(occ, x=hotel_col, y=occupancy_col, title="Average Occupancy by Hotel"), use_container_width=True)
        else:
            st.plotly_chart(px.histogram(filtered, x=occupancy_col, title="Occupancy Distribution"), use_container_width=True)
    else:
        st.info("Add an occupancy column to show occupancy analysis.")

with right2:
    st.subheader("Cancellation Analysis")
    if cancel_col:
        vals = filtered[cancel_col].astype(str).str.lower().str.strip()
        cancelled = vals.isin(["1", "true", "yes", "cancelled", "canceled"]).map({True:"Cancelled", False:"Not Cancelled"})
        chart_df = cancelled.value_counts().rename_axis("Status").reset_index(name="Bookings")
        st.plotly_chart(px.pie(chart_df, names="Status", values="Bookings", hole=0.45, title="Cancellation Split"), use_container_width=True)
    else:
        st.info("Add a cancellation column (e.g. is_canceled) to show cancellation analysis.")

st.subheader("Dataset Preview")
st.dataframe(filtered.head(100), use_container_width=True)
st.download_button("Download filtered data as CSV", filtered.to_csv(index=False).encode("utf-8"), "hospitality_filtered_data.csv", "text/csv")

st.caption("Note: KPIs and charts are based on the uploaded/sample dataset. Confirm column meanings and currency units before using results for business decisions.")
