**Customer Segmentation Using RFM Analysis**
1. Database and Table Creation:
```sql
CREATE DATABASE customer_seg;

CREATE TABLE customer_data (
    invoice_no VARCHAR(225),
    customer_id	VARCHAR(225),
    gender VARCHAR(225),
    age	INT,
    category VARCHAR(225),
    quantity INT,
    price DOUBLE,
    payment_method VARCHAR(225),
    invoice_date DATE,	
    shopping_mall VARCHAR(225)
);
```

2. Data Quality Checks:
```sql
SELECT 
	COUNT(*) AS total_rows
FROM customer_data;
```
<img src="sql-snapshot\code snapshot\total_row.png" alt="Getting started" width="100" />

```sql
SELECT 
	DISTINCT COUNT(customer_id) AS distict_customer_id
FROM customer_data;  
```
<img src="./sql-snapshot\code snapshot\distinct_customer_count.png" alt="Getting started" width="140" />

```sql
SELECT *
FROM customer_data
WHERE
    invoice_no IS NULL OR invoice_no = '' OR
    customer_id IS NULL OR customer_id ='' OR
    gender IS NULL OR gender ='' OR
    age IS NULL OR age ='' OR 
    category IS NULL OR category ='' OR
    quantity IS NULL OR quantity ='' OR
    price IS NULL OR price ='' OR
    payment_method IS NULL OR payment_method ='' OR
    invoice_date IS NULL OR
    shopping_mall IS NULL OR shopping_mall ='';
```
<img src="sql-snapshot\code snapshot\null_check.png" alt="Getting started" width="600" />

 ```sql
-- check the date data range    
SELECT
    MIN(invoice_date) AS min_date,
    MAX(invoice_date) AS max_date
FROM customer_data;    

-- SET @today = '1-1-2024';
```
<img src="sql-snapshot\code snapshot\date_range.png" width="150" />

3. RFM Calculation:
3.1. Quartile Calculation
```sql
-- Calculate RFM Values
CREATE VIEW rfm_calc AS (
	SELECT DISTINCT
        customer_id,
        DATEDIFF('2024-1-1', MAX(invoice_date)) AS r_value,
        SUM(quantity) AS f_value,
        SUM(price) AS m_value
    FROM customer_data
    GROUP BY customer_id
);
```

```sql
-- Minimum & Maximum Values
CREATE VIEW min_max AS (
	SELECT 
        MIN(rfm_calc.r_value) AS min_r,
        MAX(rfm_calc.r_value) AS max_r,
        MIN(rfm_calc.f_value) AS min_f,
        MAX(rfm_calc.f_value) AS max_f,
        MIN(rfm_calc.m_value) AS min_m,
        MAX(rfm_calc.m_value) AS max_m
     FROM rfm_calc   
 );
```
 
 ```sql
 WITH percentile AS (
	SELECT 
    -- Calculate row counts for quartile calculations
		COUNT(*) AS total_rows
	FROM rfm_calc
  ),
  ranked AS (
	SELECT 
		-- Rank the Values
		r_value, f_value, m_value,
	    ROW_NUMBER() OVER (ORDER BY r_value) AS r_rank,
        ROW_NUMBER() OVER (ORDER BY f_value) AS f_rank,
        ROW_NUMBER() OVER (ORDER BY m_value) AS m_rank
    FROM rfm_calc    
 )
 -- Calculate Quartiles
SELECT 
	'recency' AS RFM,
	min_max.min_r AS Min,
    MAX(CASE WHEN ranked.r_rank = FLOOR(0.25 * percentile.total_rows) THEN r_value ELSE NULL  END) AS Q1,
    MAX(CASE WHEN ranked.r_rank = FLOOR(0.5 * percentile.total_rows) THEN r_value ELSE NULL END) AS Median,
    MAX(CASE WHEN ranked.r_rank = FLOOR(0.75 * percentile.total_rows) THEN r_value ELSE NULL END) AS Q3,
    min_max.max_r AS Max
FROM  percentile, min_max,ranked
GROUP BY min, max
  
UNION ALL

SELECT
   'freqency' AS RFM,
    min_max.min_f AS Min,
    MAX(CASE WHEN ranked.f_rank = FLOOR(0.25 * percentile.total_rows) THEN f_value ELSE NULL END) AS Q1,
    MAX(CASE WHEN ranked.f_rank = FLOOR(0.5 * percentile.total_rows) THEN f_value ELSE NULL END) AS Median,
    MAX(CASE WHEN ranked.f_rank = FLOOR(0.75 * percentile.total_rows) THEN f_value ELSE NULL END) AS Q3,
    min_max.max_f AS Max
FROM  percentile, min_max, ranked
GROUP BY min, max

UNION ALL

SELECT
	'monetary' AS RFM,  
    min_max.min_m AS Min,
    MAX(CASE WHEN ranked.m_rank = FLOOR(0.25 * percentile.total_rows) THEN m_value ELSE NULL END) AS Q1,
    MAX(CASE WHEN ranked.m_rank = FLOOR(0.5 * percentile.total_rows) THEN m_value ELSE NULL END) AS Median,
    MAX(CASE WHEN ranked.m_rank = FLOOR(0.75 * percentile.total_rows) THEN m_value ELSE NULL END) AS Q3,
    min_max.max_m AS Max
FROM  percentile, min_max, ranked
GROUP BY min, max; 
 ```
<img src="sql-snapshot\code snapshot\quartile_calc.png" width="300" />

