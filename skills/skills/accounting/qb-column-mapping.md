# QuickBooks Excel Export – Column Mapping Reference

This document defines how exported QuickBooks Excel data maps to accounting workflows.
It is a reference file and must be consulted before executing accounting skills.

---

## General Ledger Export

Columns:
- Date → Transaction date
- Transaction Type → Voucher / Journal / Payment type
- Document No → Voucher or reference number
- Name → Payee / Customer / Vendor
- Memo → Transaction narration
- Account → Ledger account head
- Debit → Debit amount
- Credit → Credit amount
- Balance → Running balance

Used in:
- Monthly Bookkeeping Review
- Bank Reconciliation

---

## Bank Statement Export

Columns:
- Date → Bank transaction date
- Description → Bank narration
- Reference → Cheque no / transaction reference
- Debit → Bank debit
- Credit → Bank credit
- Balance → Bank running balance

Used in:
- Bank Reconciliation
- Opening / Closing balance checks

---

## Voucher Register Export

Columns:
- Voucher No → Voucher reference
- Date → Voucher date
- Payee → Recipient
- Payment Mode → Cash / Cheque / Bank
- Cheque No → Instrument number
- Tax Amount → Withholding or sales tax
- Net Amount → Amount paid
- Attachment Available → Voucher evidence

Used in:
- Monthly Bookkeeping Review
- Voucher detail verification

---

## Invoices & Payments Export

Columns:
- Invoice No
- Invoice Date
- Customer
- Invoice Amount
- Tax
- Amount Received
- Balance Outstanding
- Payment Reference

Used in:
- Income-side verification
- Recovery follow-ups
