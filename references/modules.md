# ERPNext Modules Reference

## Module Structure & Key DocTypes

### Selling Module

**Masters:**
- `Customer` - Party master for sales transactions
- `Customer Group` - Hierarchical grouping (tree structure)
- `Territory` - Geographic classification
- `Sales Person` - Commission tracking hierarchy

**Transactions:**
| DocType | Abbr | Purpose | Child Tables |
|---------|------|---------|--------------|
| Quotation | QTN | Sales proposal | Quotation Item |
| Sales Order | SO | Confirmed order | Sales Order Item |
| Delivery Note | DN | Stock delivery | Delivery Note Item |
| Sales Invoice | SINV | Billing document | Sales Invoice Item |

**Linking Pattern:**
```python
# Quotation → Sales Order
from erpnext.selling.doctype.quotation.quotation import make_sales_order

# Sales Order → Delivery Note
from erpnext.selling.doctype.sales_order.sales_order import make_delivery_note

# Sales Order → Sales Invoice
from erpnext.selling.doctype.sales_order.sales_order import make_sales_invoice

# Delivery Note → Sales Invoice
from erpnext.stock.doctype.delivery_note.delivery_note import make_sales_invoice
```

### Buying Module

**Masters:**
- `Supplier` - Party master for purchases
- `Supplier Group` - Categorization
- `Supplier Scorecard` - Performance tracking

**Transactions:**
| DocType | Abbr | Purpose | Child Tables |
|---------|------|---------|--------------|
| Material Request | MR | Internal requisition | Material Request Item |
| Request for Quotation | RFQ | Supplier inquiry | RFQ Item, RFQ Supplier |
| Supplier Quotation | SQT | Supplier response | Supplier Quotation Item |
| Purchase Order | PO | Confirmed order | Purchase Order Item |
| Purchase Receipt | PR | Goods receipt | Purchase Receipt Item |
| Purchase Invoice | PINV | Supplier bill | Purchase Invoice Item |

**Linking Pattern:**
```python
# Material Request → Purchase Order
from erpnext.stock.doctype.material_request.material_request import make_purchase_order

# Purchase Order → Purchase Receipt
from erpnext.buying.doctype.purchase_order.purchase_order import make_purchase_receipt

# Purchase Order → Purchase Invoice
from erpnext.buying.doctype.purchase_order.purchase_order import make_purchase_invoice
```

### Stock Module

**Masters:**
- `Item` - Product/Service master
- `Item Group` - Hierarchical categorization
- `Warehouse` - Storage locations (tree structure)
- `Brand`, `Manufacturer` - Product metadata
- `UOM` - Units of measure
- `Price List` - Pricing tiers

**Item Fields:**
```python
{
    'item_code': 'Required, unique identifier',
    'item_name': 'Display name',
    'item_group': 'Link to Item Group',
    'stock_uom': 'Default unit',
    'is_stock_item': '1 for inventory, 0 for service',
    'is_sales_item': 'Can be sold',
    'is_purchase_item': 'Can be purchased',
    'valuation_method': 'FIFO or Moving Average',
    'has_batch_no': 'Batch tracking',
    'has_serial_no': 'Serial number tracking',
    'has_variants': 'Template item',
    'variant_of': 'Link to template'
}
```

**Stock Entry Types:**
| Type | From Warehouse | To Warehouse | Use Case |
|------|---------------|--------------|----------|
| Material Receipt | - | Yes | Opening stock, free receipts |
| Material Issue | Yes | - | Write-offs, consumption |
| Material Transfer | Yes | Yes | Warehouse to warehouse |
| Material Transfer for Manufacture | Yes | WIP | Raw material to production |
| Manufacture | WIP | FG | Finished goods production |
| Repack | Yes | Yes | Repackaging items |
| Send to Subcontractor | Yes | Supplier | Subcontracting |

### Manufacturing Module

**Masters:**
- `Workstation` - Production equipment
- `Workstation Type` - Equipment categories
- `Operation` - Manufacturing steps
- `Routing` - Operation sequences
- `BOM (Bill of Materials)` - Product structure

