# How to Export MySQL Data to Excel

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Excel, Export, Python, Pandas

Description: Learn how to export MySQL query results to Excel (.xlsx) files using Python with pandas and openpyxl, including formatting and multi-sheet exports.

---

## Introduction

Exporting MySQL data to Excel is a common requirement for business reporting and data sharing with non-technical stakeholders. Python's pandas library combined with openpyxl makes it easy to generate formatted `.xlsx` files directly from MySQL query results.

## Prerequisites

```bash
pip install pandas openpyxl sqlalchemy mysql-connector-python
```

## Basic Export with pandas

The simplest approach reads a query result into a DataFrame and writes it to Excel:

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine('mysql+mysqlconnector://root:password@localhost/mydb')

df = pd.read_sql('SELECT id, name, email, created_at FROM customers', engine)

df.to_excel('customers.xlsx', sheet_name='Customers', index=False)
print(f"Exported {len(df)} rows to customers.xlsx")
```

## Export Multiple Queries to Different Sheets

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine('mysql+mysqlconnector://root:password@localhost/mydb')

customers_df = pd.read_sql('SELECT * FROM customers', engine)
orders_df = pd.read_sql('SELECT * FROM orders WHERE status = "completed"', engine)
products_df = pd.read_sql('SELECT * FROM products', engine)

with pd.ExcelWriter('report.xlsx', engine='openpyxl') as writer:
    customers_df.to_excel(writer, sheet_name='Customers', index=False)
    orders_df.to_excel(writer, sheet_name='Orders', index=False)
    products_df.to_excel(writer, sheet_name='Products', index=False)

print("Exported 3 sheets to report.xlsx")
```

## Adding Formatting with openpyxl

Apply column widths, bold headers, and number formats:

```python
import pandas as pd
from sqlalchemy import create_engine
from openpyxl import load_workbook
from openpyxl.styles import Font, PatternFill, Alignment
from openpyxl.utils import get_column_letter

engine = create_engine('mysql+mysqlconnector://root:password@localhost/mydb')
df = pd.read_sql('SELECT id, name, salary, hire_date FROM employees', engine)

output_path = 'employees_report.xlsx'
df.to_excel(output_path, sheet_name='Employees', index=False)

wb = load_workbook(output_path)
ws = wb['Employees']

# Bold header row with background color
header_fill = PatternFill(start_color='4472C4', end_color='4472C4', fill_type='solid')
for cell in ws[1]:
    cell.font = Font(bold=True, color='FFFFFF')
    cell.fill = header_fill
    cell.alignment = Alignment(horizontal='center')

# Auto-fit column widths
for col in ws.columns:
    max_len = max(len(str(cell.value or '')) for cell in col)
    ws.column_dimensions[get_column_letter(col[0].column)].width = max_len + 4

wb.save(output_path)
print("Formatted Excel file saved")
```

## Exporting Large Datasets in Chunks

For tables with millions of rows, read in chunks to avoid memory issues:

```python
import pandas as pd
from sqlalchemy import create_engine
from openpyxl import Workbook

engine = create_engine('mysql+mysqlconnector://root:password@localhost/mydb')

wb = Workbook()
ws = wb.active
ws.title = 'Transactions'
headers_written = False

query = 'SELECT id, customer_id, amount, txn_date FROM transactions'
for chunk in pd.read_sql(query, engine, chunksize=10000):
    if not headers_written:
        ws.append(list(chunk.columns))
        headers_written = True
    for row in chunk.itertuples(index=False):
        ws.append(list(row))

wb.save('transactions.xlsx')
print("Large export complete")
```

## Using mysql Client with CSV as Intermediate Format

```bash
mysql -u root -p mydb \
  --batch -e "SELECT id, name, email FROM customers" \
  | sed 's/\t/,/g' > /tmp/customers.csv

python3 -c "
import pandas as pd
pd.read_csv('/tmp/customers.csv').to_excel('customers.xlsx', index=False)
"
```

## Summary

pandas with openpyxl is the recommended stack for exporting MySQL data to Excel. Use `pd.read_sql` with a SQLAlchemy engine for querying, `to_excel` with `ExcelWriter` for multi-sheet reports, and openpyxl's formatting API when the output needs polish for business stakeholders. For very large tables, use chunked reads to stay within memory limits.
