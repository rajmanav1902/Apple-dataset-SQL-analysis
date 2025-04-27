# Apple-dataset-SQL-analysis
A detailed analysis of Apple dataset using MY SQL
<img src="Apple store.jpg">

# Project Overview
This project is designed to showcase advanced SQL querying techniques through the analysis of over 1 million rows of Apple retail sales data. The dataset includes information about products, stores, sales transactions, and warranty claims across various Apple retail locations globally. By tackling a variety of questions, from basic to complex, I have analyzed the data and tried to understand the trends in the sale.

## Database Schema

The project uses five main tables:

1. **stores**: Contains information about Apple retail stores.
   - `store_id`: Unique identifier for each store.
   - `store_name`: Name of the store.
   - `city`: City where the store is located.
   - `country`: Country of the store.

2. **category**: Holds product category information.
   - `category_id`: Unique identifier for each product category.
   - `category_name`: Name of the category.

3. **products**: Details about Apple products.
   - `product_id`: Unique identifier for each product.
   - `product_name`: Name of the product.
   - `category_id`: References the category table.
   - `launch_date`: Date when the product was launched.
   - `price`: Price of the product.

4. **sales**: Stores sales transactions.
   - `sale_id`: Unique identifier for each sale.
   - `sale_date`: Date of the sale.
   - `store_id`: References the store table.
   - `product_id`: References the product table.
   - `quantity`: Number of units sold.

5. **warranty**: Contains information about warranty claims.
   - `claim_id`: Unique identifier for each warranty claim.
   - `claim_date`: Date the claim was made.
   - `sale_id`: References the sales table.
   - `repair_status`: Status of the warranty claim (e.g., Paid Repaired, Warranty Void).

## Objectives

The project has multiple queries to use SQL skills to analyze the dataset in depth:

### Initital analysis

1. Find the number of stores in each country.

```sql
SELECT country, COUNT(store_id) AS number_of_stores
FROM stores
GROUP BY country;
```

2. Calculate the total number of units sold by each store.

```sql
SELECT s.store_name, SUM(sa.quantity) AS total_units_sold
FROM stores s
JOIN sales sa ON s.store_id = sa.store_id
GROUP BY s.store_name;
```

3. Identify how many sales occurred in December 2023.

```sql
SELECT COUNT(sale_id) AS number_of_sales
FROM sales
WHERE sale_date BETWEEN '2023-12-01' AND '2023-12-31';
```

4. Determine how many stores have never had a warranty claim filed.

```sql
SELECT COUNT(s.store_id) AS stores_without_claims
FROM stores s
LEFT JOIN sales sa ON s.store_id = sa.store_id
LEFT JOIN warranty w ON sa.sale_id = w.sale_id
WHERE w.claim_id IS NULL;
```

5. Calculate the percentage of warranty claims marked as "Warranty Void".

```sql
SELECT
    SUM(CASE WHEN repair_status = 'Warranty Void' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS percentage_void
FROM warranty;
```

6. Identify which store had the highest total units sold in the last year.

```sql
SELECT s.store_name, SUM(sa.quantity) AS total_units_sold
FROM stores s
JOIN sales sa ON s.store_id = sa.store_id
WHERE sa.sale_date BETWEEN '2024-04-27' AND '2025-04-27'
GROUP BY s.store_name
ORDER BY total_units_sold DESC
LIMIT 1;
```

7. Count the number of unique products sold in the last year.

```sql
SELECT COUNT(DISTINCT product_id) AS unique_products_sold
FROM sales
WHERE sale_date BETWEEN '2024-04-27' AND '2025-04-27';
```

8. Find the average price of products in each category.

```sql
SELECT c.category_name, AVG(p.price) AS average_price
FROM category c
JOIN products p ON c.category_id = p.category_id
GROUP BY c.category_name;
```

9. How many warranty claims were filed in 2020?

```sql
SELECT COUNT(claim_id) AS number_of_claims
FROM warranty
WHERE claim_date BETWEEN '2020-01-01' AND '2020-12-31';
```

10. For each store, identify the best-selling day based on highest quantity sold.

