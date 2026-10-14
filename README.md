# APL-Logistics-Profitability-Analytics
APL Logistics Profitability Intelligence Dashboard using Python, Pandas, Plotly and Streamlit

import streamlit as st
import pandas as pd
import plotly.express as px

st.set_page_config(
    page_title="APL Logistics Analytics",
    page_icon="📦",
    layout="wide"
)

st.title("APL Logistics Analytics")
st.caption("Sales | Profitability | Customers | Delivery")

uploaded_file = st.sidebar.file_uploader(
    "Upload a sanitized logistics CSV",
    type=["csv"]
)

if uploaded_file is None:
    st.info("Upload your APL Logistics CSV to begin.")
    st.stop()

df = pd.read_csv(uploaded_file)

# Convert numeric columns safely
numeric_cols = [
    "Sales", "Order Item Total",
    "Order Profit Per Order", "Benefit per order",
    "Order Item Discount", "Order Item Quantity",
    "Days for shipping (real)",
    "Days for shipment (scheduled)",
    "Late_delivery_risk"
]

for col in numeric_cols:
    if col in df.columns:
        df[col] = pd.to_numeric(df[col], errors="coerce")

required = [
    "Sales", "Order Profit Per Order",
    "Category Name", "Market", "Delivery Status"
]

missing = [c for c in required if c not in df.columns]
if missing:
    st.error(f"Missing required columns: {missing}")
    st.stop()

# Sidebar filters
st.sidebar.header("Filters")

markets = sorted(df["Market"].dropna().unique())
selected_markets = st.sidebar.multiselect(
    "Market", markets, default=markets
)

categories = sorted(df["Category Name"].dropna().unique())
selected_categories = st.sidebar.multiselect(
    "Product category", categories, default=categories
)

data = df[
    df["Market"].isin(selected_markets)
    & df["Category Name"].isin(selected_categories)
].copy()

if data.empty:
    st.warning("No rows match the selected filters.")
    st.stop()

# KPI cards
sales = data["Sales"].sum()
profit = data["Order Profit Per Order"].sum()
profit_margin = profit / sales * 100 if sales else 0
late_pct = (
    data["Late_delivery_risk"].mean() * 100
    if "Late_delivery_risk" in data.columns else 0
)

a, b, c, d = st.columns(4)
a.metric("Total Sales", f"{sales:,.2f}")
b.metric("Total Profit", f"{profit:,.2f}")
c.metric("Profit / Sales", f"{profit_margin:.2f}%")
d.metric("Late Delivery Risk", f"{late_pct:.2f}%")

st.divider()

# Profit by category
left, right = st.columns(2)

with left:
    st.subheader("Profit by Product Category")
    category_profit = (
        data.groupby("Category Name", as_index=False)
        ["Order Profit Per Order"].sum()
        .sort_values("Order Profit Per Order")
    )
    fig = px.bar(
        category_profit,
        x="Order Profit Per Order",
        y="Category Name",
        orientation="h",
        title="Total profit by category"
    )
    st.plotly_chart(fig, use_container_width=True)

with right:
    st.subheader("Sales by Market")
    market_sales = (
        data.groupby("Market", as_index=False)["Sales"].sum()
    )
    fig = px.bar(
        market_sales,
        x="Market",
        y="Sales",
        title="Sales by market"
    )
    st.plotly_chart(fig, use_container_width=True)

# Delivery analysis
st.subheader("Delivery Status")
delivery = (
    data["Delivery Status"]
    .value_counts()
    .rename_axis("Delivery Status")
    .reset_index(name="Rows")
)

fig = px.bar(
    delivery,
    x="Delivery Status",
    y="Rows",
    title="Number of order lines by delivery status"
)
st.plotly_chart(fig, use_container_width=True)

# Discount analysis
if "Order Item Discount" in data.columns:
    st.subheader("Discount and Profit Analysis")
    fig = px.scatter(
        data.sample(min(5000, len(data)), random_state=42),
        x="Order Item Discount",
        y="Order Profit Per Order",
        color="Category Name",
        opacity=0.6,
        title="Discount amount versus profit per order line"
    )
    st.plotly_chart(fig, use_container_width=True)

st.subheader("Filtered Data Preview")
st.dataframe(data.head(100), use_container_width=True)

st.download_button(
    "Download filtered data",
    data=data.to_csv(index=False).encode("utf-8"),
    file_name="apl_filtered_data.csv",
    mime="text/csv"
)
