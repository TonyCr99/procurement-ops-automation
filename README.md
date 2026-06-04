# Procurement Ops Automation

> End-to-end automation of the IT hardware procurement process — from Jira ticket detection to purchase order creation in NetSuite.

![Python](https://img.shields.io/badge/Python-3.12+-blue?logo=python)
![Playwright](https://img.shields.io/badge/Playwright-Chromium-green?logo=playwright)
![Jira](https://img.shields.io/badge/Jira-REST%20API%20v3-0052CC?logo=jira)
![NetSuite](https://img.shields.io/badge/NetSuite-OAuth%20TBA-orange)
![License](https://img.shields.io/badge/License-MIT-lightgrey)
![Status](https://img.shields.io/badge/Status-In%20Development-yellow)

---

## The Problem I Mapped

Before building anything, I documented the existing procurement workflow to identify exactly where time was being lost. The process involved **5 disconnected tools** with no integration between them:

```
Jira → Abasteo (vendor portal) → Jira (again) → NetSuite → Email to Finance → Invoice forwarding
```

Every single handoff was manual. Here are the **6 bottlenecks** I identified:

| # | Bottleneck | Impact |
|---|---|---|
| 1 | Manual quote generation in vendor portal | Variable time per request, no automation |
| 2 | Manager approval with no SLA | Process stalls with zero visibility |
| 3 | Manual PO creation in NetSuite + CC approval with no SLA | Double wait, no tracking |
| 4 | Finance payment calendar sync | Process blocked by fixed payment dates |
| 5 | Price change risk in vendor portal | Can restart the entire flow from scratch |
| 6 | Laptop delivery with no tracking | No visibility between vendor and IT |

**Result: A single hardware request took an average of 3+ hours of manual work across 4 platforms**, with 10–15 tickets per month — roughly 40+ hours of repetitive, low-value effort.

---

## The Solution

This tool monitors Jira in the background, detects hardware procurement tickets, and orchestrates the entire workflow autonomously — eliminating every manual step listed above.

```
Jira Ticket Detected
(New Joiner or Equipment Replacement)
          │
          ▼
  Parse ticket · extract hardware type, tier,
  cost center, approver from ticket fields
          │
          ▼
  Search vendor portal autonomously
  Filter by specs · stock · lowest price
          │
          ▼
  Generate quote PDF via vendor portal
  Attach PDF directly to Jira ticket
          │
          ▼
  Notify approver · Email / Slack / Teams
  SLA tracking · auto-reminder at 24h
          │
          ▼
  On approval → Create Purchase Order in NetSuite
  Notify Finance team automatically
          │
          ▼
  Monitor price changes before payment
  Trigger re-approval if price increases
          │
          ▼
  Confirm delivery · close Jira ticket
```

**A 3-hour manual process reduced to under 5 minutes of automated execution.**

---

## Key Features

- **Autonomous product selection** — filters vendor catalog by chip/processor, RAM, storage, and screen size; then picks the lowest-price option with sufficient stock
- **Two ticket types** — New Joiner (hardware for new employees) and Equipment Replacement, each with different data extraction logic
- **Playwright web scraper** — logs into vendor portal, searches by SKU, adds to cart, generates and downloads quote PDF without human intervention
- **PDF attached to Jira** — quote is uploaded directly to the ticket and temp file is deleted automatically
- **Multi-channel notifications** — Email, Slack, and Microsoft Teams with approval request and reminder templates
- **SLA enforcement** — auto-reminder after 24h of no response on approval requests
- **Price change detection** — compares current vendor price against approved quote before payment; triggers re-approval if price has changed
- **Anti-reprocess guard** — prevents duplicate processing across scheduler restarts
- **Dry run mode** — safe read-only execution to verify behavior before writing to Jira or NetSuite
- **Fully configurable** — hardware specs, tiers, SLA times, and business rules defined in `config.yaml`; no code changes needed to adapt to a different organization

---

## Architecture

```
src/
├── scheduler.py          # Main entry point — detects tickets every 15 min, orchestrates full flow
├── cli.py                # Interactive menu for manual intervention on individual tickets
├── config_loader.py      # Loads config.yaml + .env, exposes global config and env() helper
│
├── connectors/
│   ├── jira.py           # Jira REST API v3 — get tickets, update status, add comments, attach files
│   ├── vendor.py         # Playwright scraper — autonomous product selection & quote PDF generation
│   ├── netsuite.py       # NetSuite OAuth TBA — Purchase Order creation and approval submission
│   └── azure_ad.py       # Microsoft Graph API — user → cost center lookup and approver validation
│
├── modules/
│   ├── ticket_parser.py  # Detects ticket type, extracts standardized fields from Jira custom fields
│   ├── quotes.py         # Orchestrates quote flow: Jira + vendor → approval comment + notification
│   ├── notifications.py  # Email / Slack / Teams with approval_request, reminder, finance_payment templates
│   └── price_validator.py# Compares current vendor price vs approved — triggers re-approval if price rises
│
└── mock/
    ├── jira_tickets.py   # Sample tickets for local testing
    ├── vendor_catalog.py # Mock product catalog by hardware type and tier
    └── netsuite_orders.py# Mock POs with IVA helpers and PO note builder
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.12+ |
| Ticket Management | Jira REST API v3 |
| Web Scraping | Playwright (Chromium) |
| ERP Integration | NetSuite OAuth TBA — requests-oauthlib |
| Identity & Cost Centers | Microsoft Graph API (Azure AD) |
| Notifications | SMTP / Slack Webhooks / Microsoft Teams Webhooks |
| Data Processing | pandas, openpyxl |
| CLI | rich |
| Scheduling | Custom scheduler loop — no external framework |
| Configuration | YAML + .env |

---

## Hardware Selection Logic

Minimum specs per type and tier are defined in `config.yaml`. The vendor connector applies them autonomously:

| Tier | MacBook | Windows |
|---|---|---|
| Standard | M4 · 16GB · 512GB · 13" | Core Ultra 5/7 · 16GB · 512GB · 14" |
| Standard Plus | M4 · 16GB · 512GB · 14" | Core Ultra 7 · 16GB · 512GB · 14" |
| Power | M4 · 16GB · 512GB · 16" | Core Ultra 7 · 32GB · 1TB · 15.6" |

Selection order: **meets specs → stock ≥ threshold → lowest price.**

---

## Getting Started

**Requirements:** Python 3.12+, Playwright

```bash
# Install dependencies
pip install -r requirements.txt
python -m playwright install chromium

# Configure environment
cp .env.example .env
# Edit .env with your credentials

# Run in dry-run mode (read-only — no changes to Jira or NetSuite)
python -m src.scheduler --once

# Run in live mode
python -m src.scheduler --once --live

# Interactive CLI for manual operations
python -m src.cli
```

> All connectors support mock mode (`JIRA_MOCK=true`, `VENDOR_MOCK=true`, etc.) for local development without real credentials.

---

## Business Impact

| Metric | Before | After |
|---|---|---|
| Time per request | ~3 hours | < 5 minutes |
| Monthly tickets (avg) | 10–15 | 10–15 |
| Hours saved per month | — | ~40 hours |
| Tools requiring manual entry | 5 | 0 |
| Audit trail | Inconsistent | Full — every action logged in Jira |
| Price change risk | Undetected | Auto-detected before payment |

---

## Roadmap

- [x] Jira connector — read tickets, update status, add comments, attach files
- [x] Vendor scraper — autonomous product selection, cart management, quote PDF
- [x] NetSuite connector — PO creation and approval submission (mock)
- [x] Azure AD connector — cost center and approver lookup
- [x] Notifications — Email, Slack, Teams
- [x] Price validator — detects price changes before payment
- [x] CLI — interactive menu for manual operations
- [x] Scheduler — background loop with anti-reprocess guard
- [ ] NetSuite live mode — pending admin permissions
- [ ] Azure AD live mode — Graph API validation
- [ ] Web dashboard — real-time ticket and PO status
- [ ] Multi-tenant support

---

## License

MIT © [TonyCr99](https://github.com/TonyCr99)