```sql
WITH DailySales AS (
    SELECT
        s.store_name,
        sa.sale_date,
        SUM(sa.quantity) AS total_quantity_sold,
        ROW_NUMBER() OVER(PARTITION BY s.store_id ORDER BY SUM(sa.quantity) DESC, sa.sale_date) AS rn
    FROM stores s
    JOIN sales sa ON s.store_id = sa.store_id
    GROUP BY s.store_name, sa.sale_date
)
SELECT store_name, sale_date AS best_selling_day, total_quantity_sold
FROM DailySales
WHERE rn = 1;
```

### Identify trends

11. Identify the least selling product in each country for each year based on total units sold.

```sql
WITH YearlyProductSales AS (
    SELECT
        s.country,
        YEAR(sa.sale_date) AS sale_year,
        p.product_name,
        SUM(sa.quantity) AS total_units_sold,
        ROW_NUMBER() OVER(PARTITION BY s.country, YEAR(sa.sale_date) ORDER BY SUM(sa.quantity) ASC) AS rn
    FROM stores s
    JOIN sales sa ON s.store_id = sa.store_id
    JOIN products p ON sa.product_id = p.product_id
    GROUP BY s.country, YEAR(sa.sale_date), p.product_name
)
SELECT country, sale_year, product_name, total_units_sold
FROM YearlyProductSales
WHERE rn = 1;
```

12. Calculate how many warranty claims were filed within 180 days of a product sale.

```sql
SELECT COUNT(w.claim_id) AS claims_within_180_days
FROM warranty w
JOIN sales sa ON w.sale_id = sa.sale_id
WHERE DATEDIFF(w.claim_date, sa.sale_date) <= 180;
```

13. Determine how many warranty claims were filed for products launched in the last two years.

```sql
SELECT COUNT(w.claim_id) AS claims_for_recent_products
FROM warranty w
JOIN sales sa ON w.sale_id = sa.sale_id
JOIN products p ON sa.product_id = p.product_id
WHERE p.launch_date >= '2023-01-01';
```

14. List the months in the last three years where sales exceeded 5,000 units in the USA.

```sql
SELECT YEAR(sa.sale_date) AS sale_year, MONTH(sa.sale_date) AS sale_month
FROM sales sa
JOIN stores s ON sa.store_id = s.store_id
WHERE s.country = 'USA'
  AND sa.sale_date BETWEEN '2022-04-27' AND '2025-04-27'
GROUP BY YEAR(sa.sale_date), MONTH(sa.sale_date)
HAVING SUM(sa.quantity) > 5000;
```

15. Identify the product category with the most warranty claims filed in the last two years.

```sql
SELECT c.category_name, COUNT(w.claim_id) AS number_of_claims
FROM warranty w
JOIN sales sa ON w.sale_id = sa.sale_id
JOIN products p ON sa.product_id = p.product_id
JOIN category c ON p.category_id = c.category_id
WHERE w.claim_date >= '2023-01-01'
GROUP BY c.category_name
ORDER BY number_of_claims DESC
LIMIT 1;
```

### Complex trends

16. Determine the percentage chance of receiving warranty claims after each purchase for each country.

```sql
WITH CountrySales AS (
    SELECT
        s.country,
        COUNT(sa.sale_id) AS total_sales
    FROM stores s
    JOIN sales sa ON s.store_id = sa.store_id
    GROUP BY s.country
),
CountryClaims AS (
    SELECT
        st.country,
        COUNT(w.claim_id) AS total_claims
    FROM warranty w
    JOIN sales sa ON w.sale_id = sa.sale_id
    JOIN stores st ON sa.store_id = st.store_id
    GROUP BY st.country
)
SELECT
    cs.country,
    (COALESCE(cc.total_claims, 0) * 100.0 / cs.total_sales) AS percentage_claims_per_purchase
FROM CountrySales cs
LEFT JOIN CountryClaims cc ON cs.country = cc.country;
```

17. Analyze the year-by-year growth ratio for each store.

```sql
WITH YearlySales AS (
    SELECT
        s.store_name,
        YEAR(sa.sale_date) AS sale_year,
        SUM(sa.quantity) AS yearly_total_sales
    FROM stores s
    JOIN sales sa ON s.store_id = sa.store_id
    GROUP BY s.store_name, YEAR(sa.sale_date)
),
LaggedSales AS (
    SELECT
        store_name,
        sale_year,
        yearly_total_sales,
        LAG(yearly_total_sales, 1, 0) OVER (PARTITION BY store_name ORDER BY sale_year) AS previous_year_sales
    FROM YearlySales
)
SELECT
    store_name,
    sale_year,
    yearly_total_sales,
    previous_year_sales,
    CASE
        WHEN previous_year_sales = 0 THEN NULL -- Avoid division by zero
        ELSE (yearly_total_sales - previous_year_sales) * 100.0 / previous_year_sales
    END AS growth_percentage
FROM LaggedSales
ORDER BY store_name, sale_year;
```

