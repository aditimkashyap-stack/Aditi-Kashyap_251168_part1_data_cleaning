# Cleaning Log — Retail Orders Dataset

## Overview

This log documents all data quality issues found, cleaning actions taken, business rules applied, assumptions made, and limitations of the cleaning process for the `raw_orders.xlsx` dataset.

---

## 1. Issues Found

### 1.1 Duplicate Records
| Issue | Count | Details |
|---|---|---|
| Exact duplicate rows | 20 | All 21 columns identical |
| Conflicting duplicate order_ids | 12 unique order IDs | Same order_id, different field values |

### 1.2 Missing Values
| Column | Missing Count | % Missing |
|---|---|---|
| region | 26 | 2.8% |
| ship_mode | 22 | 2.4% |
| discount | 18 | 1.9% |

### 1.3 Text/Formatting Issues
| Column | Issues Found |
|---|---|
| segment | Mixed case (Consumer, consumer, SMALL BUSINESS), leading/trailing spaces (e.g. `  Consumer `) |
| region | Mixed case (North, NORTH, north), extra spaces |
| category | Mixed case, double-space variants (e.g. `Office  Supplies`) |
| sub_category | Mixed case, extra spaces |
| ship_mode | Mixed case, double-space variants (e.g. `Standard  Class`) |
| payment_status | Mixed case (Paid, PENDING, failed) |
| order_status | Mixed case (Completed, completed, CANCELLED) |
| customer_name | Minor spacing inconsistencies |

### 1.4 Date Format Issues
- Multiple date formats found across `order_date` and `ship_date`:
  - `21 Jul 2024` (day-month-year text)
  - `08/31/2024` (MM/DD/YYYY)
  - `06/08/2024` (MM/DD/YYYY)
  - `28-11-2024` (DD-MM-YYYY dash separated)
  - `2024-05-24` (ISO format)
- All formats were successfully parsed using `dateutil.parser`

### 1.5 Invalid Date Logic
| Issue | Count |
|---|---|
| Ship date before order date | 94 |

### 1.6 Discount Issues
| Issue | Count |
|---|---|
| Missing discount values | 18 |
| Negative discount values | 15 |
| Discounts > 0.65 (above policy max) | 0 |

### 1.7 Sales/Profit Calculation Mismatches
| Issue | Count |
|---|---|
| Sales value does not match qty × unit_price × (1-discount) | 56 |
| Profit value does not match calculated_sales - cost | 56 |

---

## 2. Cleaning Actions Performed

### 2.1 Text Standardization
- Applied `TRIM` equivalent (`.strip()`) to all text fields to remove leading/trailing whitespace
- Applied `SUBSTITUTE` equivalent (`re.sub(r'\s+', ' ', ...)`) to collapse multiple internal spaces to single space
- Applied `.title()` case normalization to: `customer_name`, `state`, `city`, `sub_category`
- Applied manual mapping dictionaries for controlled vocabulary fields:
  - `segment`: mapped all variants to `Consumer`, `Small Business`, `Corporate`, `Home Office`
  - `region`: mapped all variants to `North`, `South`, `East`, `West`
  - `category`: mapped all variants to `Office Supplies`, `Furniture`, `Technology`
  - `order_status`: mapped to `Completed`, `Cancelled`, `Returned`
  - `payment_status`: mapped to `Paid`, `Pending`, `Failed`, `Refunded`
  - `ship_mode`: mapped to `Second Class`, `First Class`, `Same Day`, `Standard Class`

### 2.2 Date Cleaning
- Parsed all date formats using Python `dateutil.parser` with `dayfirst=False`
- Standardized all dates to `DD-MMM-YYYY` format in the cleaned file
- Created `shipping_delay_days` column as `ship_date - order_date` in days
- Extracted `order_month` and `order_year` from `order_date`

### 2.3 Duplicate Handling
- **Exact duplicates**: 20 rows removed entirely (fully identical records)
- **Conflicting duplicate order_ids**: 12 unique order IDs found with different field values — flagged with `data_quality_flag = Invalid`. First occurrence retained; all occurrences flagged.

### 2.4 Missing Value Handling
- `region`: Filled with `Unknown` per business rule
- `ship_mode`: Filled with `Unknown` per business rule
- `discount`: Set to `0.0` where other numeric fields (quantity, unit_price, sales, cost) are valid