**BOM Structure:**
```python
bom = frappe.get_doc({
    'doctype': 'BOM',
    'item': 'FG-001',  # Finished good
    'quantity': 1,
    'is_active': 1,
    'is_default': 1,
    'items': [  # Raw materials
        {'item_code': 'RM-001', 'qty': 2, 'rate': 50},
        {'item_code': 'RM-002', 'qty': 1, 'rate': 100}
    ],
    'operations': [  # Manufacturing steps
        {
            'operation': 'Assembly',
            'workstation': 'Assembly Line 1',
            'time_in_mins': 30
        }
    ]
})
```

**Production Flow:**
```
Production Plan → Work Order → Job Card → Stock Entry (Manufacture)
```

### Accounts Module

**Masters:**
- `Account` - Chart of Accounts (tree)
- `Cost Center` - Cost tracking (tree)
- `Mode of Payment` - Payment methods
- `Payment Terms Template` - Payment conditions
- `Tax Category`, `Item Tax Template` - Tax configuration

**Transactions:**
| DocType | Purpose | GL Impact |
|---------|---------|-----------|
| Sales Invoice | Customer billing | Dr Debtors, Cr Income |
| Purchase Invoice | Supplier bills | Dr Expense/Stock, Cr Creditors |
| Payment Entry | Cash/Bank transactions | Dr/Cr Bank, Cr/Dr Party |
| Journal Entry | Manual GL entries | Custom entries |

**Account Types:**
- Asset, Liability, Income, Expense, Equity
- Receivable, Payable (party-linked)
- Bank, Cash (payment accounts)
- Stock (inventory valuation)

### HR Module

**Masters:**
- `Employee` - Employee records
- `Department` - Organizational units
- `Designation` - Job titles
- `Employment Type` - Full-time, Part-time, Contract
- `Leave Type` - Leave categories
- `Salary Component` - Pay elements

**Transactions:**
| DocType | Purpose |
|---------|---------|
| Leave Application | Employee leave requests |
| Attendance | Daily attendance tracking |
| Salary Slip | Monthly payroll |
| Payroll Entry | Bulk salary processing |
| Expense Claim | Employee reimbursements |

### Projects Module

**DocTypes:**
- `Project` - Project master
- `Task` - Work items (hierarchical)
- `Timesheet` - Time tracking
- `Activity Type` - Work categories
- `Project Template` - Reusable project structures

**Project Costing:**
```python
# Get project profitability
from erpnext.projects.doctype.project.project import get_project_info

# Projects can be linked to:
# - Sales Orders (customer projects)
# - Tasks
# - Timesheets
# - Purchase Invoices (expenses)
```

## Document Naming Series

```python
# Common naming patterns (configure in Naming Series)
{
    'Sales Invoice': 'SINV-.YYYY.-',     # SINV-2024-00001
    'Purchase Invoice': 'PINV-.YYYY.-',
    'Sales Order': 'SO-.YYYY.-',
    'Purchase Order': 'PO-.YYYY.-',
    'Delivery Note': 'DN-.YYYY.-',
    'Stock Entry': 'STE-.YYYY.-',
    'Journal Entry': 'JV-.YYYY.-',
    'Payment Entry': 'PE-.YYYY.-'
}
```

## Multi-Company Support

```python
# ERPNext supports multiple companies
# Each transaction must specify company

# Company-specific accounts use abbreviation suffix
# e.g., "Debtors - MC" for company "My Company (MC)"

# Get default company for user
default_company = frappe.defaults.get_user_default('company')
```

## Permissions Model

ERPNext uses role-based permissions:

| Role | Typical Access |
|------|---------------|
| Sales User | Quotation, Sales Order, Customer |
| Sales Manager | Above + approve, reports |
| Purchase User | PO, Supplier, Material Request |
| Stock User | Stock Entry, Delivery Note |
| Accounts User | Invoices, Payment Entry |
| Manufacturing User | Work Order, BOM, Job Card |
| HR User | Employee, Leave, Attendance |

```python
# Check permission
if frappe.has_permission('Sales Order', 'submit', doc=so):
    so.submit()
```