18. Calculate the correlation between product price and warranty claims for products sold in the last five years, segmented by price range.

```sql
WITH RecentSalesClaims AS (
    SELECT
        p.price,
        CASE
            WHEN p.price < 500 THEN 'Under $500'
            WHEN p.price BETWEEN 500 AND 1000 THEN '$500 - $1000'
            ELSE 'Over $1000'
        END AS price_range,
        CASE WHEN w.claim_id IS NOT NULL THEN 1 ELSE 0 END AS has_claim
    FROM sales sa
    JOIN products p ON sa.product_id = p.product_id
    LEFT JOIN warranty w ON sa.sale_id = w.sale_id
    WHERE sa.sale_date >= DATE_SUB(CURDATE(), INTERVAL 5 YEAR)
),
PriceRangeStats AS (
    SELECT
        price_range,
        AVG(price) AS avg_price,
        AVG(has_claim) AS claim_probability,
        COUNT(*) AS num_sales
    FROM RecentSalesClaims
    GROUP BY price_range
)
SELECT
    price_range,
    avg_price,
    claim_probability,
    num_sales
FROM PriceRangeStats;
```
19. Identify the store with the highest percentage of "Paid Repaired" claims relative to total claims filed.

```sql
WITH StoreClaimStatus AS (
    SELECT
        s.store_name,
        COUNT(w.claim_id) AS total_claims,
        SUM(CASE WHEN w.repair_status = 'Paid Repaired' THEN 1 ELSE 0 END) AS paid_repaired_claims
    FROM stores s
    JOIN sales sa ON s.store_id = sa.store_id
    LEFT JOIN warranty w ON sa.sale_id = w.sale_id
    GROUP BY s.store_name
)
SELECT
    store_name,
    (paid_repaired_claims * 100.0 / total_claims) AS percentage_paid_repaired
FROM StoreClaimStatus
WHERE total_claims > 0 -- Avoid division by zero
ORDER BY percentage_paid_repaired DESC
LIMIT 1;
```

20. Write a query to calculate the monthly running total of sales for each store over the past four years and compare trends during this period.

```sql
WITH MonthlyStoreSales AS (
    SELECT
        s.store_name,
        DATE_FORMAT(sa.sale_date, '%Y-%m') AS sale_month,
        SUM(sa.quantity) AS monthly_sales
    FROM sales sa
    JOIN stores s ON sa.store_id = s.store_id
    WHERE sa.sale_date >= DATE_SUB(CURDATE(), INTERVAL 4 YEAR)
    GROUP BY s.store_name, DATE_FORMAT(sa.sale_date, '%Y-%m')
),
RunningTotalSales AS (
    SELECT
        store_name,
        sale_month,
        monthly_sales,
        SUM(monthly_sales) OVER (PARTITION BY store_name ORDER BY sale_month) AS running_total
    FROM MonthlyStoreSales
)
SELECT
    store_name,
    sale_month,
    monthly_sales,
    running_total
FROM RunningTotalSales
ORDER BY store_name, sale_month;
```
## Project Focus

This project primarily focuses on developing and showcasing the following SQL skills:

- **Complex Joins and Aggregations**: Demonstrating the ability to perform complex SQL joins and aggregate data meaningfully.
- **Window Functions**: Using advanced window functions for running totals, growth analysis, and time-based queries.
- **Data Segmentation**: Analyzing data across different time frames to gain insights into product performance.
- **Correlation Analysis**: Applying SQL functions to determine relationships between variables, such as product price and warranty claims.
- **Real-World Problem Solving**: Answering business-related questions that reflect real-world scenarios faced by data analysts.


## Dataset

- **Size**: 1 million+ rows of sales data.
- **Period Covered**: The data spans multiple years, allowing for long-term trend analysis.
- **Geographical Coverage**: Sales data from Apple stores across various countries.