### 2.5 Discount Correction
- Negative discounts: Flagged as `Invalid` in `data_quality_flag`; `cleaned_discount` retains original value for audit trail
- `calculated_sales` uses `cleaned_discount.clip(lower=0)` to prevent negative discount from inflating sales

### 2.6 Sales & Profit Recalculation
- `calculated_sales = quantity × unit_price × (1 - cleaned_discount)`
- `calculated_profit = calculated_sales - cost`
- `profit_margin = calculated_profit / calculated_sales`
- Original `sales` and `profit` columns are NOT overwritten — calculated columns are added as new columns

---

## 3. Business Rules Applied

| Rule | Action Taken |
|---|---|
| Missing region | Filled as `Unknown`; flagged in data_quality_report |
| Missing ship_mode | Filled as `Unknown`; flagged in data_quality_report |
| Missing discount | Treated as `0` only if all other sales fields are valid |
| Negative discount | Flagged as `Invalid` in `data_quality_flag` |
| Discount above 0.65 | Flagged as `Invalid` (none found in this dataset) |
| Cancelled orders | Excluded from final completed sales pivot summaries |
| Failed payments | Excluded from final completed sales pivot summaries |
| Refunded orders | Separately summarized in Exceptions by Region sheet |
| Ship date before order date | Flagged as `Invalid`; `shipping_delay_days` will be negative |
| Exact duplicate rows | Removed entirely |
| Conflicting duplicate order_ids | Flagged for review; retained but marked `Invalid` |

---

## 4. Assumptions Made

1. **Discount tolerance**: Maximum valid discount is 0.65 (65%) based on observed data maximum. Any negative discount is treated as a data entry error.
2. **Sales recalculation tolerance**: A difference of more than ₹0.50 between original `sales` and `calculated_sales` is considered a mismatch.
3. **Date parsing ambiguity**: For ambiguous dates (e.g. `06/08/2024`), `dayfirst=False` is used — so this is interpreted as June 8 (US format), not August 6.
4. **Missing discount with valid fields**: Where `discount` is null but `quantity`, `unit_price`, `sales`, and `cost` are all valid numbers, discount is assumed to be 0.
5. **Conflicting duplicates**: The first occurrence of a conflicting duplicate order_id is retained as the "canonical" record. No business logic was available to determine which duplicate is correct.
6. **Completed sales scope**: Pivot summaries for revenue and profit use only records where `order_status = Completed` AND `payment_status = Paid`.
7. **Currency**: All monetary values are assumed to be in Indian Rupees (₹) given the Indian customer/state data.

---

## 5. Records Removed

| Action | Count |
|---|---|
| Exact duplicate rows removed | 20 |
| **Total removed** | **20** |

No other records were deleted. Problematic records (conflicting duplicates, invalid discounts, invalid ship dates, calculation mismatches) were **flagged** rather than deleted.

---

## 6. Records Flagged

| Flag Type | Count |
|---|---|
| `Invalid` (serious data issues) | 128 |
| `Warning` (minor issues, usable with caution) | 70 |
| `Clean` (passed all rules) | 714 |
| **Total after cleaning** | **912** |

### Invalid flag conditions (any one of):
- Conflicting duplicate order_id
- Ship date before order date (negative `shipping_delay_days`)
- Negative `cleaned_discount`

### Warning flag conditions (any one of):
- Missing `order_date` or `ship_date`
- `region` filled as `Unknown`
- `ship_mode` filled as `Unknown`
- `calculated_sales` differs from original `sales` by > ₹0.50

---

## 7. Limitations of the Cleaning Process

1. **Date format ambiguity**: For dates like `06/08/2024`, the interpretation (June 8 vs August 6) depends on the assumed locale. `dayfirst=False` was used consistently but may misparse some records.

2. **No source-of-truth for conflicting duplicates**: When the same `order_id` appears with different values, there is no way to determine which record is correct without access to the source system. Both are flagged but the first occurrence is retained.

3. **Discount policy threshold**: The maximum allowed discount (0.65) was inferred from the data maximum. If the actual business policy differs, this threshold should be updated.

4. **No customer deduplication**: `customer_name` and `customer_id` were not cross-checked. A customer may appear under slightly different name spellings that were not fully reconciled.

5. **Cost field not validated**: The `cost` column was accepted as-is. Negative or zero cost values were not separately flagged.

6. **No referential integrity checks**: `customer_id`, `product_name` etc. were not validated against a master reference — such a table was not provided.

7. **Manual mapping completeness**: The text normalization mappings are based on variants observed in this dataset. New data imports may contain variants not yet mapped.
