import streamlit as st
import pandas as pd
import matplotlib.pyplot as plt
import plotly.express as px

# Streamlit App Titel
st.set_page_config(page_title="Amazon Lagerbestands-Analyse", layout="wide")
st.title("📊 Amazon Lagerbestands-Analyse Tool")
st.write("Lade die Lagerbestandsdatei hoch und erhalte umfassende Analysen zu Verkäufen, Lagerbewegungen und Retouren.")

# Datei-Upload
uploaded_file = st.file_uploader("📂 Lade deine Amazon Lagerbestands-Datei hoch (CSV oder Excel)", type=["csv", "xlsx"])

if uploaded_file is not None:
    # Datei einlesen
    if uploaded_file.name.endswith(".csv"):
        df = pd.read_csv(uploaded_file, encoding="ISO-8859-1")
    else:
        df = pd.read_excel(uploaded_file)
    
    st.sidebar.header("🔍 Filter")
    zeitraum = st.sidebar.selectbox("📅 Wähle den Zeitraum", [
        "Letzter Tag (gestern)", "In den letzten 3 Tagen", "In den letzten 7 Tagen", "In den letzten 14 Tagen",
        "Letzte 30 Tage", "In den letzten 90 Tagen", "In den letzten 180 Tagen", "In den letzten 365 Tagen"])
    
    event_type = st.sidebar.selectbox("🔄 Wähle den Event-Typ", df["Event Type"].unique())
    df_filtered = df[df["Event Type"] == event_type]
    
    selected_skus = st.sidebar.multiselect("📦 Wähle SKUs aus", df_filtered["MSKU"].unique())
    if selected_skus:
        df_filtered = df_filtered[df_filtered["MSKU"].isin(selected_skus)]
    
    st.subheader("📊 Dashboard - Übersicht")
    col1, col2, col3 = st.columns(3)
    col1.metric("Gesamtverkäufe", df_filtered["Quantity"].sum())
    col2.metric("Gesamtretouren", df_filtered[df_filtered["Event Type"].str.contains("Return", na=False)]["Quantity"].sum())
    col3.metric("Ungewöhnliche Anomalien", df_filtered["Quantity"].abs().max())
    
    # Verkaufsanalyse
    st.subheader("📈 Verkaufsanalyse")
    verkauf_sku = df_filtered.groupby(["MSKU", "ASIN", "Title"])["Quantity"].sum().reset_index()
    fig = px.bar(verkauf_sku.head(10), x="Title", y="Quantity", title="Top 10 Produkte nach Verkäufen", text_auto=True)
    st.plotly_chart(fig, use_container_width=True)
    
    # Retourenanalyse
    st.subheader("🔄 Retourenanalyse")
    retouren = df_filtered[df_filtered["Event Type"].str.contains("Return", na=False)]
    retouren_sku = retouren.groupby(["MSKU", "ASIN", "Title"])["Quantity"].sum().reset_index()
    fig = px.bar(retouren_sku.head(10), x="Title", y="Quantity", title="Top 10 Produkte mit Retouren", text_auto=True, color="Quantity")
    st.plotly_chart(fig, use_container_width=True)
    
    # Anomalie-Erkennung
    st.subheader("⚠️ Anomalie-Erkennung")
    threshold = st.slider("📈 Setze die Schwelle für ungewöhnliche Veränderungen (%)", 10, 100, 50)
    verkauf_sku_change = verkauf_sku["Quantity"].pct_change().fillna(0) * 100
    anomalien = verkauf_sku[verkauf_sku_change.abs() > threshold]
    if not anomalien.empty:
        st.error("🚨 Ungewöhnliche Veränderungen erkannt!")
        st.dataframe(anomalien)
    else:
        st.success("✅ Keine ungewöhnlichen Veränderungen erkannt.")
    
    # Lagerbestandsentwicklung
    st.subheader("📉 Lagerbestandsverlauf")
    if "Date and Time" in df.columns:
        df["Date and Time"] = pd.to_datetime(df["Date and Time"])
        df_sorted = df.sort_values(by=["Date and Time"])
        bestandsverlauf = df_sorted.groupby(["Date and Time", "MSKU"]).sum()["Quantity"].unstack().fillna(0)
        st.line_chart(bestandsverlauf)
    
    # Export der Ergebnisse
    st.subheader("📤 Export")
    if st.button("Export als Excel"):
        df_filtered.to_excel("amazon_analysen.xlsx", index=False)
        st.success("Datei erfolgreich gespeichert: amazon_analysen.xlsx")

# GitHub Dateien erstellen
with open("requirements.txt", "w") as req_file:
    req_file.write("streamlit\npandas\nmatplotlib\nplotly\nopenpyxl")

with open(".gitignore", "w") as gitignore:
    gitignore.write("__pycache__/\n*.csv\n*.xlsx\n.env")

with open("README.md", "w") as readme:
    readme.write("""
# Amazon Lagerbestands-Analyse Tool

Dieses Tool analysiert Amazon-Lagerbestandsberichte und bietet umfassende Einblicke in Verkäufe, Retouren und Lagerbewegungen.

## Neue Features:
- 📊 **Modernes Dashboard mit interaktiven Charts**
- 🎯 **Filterung nach Zeitraum, Event-Typ und SKUs**
- ⚠️ **Anomalie-Erkennung für plötzliche Veränderungen**
- 📉 **Lagerbestandsverlauf als interaktives Diagramm**
- 📤 **Export der Daten als Excel-Datei**

## Installation
1. Python 3 installieren
2. Benötigte Pakete installieren:
   ```
   pip install -r requirements.txt
   ```
3. Das Tool starten:
   ```
   streamlit run app.py
   ```

## Nutzung
1. Amazon-Lagerbestandsbericht hochladen (CSV/Excel)
2. Dashboard-Filter setzen (Zeitraum, SKUs, Event-Typ)
3. Verkaufs- und Retourenanalysen einsehen
4. Anomalien und Lagerbestände auswerten
5. Ergebnisse als Excel exportieren

""")
