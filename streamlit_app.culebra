import pandas as pd
import numpy as np
import streamlit as st
import matplotlib.pyplot as plt

st.set_page_config(page_title="Twin-Source Weather Logger", layout="wide")

st.title("Twin-Source Weather Logger — SD Log Explorer")

uploaded = st.file_uploader("Upload sensor_log.csv", type=["csv"])

@st.cache_data
def load_df(file) -> pd.DataFrame:
    df = pd.read_csv(file)
    df["date_time"] = pd.to_datetime(df["date_time"], errors="coerce")
    df = df.dropna(subset=["date_time"])
    df["sensor_value"] = pd.to_numeric(df["sensor_value"], errors="coerce")
    return df

@st.cache_data
def pivot(df: pd.DataFrame) -> pd.DataFrame:
    wide = (
        df.pivot_table(index="date_time", columns="measurement", values="sensor_value", aggfunc="mean")
          .sort_index()
    )
    if {"Temperature", "web_temp75024"}.issubset(wide.columns):
        wide["delta_temp_F"] = wide["Temperature"] - wide["web_temp75024"]
    if {"Humidity", "web_humidity75024"}.issubset(wide.columns):
        wide["delta_humidity_pct"] = wide["Humidity"] - wide["web_humidity75024"]
    return wide

if uploaded is None:
    st.info("Upload your CSV to begin.")
    st.stop()

df = load_df(uploaded)
wide = pivot(df)

# Sidebar filters
min_dt, max_dt = wide.index.min(), wide.index.max()
start, end = st.sidebar.slider(
    "Time window",
    min_value=min_dt.to_pydatetime(),
    max_value=max_dt.to_pydatetime(),
    value=(min_dt.to_pydatetime(), max_dt.to_pydatetime()),
)
wide_f = wide.loc[start:end]

st.subheader("Quick stats")
col1, col2, col3 = st.columns(3)
col1.metric("Rows (wide)", f"{len(wide_f):,}")
col2.metric("Measurements (wide cols)", f"{wide_f.shape[1]}")
col3.metric("Date range", f"{wide_f.index.min().date()} → {wide_f.index.max().date()}")

st.subheader("Time series")
plot_cols = st.multiselect(
    "Select columns",
    options=list(wide.columns),
    default=[c for c in ["Temperature", "web_temp75024", "Humidity", "web_humidity75024", "delta_temp_F"] if c in wide.columns],
)

if plot_cols:
    fig, ax = plt.subplots()
    wide_f[plot_cols].plot(ax=ax, alpha=0.9)
    ax.set_xlabel("time")
    ax.grid(True, alpha=0.3)
    st.pyplot(fig, clear_figure=True)

st.subheader("Delta distributions")
for col in ["delta_temp_F", "delta_humidity_pct"]:
    if col in wide_f.columns:
        fig, ax = plt.subplots()
        ax.hist(wide_f[col].dropna(), bins=30)
        ax.set_title(col)
        st.pyplot(fig, clear_figure=True)

st.subheader("Raw log preview")
st.dataframe(df.sort_values("date_time", ascending=False).head(200), use_container_width=True)

st.download_button(
    "Download wide_timeseries.csv",
    data=wide_f.reset_index().to_csv(index=False).encode("utf-8"),
    file_name="wide_timeseries_filtered.csv",
    mime="text/csv",
)
