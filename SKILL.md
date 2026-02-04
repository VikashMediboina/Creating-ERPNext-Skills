---
name: erpnext
description: Complete guide for ERPNext ERP development and customization. Use this skill when working with ERPNext business modules (Accounting, Sales, Buying, Stock, Manufacturing, HR, Projects), creating business transactions (Sales Orders, Purchase Invoices, Stock Entries, Journal Entries), implementing business workflows (Quote-to-Cash, Procure-to-Pay, Manufacturing cycles), building custom ERPNext apps, or integrating with ERPNext via REST API. Triggers include mentions of "ERPNext", "Sales Invoice", "Purchase Order", "Stock Entry", "BOM", "Work Order", "Chart of Accounts", "Customer", "Supplier", "Item master", "Warehouse", "Journal Entry", "Payment Entry", or any ERP business transaction terminology.
---

# ERPNext Development Guide

ERPNext is a full-featured open-source ERP built on the Frappe Framework. This skill covers ERPNext-specific modules, business workflows, and patterns. For underlying Frappe Framework concepts (DocTypes, controllers, hooks, bench commands), see the frappe-framework skill.

## Core Modules Overview

| Module | Key DocTypes | Purpose |
|--------|-------------|---------|
| **Selling** | Customer, Quotation, Sales Order, Delivery Note | Quote-to-cash cycle |
| **Buying** | Supplier, Request for Quotation, Purchase Order, Purchase Receipt | Procure-to-pay cycle |
| **Accounts** | Sales Invoice, Purchase Invoice, Payment Entry, Journal Entry | Financial transactions |
| **Stock** | Item, Warehouse, Stock Entry, Stock Reconciliation | Inventory management |
| **Manufacturing** | BOM, Work Order, Job Card, Workstation | Production planning |
| **HR** | Employee, Leave Application, Salary Slip, Payroll Entry | Human resources |
| **Projects** | Project, Task, Timesheet | Project management |
| **Assets** | Asset, Asset Category, Asset Depreciation | Fixed asset management |

## Business Transaction Workflows

### Quote-to-Cash (Sales Cycle)

```
Lead → Opportunity → Quotation → Sales Order → Delivery Note → Sales Invoice → Payment Entry
```

```python
# Create Sales Order from Quotation
from erpnext.selling.doctype.quotation.quotation import make_sales_order

sales_order = make_sales_order(quotation.name)
sales_order.insert()
sales_order.submit()

# Create Delivery Note from Sales Order  
from erpnext.selling.doctype.sales_order.sales_order import make_delivery_note

delivery = make_delivery_note(sales_order.name)
delivery.insert()
delivery.submit()

# Create Sales Invoice from Sales Order
from erpnext.selling.doctype.sales_order.sales_order import make_sales_invoice

invoice = make_sales_invoice(sales_order.name)
invoice.insert()
invoice.submit()
```

### Procure-to-Pay (Purchase Cycle)

```
Material Request → Supplier Quotation → Purchase Order → Purchase Receipt → Purchase Invoice → Payment Entry
```

```python
# Create Purchase Order
po = frappe.get_doc({
    'doctype': 'Purchase Order',
    'supplier': 'Supplier Name',
    'items': [{
        'item_code': 'ITEM-001',
        'qty': 100,
        'rate': 50,
        'schedule_date': frappe.utils.add_days(frappe.utils.today(), 7)
    }]
})
po.insert()
po.submit()

# Create Purchase Receipt from PO
from erpnext.buying.doctype.purchase_order.purchase_order import make_purchase_receipt

receipt = make_purchase_receipt(po.name)
receipt.insert()
receipt.submit()

# Create Purchase Invoice from PO
from erpnext.buying.doctype.purchase_order.purchase_order import make_purchase_invoice

invoice = make_purchase_invoice(po.name)
invoice.insert()
invoice.submit()
```

### Manufacturing Cycle

```
BOM → Production Plan → Work Order → Stock Entry (Material Transfer) → Job Card → Stock Entry (Manufacture)
```

```python
# Create Work Order from BOM
wo = frappe.get_doc({
    'doctype': 'Work Order',
    'production_item': 'FG-001',
    'bom_no': 'BOM-FG-001-001',
    'qty': 10,
    'company': 'My Company'
})
wo.insert()
wo.submit()

# Start Work Order (creates material transfer)
wo.start()

# Complete manufacturing
from erpnext.manufacturing.doctype.work_order.work_order import make_stock_entry

stock_entry = make_stock_entry(wo.name, 'Manufacture', wo.qty)
stock_entry.insert()
stock_entry.submit()
```

