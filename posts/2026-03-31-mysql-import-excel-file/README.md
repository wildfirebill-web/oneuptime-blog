# How to Import an Excel File into MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Excel, Import, Python, Data Migration

Description: Learn how to import Excel (.xlsx) files into MySQL using Python with openpyxl and pandas, enabling reliable spreadsheet-to-database data loading.

---

## Introduction

Excel files are one of the most common formats for receiving data from business stakeholders. While MySQL has no native Excel import capability, Python libraries like `openpyxl` and `pandas` make it straightforward to read `.xlsx` files and load them into MySQL tables efficiently.

## Prerequisites

Install the required Python packages:

```bash
pip install openpyxl pandas mysql-connector-python sqlalchemy
```

## Sample Excel File Structure

Assume an Excel file `employees.xlsx` with columns: `id`, `name`, `department`, `salary`, `hire_date`.

## Importing with pandas and SQLAlchemy

pandas provides the simplest approach - read the Excel file and write directly to MySQL:

```python
import pandas as pd
from sqlalchemy import create_engine

# Read the Excel file
df = pd.read_excel('employees.xlsx', sheet_name='Sheet1')

# Preview the data
print(df.head())
print(df.dtypes)

# Create SQLAlchemy engine
engine = create_engine('mysql+mysqlconnector://root:password@localhost/mydb')

# Write to MySQL table
df.to_sql(
    name='employees',
    con=engine,
    if_exists='append',   # use 'replace' to drop and recreate
    index=False,
    chunksize=1000
)
print(f"Imported {len(df)} rows")
```

## Importing with openpyxl for More Control

For custom column mapping and data transformation:

```python
import openpyxl
import mysql.connector
from datetime import datetime

conn = mysql.connector.connect(
    host='localhost', user='root',
    password='password', database='mydb'
)
cursor = conn.cursor()

cursor.execute("""
    CREATE TABLE IF NOT EXISTS employees (
        id INT PRIMARY KEY,
        name VARCHAR(100),
        department VARCHAR(100),
        salary DECIMAL(10,2),
        hire_date DATE
    )
""")

wb = openpyxl.load_workbook('employees.xlsx')
ws = wb.active

insert_sql = """
    INSERT INTO employees (id, name, department, salary, hire_date)
    VALUES (%s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
      name=VALUES(name), department=VALUES(department),
      salary=VALUES(salary), hire_date=VALUES(hire_date)
"""

rows_to_insert = []
for row in ws.iter_rows(min_row=2, values_only=True):
    emp_id, name, dept, salary, hire_date = row
    if isinstance(hire_date, datetime):
        hire_date = hire_date.date()
    rows_to_insert.append((emp_id, name, dept, salary, hire_date))

cursor.executemany(insert_sql, rows_to_insert)
conn.commit()
print(f"Imported {cursor.rowcount} rows")
cursor.close()
conn.close()
```

## Handling Multiple Sheets

Import different sheets into different tables:

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine('mysql+mysqlconnector://root:password@localhost/mydb')

excel_file = pd.ExcelFile('company_data.xlsx')

for sheet_name in excel_file.sheet_names:
    df = excel_file.parse(sheet_name)
    table_name = sheet_name.lower().replace(' ', '_')
    df.to_sql(table_name, con=engine, if_exists='replace', index=False)
    print(f"Imported sheet '{sheet_name}' into table '{table_name}'")
```

## Converting Excel to CSV First

An alternative is to convert Excel to CSV first, then use `LOAD DATA INFILE`:

```bash
python3 -c "
import pandas as pd
pd.read_excel('employees.xlsx').to_csv('/tmp/employees.csv', index=False)
"

mysql -u root -p mydb -e "
LOAD DATA INFILE '/tmp/employees.csv'
INTO TABLE employees
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '\"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
"
```

## Summary

The most flexible way to import Excel files into MySQL is using Python's pandas library with SQLAlchemy, which handles type inference, date parsing, and chunked inserts automatically. Use openpyxl directly when you need precise control over cell-level data transformation. For recurring imports, wrap the script in a scheduled job with error logging.
