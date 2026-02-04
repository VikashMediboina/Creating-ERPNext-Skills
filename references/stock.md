# ERPNext Stock Reference

## Inventory Valuation Methods

### FIFO (First In, First Out)

- Cost of oldest stock used first
- Actual cost tracking per batch
- More accurate for perishables

```python
# Item with FIFO valuation
item = frappe.get_doc({
    'doctype': 'Item',
    'item_code': 'FIFO-ITEM',
    'valuation_method': 'FIFO'
})
```

### Moving Average

- Weighted average cost recalculated on each receipt
- Simpler, smooths out price fluctuations

```python
# Moving Average calculation
new_avg_rate = (existing_qty * existing_rate + incoming_qty * incoming_rate) / (existing_qty + incoming_qty)
```

## Stock Ledger Entry

Every stock transaction creates Stock Ledger Entries (SLE):

```python
# SLE fields
{
    'item_code': 'ITEM-001',
    'warehouse': 'Stores - MC',
    'posting_date': '2024-01-15',
    'posting_time': '14:30:00',
    'voucher_type': 'Stock Entry',
    'voucher_no': 'STE-00001',
    'actual_qty': 100,  # Positive for in, negative for out
    'qty_after_transaction': 150,
    'valuation_rate': 50.5,
    'stock_value': 7575,
    'incoming_rate': 50.5
}
```

## Warehouse Management

### Warehouse Structure (Tree)

```
All Warehouses - MC
├── Stores - MC
│   ├── Raw Materials - MC
│   └── Packing Materials - MC
├── Work In Progress - MC
├── Finished Goods - MC
└── Rejected Items - MC
```

### Warehouse Properties

```python
warehouse = frappe.get_doc({
    'doctype': 'Warehouse',
    'warehouse_name': 'Main Store',
    'company': 'My Company',
    'parent_warehouse': 'All Warehouses - MC',
    'is_group': 0,
    'warehouse_type': 'Stores',  # Stores, WIP, Finished Goods, Rejected
    'account': 'Stock In Hand - MC',
    'default_in_transit_warehouse': 'Transit - MC'
})
```

## Serial Number Tracking

```python
# Enable on Item
item.has_serial_no = 1
item.serial_no_series = 'SN-.#####'

# Create serials on receipt
se = frappe.get_doc({
    'doctype': 'Stock Entry',
    'stock_entry_type': 'Material Receipt',
    'items': [{
        'item_code': 'SERIAL-ITEM',
        'qty': 3,
        'serial_no': 'SN-00001\nSN-00002\nSN-00003',  # Newline separated
        'basic_rate': 100
    }]
})

# Query serial status
serials = frappe.db.get_all('Serial No',
    filters={'item_code': 'SERIAL-ITEM', 'warehouse': 'Stores - MC'},
    fields=['name', 'status', 'purchase_rate']
)
```

## Batch Number Tracking

```python
# Enable on Item
item.has_batch_no = 1
item.create_new_batch = 1
item.batch_number_series = 'BATCH-.#####'

# Batch fields
batch = frappe.get_doc({
    'doctype': 'Batch',
    'item': 'BATCH-ITEM',
    'batch_id': 'BATCH-00001',
    'expiry_date': '2024-12-31',
    'manufacturing_date': '2024-01-01'
})

# Stock Entry with batch
se = frappe.get_doc({
    'doctype': 'Stock Entry',
    'stock_entry_type': 'Material Receipt',
    'items': [{
        'item_code': 'BATCH-ITEM',
        'qty': 100,
        'batch_no': 'BATCH-00001',
        'basic_rate': 50
    }]
})
```

## Stock Queries

### Get Stock Balance

```python
from erpnext.stock.utils import get_stock_balance

qty = get_stock_balance(
    item_code='ITEM-001',
    warehouse='Stores - MC',
    posting_date=frappe.utils.today(),
    posting_time='23:59:59'
)
```

### Get Stock Value

```python
from erpnext.stock.utils import get_stock_value_on

value = get_stock_value_on(
    warehouse='Stores - MC',
    posting_date=frappe.utils.today()
)
```

### Get Batch Balance

```python
from erpnext.stock.doctype.batch.batch import get_batch_qty

qty = get_batch_qty(
    batch_no='BATCH-00001',
    warehouse='Stores - MC'
)
```

### Get Serial Numbers

```python
# Get available serials
serials = frappe.db.get_all('Serial No',
    filters={
        'item_code': 'SERIAL-ITEM',
        'warehouse': 'Stores - MC',
        'status': 'Active'
    },
    pluck='name'
)
```

## Stock Reconciliation

For adjusting stock quantities/values:

```python
recon = frappe.get_doc({
    'doctype': 'Stock Reconciliation',
    'purpose': 'Stock Reconciliation',  # or 'Opening Stock'
    'items': [{
        'item_code': 'ITEM-001',
        'warehouse': 'Stores - MC',
        'qty': 150,  # Set actual quantity
        'valuation_rate': 50  # Set valuation rate
    }]
})
recon.insert()
recon.submit()
```

## Reorder Level

```python
# Set on Item
item = frappe.get_doc('Item', 'ITEM-001')
item.append('reorder_levels', {
    'warehouse': 'Stores - MC',
    'warehouse_reorder_level': 100,  # Trigger reorder
    'warehouse_reorder_qty': 500,    # Order this much
    'material_request_type': 'Purchase'  # or 'Transfer', 'Manufacture'
})
item.save()

# Auto-create Material Request
from erpnext.stock.reorder_item import reorder_item

reorder_item()  # Called by scheduler
```

## Projected Quantity

```python
from erpnext.stock.dashboard.item_dashboard import get_data

data = get_data(
    item_code='ITEM-001',
    warehouse='Stores - MC'
)
# Returns: actual_qty, planned_qty, reserved_qty, ordered_qty, projected_qty
```

**Projected Qty Formula:**
```
Projected Qty = Actual Qty 
              + Planned Qty (Work Orders)
              + Ordered Qty (Purchase Orders)
              - Reserved Qty (Sales Orders)
              - Reserved for Production
              - Reserved for Subcontract
```

## Landed Cost Voucher

Add additional costs to purchase receipts:

```python
lcv = frappe.get_doc({
    'doctype': 'Landed Cost Voucher',
    'purchase_receipts': [{
        'receipt_document_type': 'Purchase Receipt',
        'receipt_document': 'PR-00001',
        'supplier': 'Supplier Name',
        'grand_total': 10000
    }],
    'taxes': [{
        'expense_account': 'Freight - MC',
        'description': 'Shipping charges',
        'amount': 500
    }],
    'distribute_charges_based_on': 'Amount'  # or 'Qty'
})
lcv.insert()
lcv.submit()
# This updates the valuation of items in the purchase receipt
```

## Putaway Rules

Automatic warehouse assignment:

```python
rule = frappe.get_doc({
    'doctype': 'Putaway Rule',
    'company': 'My Company',
    'item_code': 'ITEM-001',
    'warehouse': 'Bin A - MC',
    'capacity': 1000,
    'stock_uom': 'Nos',
    'priority': 1
})
```

## Pick List

For warehouse picking:

```python
from erpnext.stock.doctype.pick_list.pick_list import create_pick_list

pick_list = create_pick_list(
    source_name='SO-00001'  # Sales Order
)
pick_list.insert()

# Pick list locations are auto-assigned based on:
# - FIFO/batch expiry
# - Putaway rules
# - Shelf location
```

## Stock Reports

### Stock Balance

```python
from erpnext.stock.report.stock_balance.stock_balance import execute

filters = {
    'company': 'My Company',
    'from_date': '2024-01-01',
    'to_date': '2024-12-31',
    'item_code': 'ITEM-001'
}
columns, data = execute(filters)
```

### Stock Ledger

```python
from erpnext.stock.report.stock_ledger.stock_ledger import execute

filters = {
    'company': 'My Company',
    'from_date': '2024-01-01',
    'to_date': '2024-12-31',
    'item_code': 'ITEM-001',
    'warehouse': 'Stores - MC'
}
columns, data = execute(filters)
```

### Stock Ageing

```python
from erpnext.stock.report.stock_ageing.stock_ageing import execute

filters = {
    'company': 'My Company',
    'to_date': '2024-12-31',
    'range1': 30,
    'range2': 60,
    'range3': 90
}
columns, data = execute(filters)
```

## Perpetual Inventory

ERPNext uses perpetual inventory by default:

1. **Stock Receipt** → Creates GL entry for stock value
2. **Stock Issue** → Reduces stock value in GL
3. **Real-time Balance** → Stock and accounts always in sync

```python
# Check if perpetual inventory is enabled
company = frappe.get_doc('Company', 'My Company')
if company.enable_perpetual_inventory:
    # GL entries created automatically
    pass
```

## Common Stock Utilities

```python
# Get latest stock value
from erpnext.stock.utils import get_latest_stock_qty

qty = get_latest_stock_qty(
    item_code='ITEM-001',
    warehouse='Stores - MC'
)

# Get incoming rate
from erpnext.stock.utils import get_incoming_rate

rate = get_incoming_rate({
    'item_code': 'ITEM-001',
    'warehouse': 'Stores - MC',
    'posting_date': frappe.utils.today(),
    'posting_time': '12:00:00',
    'qty': 10
})

# Get bin (warehouse-item combination)
from erpnext.stock.doctype.bin.bin import get_bin

bin = get_bin(
    item_code='ITEM-001',
    warehouse='Stores - MC'
)
# bin.actual_qty, bin.reserved_qty, bin.projected_qty
```