## Key DocType Patterns

### Items (Products/Services)

```python
# Stock Item
item = frappe.get_doc({
    'doctype': 'Item',
    'item_code': 'PROD-001',
    'item_name': 'Product One',
    'item_group': 'Products',
    'stock_uom': 'Nos',
    'is_stock_item': 1,
    'valuation_method': 'FIFO',  # or 'Moving Average'
    'default_warehouse': 'Stores - MC'
})

# Service Item (non-stock)
service = frappe.get_doc({
    'doctype': 'Item',
    'item_code': 'SERV-001',
    'item_name': 'Consulting Service',
    'item_group': 'Services',
    'is_stock_item': 0,
    'stock_uom': 'Hour'
})
```

### Stock Entry Types

```python
# Material Receipt (incoming stock)
se = frappe.get_doc({
    'doctype': 'Stock Entry',
    'stock_entry_type': 'Material Receipt',
    'to_warehouse': 'Stores - MC',
    'items': [{'item_code': 'ITEM-001', 'qty': 100, 'basic_rate': 50}]
})

# Material Issue (outgoing stock)
se = frappe.get_doc({
    'doctype': 'Stock Entry',
    'stock_entry_type': 'Material Issue',
    'from_warehouse': 'Stores - MC',
    'items': [{'item_code': 'ITEM-001', 'qty': 10}]
})

# Material Transfer
se = frappe.get_doc({
    'doctype': 'Stock Entry',
    'stock_entry_type': 'Material Transfer',
    'from_warehouse': 'Stores - MC',
    'to_warehouse': 'Finished Goods - MC',
    'items': [{'item_code': 'ITEM-001', 'qty': 50}]
})

# Manufacture (consume raw materials, produce finished goods)
se = frappe.get_doc({
    'doctype': 'Stock Entry',
    'stock_entry_type': 'Manufacture',
    'work_order': 'WO-00001',
    'bom_no': 'BOM-FG-001-001',
    'fg_completed_qty': 10
})
```

### Journal Entries

```python
# Basic Journal Entry
je = frappe.get_doc({
    'doctype': 'Journal Entry',
    'voucher_type': 'Journal Entry',
    'posting_date': frappe.utils.today(),
    'accounts': [
        {
            'account': 'Cash - MC',
            'debit_in_account_currency': 1000
        },
        {
            'account': 'Sales - MC',
            'credit_in_account_currency': 1000
        }
    ]
})
je.insert()
je.submit()
```

### Payment Entry

```python
# Receive payment from customer
from erpnext.accounts.doctype.payment_entry.payment_entry import get_payment_entry

pe = get_payment_entry('Sales Invoice', 'SINV-00001')
pe.insert()
pe.submit()

# Manual payment entry
pe = frappe.get_doc({
    'doctype': 'Payment Entry',
    'payment_type': 'Receive',
    'party_type': 'Customer',
    'party': 'Customer Name',
    'paid_amount': 1000,
    'received_amount': 1000,
    'paid_to': 'Cash - MC',
    'references': [{
        'reference_doctype': 'Sales Invoice',
        'reference_name': 'SINV-00001',
        'allocated_amount': 1000
    }]
})
```

## Common API Patterns

### Get Customer/Supplier Details

```python
from erpnext.accounts.party import get_party_details

# Get customer details for transactions
details = get_party_details(
    party='Customer Name',
    party_type='Customer',
    posting_date=frappe.utils.today()
)
# Returns: address, contact, price_list, tax_template, etc.
```

### Get Item Details

```python
from erpnext.stock.get_item_details import get_item_details

args = {
    'item_code': 'ITEM-001',
    'company': 'My Company',
    'doctype': 'Sales Invoice',
    'customer': 'Customer Name'
}
details = get_item_details(args)
# Returns: description, stock_uom, price_list_rate, warehouse, etc.
```

### Check Stock Availability

```python
from erpnext.stock.utils import get_stock_balance

qty = get_stock_balance(
    item_code='ITEM-001',
    warehouse='Stores - MC',
    posting_date=frappe.utils.today()
)

# Get with valuation
from erpnext.stock.utils import get_stock_value_and_qty
value, qty = get_stock_value_and_qty(
    item_code='ITEM-001',
    warehouse='Stores - MC'
)
```

### Price List Queries

```python
from erpnext.stock.get_item_details import get_price_list_rate_for

rate = get_price_list_rate_for({
    'price_list': 'Standard Selling',
    'item_code': 'ITEM-001'
})
```

