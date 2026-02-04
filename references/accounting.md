# ERPNext Accounting Reference

## Chart of Accounts Structure

ERPNext uses a hierarchical chart of accounts (tree structure):

```
Assets
├── Current Assets
│   ├── Accounts Receivable
│   │   └── Debtors
│   ├── Bank Accounts
│   │   └── Primary Bank
│   ├── Cash In Hand
│   │   └── Cash
│   └── Stock Assets
│       ├── Stock In Hand
│       └── Stock Received But Not Billed
├── Fixed Assets
│   └── Furniture and Equipment
└── Investments

Liabilities
├── Current Liabilities
│   ├── Accounts Payable
│   │   └── Creditors
│   └── Duties and Taxes
│       └── Tax Payable
└── Long-term Liabilities

Equity
├── Capital Stock
└── Retained Earnings

Income
├── Direct Income
│   └── Sales
└── Indirect Income

Expenses
├── Direct Expenses
│   ├── Cost of Goods Sold
│   └── Stock Adjustment
└── Indirect Expenses
    ├── Administrative Expenses
    └── Depreciation
```

## Account Types

| Account Type | Purpose | Required For |
|--------------|---------|--------------|
| Receivable | Customer outstanding | Party accounting |
| Payable | Supplier outstanding | Party accounting |
| Bank | Bank transactions | Payment Entry |
| Cash | Cash transactions | Payment Entry, POS |
| Stock | Inventory valuation | Perpetual inventory |
| Tax | Tax collection/payment | Tax configuration |
| Cost of Goods Sold | COGS tracking | Stock valuation |
| Depreciation | Asset depreciation | Fixed assets |
| Round Off | Rounding adjustments | Invoice rounding |

## Creating Accounts

```python
# Create account programmatically
account = frappe.get_doc({
    'doctype': 'Account',
    'account_name': 'Marketing Expenses',
    'parent_account': 'Indirect Expenses - MC',
    'company': 'My Company',
    'root_type': 'Expense',
    'report_type': 'Profit and Loss',
    'account_type': '',  # Leave blank for regular accounts
    'is_group': 0
})
account.insert()
```

## GL Entry Patterns

### Sales Invoice GL Entries

```
On Submit:
  Dr Debtors (Receivable)           1,180
    Cr Sales (Income)                       1,000
    Cr Output Tax (Tax)                       180

If Update Stock enabled:
  Dr Cost of Goods Sold (Expense)     700
    Cr Stock In Hand (Stock)                  700
```

### Purchase Invoice GL Entries

```
On Submit (Stock Item):
  Dr Stock Received But Not Billed    800
  Dr Input Tax (Tax)                  144
    Cr Creditors (Payable)                    944

On Submit (Expense Item):
  Dr Expense Account                  800
  Dr Input Tax (Tax)                  144
    Cr Creditors (Payable)                    944
```

### Purchase Receipt GL Entries

```
On Submit:
  Dr Stock In Hand                    800
    Cr Stock Received But Not Billed          800
```

### Payment Entry GL Entries

```
Receive from Customer:
  Dr Bank/Cash                      1,180
    Cr Debtors                              1,180

Pay to Supplier:
  Dr Creditors                        944
    Cr Bank/Cash                              944
```

### Stock Entry GL Entries

```
Material Receipt:
  Dr Stock In Hand                  1,000
    Cr Stock Adjustment                     1,000

Material Issue:
  Dr Stock Adjustment               1,000
    Cr Stock In Hand                        1,000

Manufacture:
  Dr Stock In Hand (FG)             1,500
    Cr Stock In Hand (RM)                   1,500
```

## Journal Entry Types

```python
# Available voucher types
voucher_types = [
    'Journal Entry',        # General purpose
    'Bank Entry',          # Bank transactions
    'Cash Entry',          # Cash transactions
    'Credit Card Entry',   # Credit card
    'Debit Note',          # Supplier credit
    'Credit Note',         # Customer credit
    'Contra Entry',        # Bank to cash
    'Excise Entry',        # Excise duty
    'Write Off Entry',     # Bad debt write-off
    'Opening Entry',       # Opening balances
    'Depreciation Entry',  # Asset depreciation
    'Exchange Rate Revaluation'
]
```

## Multi-Currency Handling

```python
# Transaction in foreign currency
si = frappe.get_doc({
    'doctype': 'Sales Invoice',
    'customer': 'Foreign Customer',
    'currency': 'EUR',
    'conversion_rate': 1.1,  # EUR to base currency (USD)
    'items': [
        {
            'item_code': 'ITEM-001',
            'qty': 10,
            'rate': 100  # EUR
        }
    ]
})
# base_grand_total will be calculated as grand_total * conversion_rate
```

