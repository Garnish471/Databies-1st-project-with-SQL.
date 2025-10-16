# Databies-1st-project-with-SQL.
This documentation outlines the complete, step-by-step data cleaning process performed on the **Sales** and **Product** tables. 
It is formatted for clear presentation on platforms like GitHub, detailing the issue, the decision made, and the corresponding SQL code.

# 📝 Data Cleaning Documentation: Sales and Product Tables
This document details the data quality issues identified and the corrective measures taken using MySQL to produce clean, reliable datasets for analysis.

## I. Sales Table Cleaning (`Sales`)
The Sales table was cleaned for **Validity** (Future Dates) and **Consistency/Accuracy** (Product ID mapping and `total_amount`).

### 1\. Issue: Future Dates (Validity)
The original sales data contained records dated in **2025**, which are logically invalid for historical sales analysis.
| **Validity** | `sale_date` | Records found with dates in 2025 (e.g., $2025-07-03$). | Delete all records where the year is $\ge 2025$. |

**SQL Implementation:**
Step 1: Remove Future Dates
DELETE FROM Sales
WHERE YEAR(sale_date) >= 2025;

### Verification Check: Should return 0
SELECT COUNT(*) FROM Sales
WHERE YEAR(sale_date) >= 2025;

### 2\. Issue: Logical Duplicates and Inaccurate `product_id` (Consistency)

The `Products` table had multiple `product_id`s pointing to the exact same physical product (e.g., IDs 10 and 11 for the same yogurt). 
The `Sales` table needed to be updated to reference only one canonical `product_id` (the lowest ID) for each product group.

| **Consistency** | `product_id` | Sales records reference `product_id`s that are logical duplicates in the `Products` table. 
Map all old, duplicate `product_id`s to the minimum `product_id` of that logical product group. |

**SQL Implementation:**
Step 2a: Create a temporary table to map all old product IDs to the minimum ID in their group
CREATE TEMPORARY TABLE ProductMapping AS
SELECT
    p.product_id AS old_product_id,
    MIN(p.product_id) OVER (PARTITION BY p.product_name, p.category, p.unit_price, p.brand, p.stock_quantity) AS new_product_id
FROM Products p;

Step 2b: Update the Sales table using the new, consistent product_id
UPDATE Sales s
INNER JOIN ProductMapping pm ON s.product_id = pm.old_product_id
SET s.product_id = pm.new_product_id
WHERE s.product_id != pm.new_product_id;


### 3\. Issue: Inaccurate `total_amount` (Accuracy)

The recorded `total_amount` did not always equal the calculated amount ($\text{Quantity} \times \text{Unit Price}$). The calculated value is considered the source of truth.

| **Accuracy** | `total_amount` | Does not match $(\text{Quantity} \times \text{Unit Price})$ from the `Products` table. 
Recalculate and update `total_amount` using $\text{Quantity} \times \text{Unit Price}$ from the now-consistent `Products` table. |

**SQL Implementation:**
Step 3: Recalculate and correct total_amount
Note: Uses the updated product_id column from Step 2b
UPDATE Sales s
INNER JOIN Products p ON s.product_id = p.product_id
SET s.total_amount = s.quantity * p.unit_price;

## II. Product Table Cleaning (`Products`)

The Product table was cleaned for **Integrity** by removing logical duplicate rows after ensuring the Sales table was updated to use the correct IDs.

### 4\. Issue: Logical Duplicates (Integrity)

Multiple rows existed where all descriptive attributes were identical, but each had a unique `product_id`.
| **Integrity** | All | Identical product attributes (`product_name`, `unit_price`, etc.) are linked to different `product_id`s. | Delete the duplicate rows, **keeping only the row with the minimum `product_id`**. |

**SQL Implementation:**
Step 4a: Create a temporary table to find the minimum product_id for each unique product specification (IDs to KEEP)
CREATE TEMPORARY TABLE IDsToKeep AS
SELECT MIN(product_id) AS min_id
FROM Products
GROUP BY product_name, category, unit_price, brand, stock_quantity;

Step 4b: Delete all product IDs that are NOT the minimum ID for their group
DELETE FROM Products
WHERE product_id NOT IN (SELECT min_id FROM IDsToKeep);

Verification Check: Should return an empty set
SELECT product_name, COUNT(*)
FROM Products
GROUP BY product_name
HAVING COUNT(*) > 1;

## Summary of Results
| **Sales** | 3,000 | 2,007 | Future Dates, Inconsistent Product IDs, Inaccurate Total Amounts |
| **Products** | 25 | 21 | Logical Duplicates |

The resulting `Sales` and `Products` tables are now harmonized and suitable for accurate sales performance analysis.