## Accounting Integration

### General Ledger Entries

ERPNext automatically creates GL entries on document submission:

| Document | Debit | Credit |
|----------|-------|--------|
| Sales Invoice | Debtors | Income Account |
| Purchase Invoice | Expense/Stock | Creditors |
| Payment Entry (Receive) | Bank/Cash | Debtors |
| Payment Entry (Pay) | Creditors | Bank/Cash |
| Stock Entry (Receipt) | Stock In Hand | Stock Received Not Billed |

### Custom GL Entries

```python
from erpnext.accounts.general_ledger import make_gl_entries

gl_entries = []
gl_entries.append({
    'account': 'Debtors - MC',
    'party_type': 'Customer',
    'party': 'Customer Name',
    'debit': 1000,
    'debit_in_account_currency': 1000,
    'voucher_type': 'Sales Invoice',
    'voucher_no': 'SINV-00001',
    'posting_date': frappe.utils.today()
})
# ... add credit entry

make_gl_entries(gl_entries)
```

## Document Status Workflow

Most transaction documents follow:

```
Draft (docstatus=0) → Submitted (docstatus=1) → Cancelled (docstatus=2)
```

```python
# Submit document
doc.submit()  # Changes docstatus to 1

# Cancel document
doc.cancel()  # Changes docstatus to 2

# Amend cancelled document
amended = frappe.copy_doc(doc)
amended.amended_from = doc.name
amended.insert()
```

## ERPNext Settings

### Company Setup

```python
company = frappe.get_doc({
    'doctype': 'Company',
    'company_name': 'My Company',
    'abbr': 'MC',
    'default_currency': 'USD',
    'country': 'United States',
    'default_warehouse_for_sales_return': 'Stores - MC'
})
```

### Stock Settings

```python
# Get stock settings
settings = frappe.get_single('Stock Settings')
settings.auto_insert_price_list_rate_if_missing = 1
settings.allow_negative_stock = 0
settings.save()
```

### Accounts Settings

```python
settings = frappe.get_single('Accounts Settings')
settings.allow_cost_center_in_entry_of_bs_account = 1
settings.save()
```

## REST API for ERPNext

### Create Sales Order via API

```bash
# Create Sales Order
curl -X POST https://erpnext.example.com/api/resource/Sales%20Order \
  -H "Authorization: token api_key:api_secret" \
  -H "Content-Type: application/json" \
  -d '{
    "customer": "Customer Name",
    "delivery_date": "2024-12-31",
    "items": [
      {"item_code": "ITEM-001", "qty": 10, "rate": 100}
    ]
  }'
```

### Submit Document via API

```bash
# Submit document (change docstatus)
curl -X PUT https://erpnext.example.com/api/resource/Sales%20Order/SO-00001 \
  -H "Authorization: token api_key:api_secret" \
  -H "Content-Type: application/json" \
  -d '{"docstatus": 1}'
```

### Call Whitelisted Method

```bash
# Get item stock
curl -X POST https://erpnext.example.com/api/method/erpnext.stock.utils.get_stock_balance \
  -H "Authorization: token api_key:api_secret" \
  -d "item_code=ITEM-001&warehouse=Stores - MC"
```

## Common Customization Patterns

### Custom Validation on Sales Order

```python
# In custom app: custom_app/hooks.py
doc_events = {
    "Sales Order": {
        "validate": "custom_app.overrides.sales_order.validate_custom_fields"
    }
}

# In custom_app/overrides/sales_order.py
def validate_custom_fields(doc, method):
    if doc.custom_field and doc.grand_total > 10000:
        frappe.throw("Orders over 10,000 require approval")
```

### Auto-Create Follow-up Document

```python
# Auto-create Delivery Note on Sales Order submit
doc_events = {
    "Sales Order": {
        "on_submit": "custom_app.overrides.sales_order.create_delivery_note"
    }
}

def create_delivery_note(doc, method):
    from erpnext.selling.doctype.sales_order.sales_order import make_delivery_note
    dn = make_delivery_note(doc.name)
    dn.insert()
```

## References

For detailed module documentation, see:
- [references/modules.md](references/modules.md) - Core module details and DocType relationships
- [references/accounting.md](references/accounting.md) - Chart of Accounts, GL entries, tax setup
- [references/stock.md](references/stock.md) - Inventory valuation, serial/batch, warehouses

Official documentation: https://docs.frappe.io/erpnext