3.2. Score Assignment:
```sql
-- Calculate RMF Scores
CREATE VIEW rfm_score AS
SELECT
    customer_id,
    r_value,
    f_value,
    m_value,
    NTILE(5) OVER (ORDER BY r_value DESC) AS r_score,
    NTILE(5) OVER (ORDER BY f_value) AS f_score,
    NTILE(5) OVER (ORDER BY m_value) AS m_score
FROM rfm_calc;
```    

```sql
-- calculate the ranges (minimum and maximum values) for recency, frequency, and monetary scores
WITH r_score_range AS (
	SELECT 
        ROW_NUMBER() OVER (ORDER BY r_score) AS I,
        r_score,
        MIN(r_value) AS min_r,
        MAX(r_value) AS max_r
	FROM rfm_score
    GROUP BY r_score
),
f_score_range AS (
	SELECT 
        ROW_NUMBER() OVER (ORDER BY f_score) AS I,
        f_score,
        MIN(f_value) AS min_f,
        MAX(f_value) AS max_f
    FROM rfm_score
    GROUP BY f_score
),
m_score_range AS (
	SELECT
        ROW_NUMBER() OVER (ORDER BY m_score) AS I,
        m_score,
        MIN(m_value) AS min_m,
        MAX(m_value) AS max_m
	FROM rfm_score
    GROUP BY m_score
)
SELECT
    R.r_score, min_r, max_r,
    F.f_score, min_f, max_f,
    M.m_score, min_m, max_m
FROM r_score_range AS R
JOIN f_score_range AS F ON R.I = F.I
JOIN m_score_range AS M ON R.I = M.I;
```
<img src="sql-snapshot\code snapshot\score_range.png" width="450" />

```sql
CREATE VIEW rfm_view AS 
WITH avg_rfm_score AS(
-- Calculate Avg RFM Score
	SELECT
        customer_id,
        CONCAT_WS("-", r_score, f_score, m_score) AS r_f_m,
        ROUND((r_score + f_score + m_score)/3,2) AS avg_rfm
    FROM rfm_score
)
SELECT
    V.customer_id,
    S.r_score, S.f_score, S.m_score,
    V.r_f_m, V.avg_rfm
FROM avg_rfm_score AS V  
JOIN rfm_score AS S
ON S.customer_id = V.customer_id;
```
	
```sql
SELECT * FROM rfm_view
ORDER BY avg_rfm DESC
LIMIT 10; 
```   
<img src="sql-snapshot\code snapshot\top_avg_rfm.png" width="350" />

3.3. Customer Segmentation:
```sql
-- Create a View for the Customer Segments & Value Segments using the View "rfm_view"  
CREATE VIEW customer_value_seg AS 
	SELECT 
		*,
        CASE WHEN avg_rfm > 4 THEN 'High' 
            WHEN avg_rfm <= 4 AND avg_rfm > 2.5 THEN 'Medium'
            WHEN avg_rfm <= 2.5 THEN 'Low'
        END AS value_seg,
        CASE WHEN r_score > 4 AND f_score > 4 AND m_score > 4 THEN 'VIP'
            WHEN f_score >= 3 and m_score < 4 THEN 'Regular'
            WHEN r_score <= 3 and r_score > 1 THEN 'Dormat'
            WHEN r_score = 1 THEN 'Churned'
            WHEN r_score >= 4 and f_score <= 4 THEN 'New Customer'
            ELSE "Other"
         END AS cus_seg
     FROM  rfm_view; 
```
     
```sql
SELECT * FROM  customer_value_seg
ORDER BY avg_rfm DESC
LIMIT 5;
```
<img src="sql-snapshot\code snapshot\top_cus_va_seg.png" width="450" />

3.4. Customer Distribution:
```sql
-- Distribution of Customers by Value Segment
SELECT
    value_seg,
    COUNT(customer_id) AS customer_count
FROM customer_value_seg
GROUP BY value_seg;  
``` 
<img src="sql-snapshot\code snapshot\dis_va.png" width="200" />

```sql
-- Distribution of Customers by Customer Segment
SELECT 
    cus_seg,
    COUNT(customer_id) AS customer_count
FROM customer_value_seg
GROUP BY cus_seg; 
```
<img src="sql-snapshot\code snapshot\dis_cus.png" width="200" />

```sql
-- Distribution of customers across different RFM customer segments within each value segment
SELECT
    value_seg,
    cus_seg,
   COUNT(customer_id) AS customer_count
FROM customer_value_seg
GROUP BY value_seg, cus_seg
ORDER BY customer_count DESC; 

```
<img src="sql-snapshot\code snapshot\dis_cus_va.png" width="250" />

**These are some recommendation based on customer segmentation results:**

- Focusing on Medium-Value Customers:
The largest groups are Medium-Value customers, particularly the "Regular" and "New Customer" segments. Consider strategies to retain these customers and increase their value.Ex:  Loyalty programs, personalized marketing or improved customer service can be effective.

- Engaging Dormant Customers:
Both Medium and Low-Value segments have significant numbers of Dormant customers. Reactivation campaigns, such as special offers or reminders of past positive experiences, can help re-engage these customers.

- Addressing Churned Customers:
There are notable counts of Churned customers in the Medium and Low-Value segments. Investigate common reasons for churn and develop targeted win-back strategies to regain these customers.

- Leveraging High-Value Customers:
High-Value segments, including "VIP" and "New Customer," are smaller but highly valuable. Ensure these customers receive top-tier service and exclusive offers to maintain their loyalty and encourage word-of-mouth referrals.

- New Customer Nurturing:
A significant number of New Customers are present across all value segments. Implement onboarding programs to ensure these customers have a positive initial experience, which can increase their lifetime value.

- Regular Customers:
Regular customers, especially those in the Medium-Value segment, are crucial for consistent revenue. Maintain engagement through regular updates, rewards, and personalized communication to foster long-term loyalty.