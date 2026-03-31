# How to Use ClickHouse with Streamlit

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Streamlit, Python, Data Visualization, Analytics, Dashboard

Description: Learn how to connect a Streamlit app to ClickHouse to build interactive data dashboards with real-time query results using Python.

---

Streamlit is a popular Python framework for building data applications with minimal code. Connecting it to ClickHouse lets you create interactive dashboards backed by fast analytical queries on large datasets.

## Installing Dependencies

```bash
pip install streamlit clickhouse-connect pandas
```

## Connecting to ClickHouse

Use the `clickhouse-connect` client for Python.

```python
import clickhouse_connect

client = clickhouse_connect.get_client(
    host='localhost',
    port=8123,
    username='default',
    password='',
    database='default'
)
```

## Building a Simple Dashboard

Create a file `app.py`:

```python
import streamlit as st
import clickhouse_connect
import pandas as pd

st.title("ClickHouse Analytics Dashboard")

client = clickhouse_connect.get_client(
    host='localhost',
    port=8123,
    username='default',
    password=''
)

@st.cache_data(ttl=60)
def get_top_pages():
    result = client.query("""
        SELECT page, count() AS views
        FROM default.page_views
        WHERE created_at >= today() - 7
        GROUP BY page
        ORDER BY views DESC
        LIMIT 10
    """)
    return pd.DataFrame(result.result_rows, columns=result.column_names)

df = get_top_pages()
st.bar_chart(df.set_index('page')['views'])
st.dataframe(df)
```

Run it with:

```bash
streamlit run app.py
```

## Adding Interactive Filters

Streamlit widgets make it easy to add date pickers and dropdowns.

```python
import streamlit as st
from datetime import date, timedelta

start_date = st.date_input("Start date", value=date.today() - timedelta(days=7))
end_date = st.date_input("End date", value=date.today())

@st.cache_data(ttl=30)
def get_events(start, end):
    result = client.query("""
        SELECT toDate(created_at) AS day, count() AS events
        FROM default.events
        WHERE created_at BETWEEN %(start)s AND %(end)s
        GROUP BY day
        ORDER BY day
    """, parameters={'start': str(start), 'end': str(end)})
    return pd.DataFrame(result.result_rows, columns=result.column_names)

df = get_events(start_date, end_date)
st.line_chart(df.set_index('day'))
```

## Using Session State for Live Refresh

```python
if st.button("Refresh Data"):
    st.cache_data.clear()
    st.rerun()
```

## Displaying Raw Query Results

```python
query = st.text_area("Custom SQL Query", value="SELECT * FROM system.tables LIMIT 5")
if st.button("Run Query"):
    result = client.query(query)
    df = pd.DataFrame(result.result_rows, columns=result.column_names)
    st.dataframe(df)
```

## Summary

Streamlit and ClickHouse together enable rapid development of analytical dashboards. ClickHouse provides fast query execution over large datasets, while Streamlit's caching, widgets, and chart components let you build useful data apps with a few dozen lines of Python.