## Tax Configuration

### Tax Template Setup

```python
# Sales Taxes and Charges Template
tax_template = frappe.get_doc({
    'doctype': 'Sales Taxes and Charges Template',
    'title': 'Standard Tax',
    'company': 'My Company',
    'taxes': [
        {
            'charge_type': 'On Net Total',
            'account_head': 'Output Tax - MC',
            'description': 'VAT 18%',
            'rate': 18
        }
    ]
})
```

### Tax Calculation Types

| Charge Type | Calculation |
|-------------|-------------|
| On Net Total | % of net total |
| On Previous Row Total | % of previous row |
| On Previous Row Amount | % of previous row amount |
| Actual | Fixed amount |
| On Item Quantity | Amount × quantity |

### Item Tax Template

```python
# Different tax rates per item
item_tax = frappe.get_doc({
    'doctype': 'Item Tax Template',
    'title': 'Reduced Rate Items',
    'taxes': [
        {
            'tax_type': 'Output Tax - MC',
            'tax_rate': 5  # Override standard rate
        }
    ]
})

# Assign to item
item.taxes = [{'item_tax_template': 'Reduced Rate Items'}]
```

## Payment Terms

```python
# Payment Terms Template
template = frappe.get_doc({
    'doctype': 'Payment Terms Template',
    'template_name': 'Net 30',
    'terms': [
        {
            'payment_term': 'Net 30',
            'invoice_portion': 100,
            'credit_days': 30
        }
    ]
})

# Split payment terms
split_terms = {
    'terms': [
        {'payment_term': '50% Advance', 'invoice_portion': 50, 'credit_days': 0},
        {'payment_term': '50% on Delivery', 'invoice_portion': 50, 'credit_days': 30}
    ]
}
```

## Cost Center Accounting

```python
# Cost Center structure (tree)
"""
Main - MC
├── Sales - MC
│   ├── Region A - MC
│   └── Region B - MC
├── Administration - MC
└── Production - MC
"""

# Transactions can be tagged with cost center
si_item = {
    'item_code': 'ITEM-001',
    'qty': 10,
    'rate': 100,
    'cost_center': 'Sales - MC'
}
```

## Period Closing

```python
# Close accounting period
closing = frappe.get_doc({
    'doctype': 'Period Closing Voucher',
    'posting_date': '2024-12-31',
    'fiscal_year': '2024',
    'company': 'My Company',
    'closing_account_head': 'Retained Earnings - MC'
})
closing.insert()
closing.submit()
```

## Financial Reports

### Trial Balance
```python
from erpnext.accounts.report.trial_balance.trial_balance import execute

filters = {
    'company': 'My Company',
    'from_date': '2024-01-01',
    'to_date': '2024-12-31'
}
columns, data = execute(filters)
```

### General Ledger
```python
from erpnext.accounts.report.general_ledger.general_ledger import execute

filters = {
    'company': 'My Company',
    'from_date': '2024-01-01',
    'to_date': '2024-12-31',
    'account': 'Debtors - MC'
}
columns, data = execute(filters)
```

### Accounts Receivable
```python
from erpnext.accounts.report.accounts_receivable.accounts_receivable import execute

filters = {
    'company': 'My Company',
    'ageing_based_on': 'Due Date',
    'range1': 30,
    'range2': 60,
    'range3': 90,
    'range4': 120
}
columns, data = execute(filters)
```

## Budget Management

```python
# Create budget
budget = frappe.get_doc({
    'doctype': 'Budget',
    'budget_against': 'Cost Center',
    'cost_center': 'Marketing - MC',
    'fiscal_year': '2024',
    'company': 'My Company',
    'accounts': [
        {
            'account': 'Marketing Expenses - MC',
            'budget_amount': 100000
        }
    ],
    'action_if_exceeded': 'Warn'  # or 'Stop'
})
```

## Common Accounting Queries

```python
# Get account balance
from erpnext.accounts.utils import get_balance_on

balance = get_balance_on(
    account='Cash - MC',
    date=frappe.utils.today()
)

# Get party balance
from erpnext.accounts.utils import get_balance_on

customer_balance = get_balance_on(
    party_type='Customer',
    party='Customer Name',
    date=frappe.utils.today()
)

# Get fiscal year
from erpnext.accounts.utils import get_fiscal_year

fiscal_year = get_fiscal_year(
    date=frappe.utils.today(),
    company='My Company'
)
```
