# Apple Sales-1M+ records
POSTRAGE SQL, EXCEL

# Apple Retail Sales SQL Project - Analyzing Millions of Sales Rows
# ![Apple Logo](https://github.com/najirh/Apple-Retail-Sales-SQL-Project---Analyzing-Millions-of-Sales-Rows/blob/main/Apple_Changsha_RetailTeamMembers_09012021_big.jpg.slideshow-xlarge_2x.jpg)


## Project Overview

This project is designed to showcase advanced SQL querying techniques through the analysis of over 1 million rows of Apple retail sales data. The dataset includes information about products, stores, sales transactions, and warranty claims across various Apple retail locations globally. By tackling a variety of questions, from basic to complex, you'll demonstrate your ability to write sophisticated SQL queries that extract valuable insights from large datasets.

The project is ideal for data analysts looking to enhance their SQL skills by working with a large-scale dataset and solving real-world business questions.

## Entity Relationship Diagram (ERD)

![ERD](https://github.com/najirh/Apple-Retail-Sales-SQL-Project---Analyzing-Millions-of-Sales-Rows/blob/main/erd.png)


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
## Exploratory Data Analysis
```sql
SELECT  DISTINCT repair_status FROM warranty;
SELECT COUNT(*) FROM sales;
```
### Improving query performance
```sql
EXPLAIN ANALYZE
SELECT * FROM sales
WHERE product_id ='P-44'
```

- Execution time= 134.590ms
- planning time = 0.111ms

```sql
CREATE INDEX sales_products_id ON sales(product_id);
```
- Execution time= 5.097ms
- planning time = 0.138ms

```sql
EXPLAIN ANALYZE
SELECT*FROM sales
WHERE store_id='ST-31'
```

- Planning Time = 0.166ms
-  Execution Time = 112.613ms

```sql
CREATE INDEX sales_store_id ON sales(store_id);
```

- Planning Time = 0.143ms
-  Execution Time = 1.794ms

```sql
CREATE INDEX sales_store_date ON sales(sale_date);
```
- Planning Time = 0.204ms
- Execution Time = 1.889ms

By creating indexes for individual columns the time of planning and execution dropped rapidly.

## Objectives

1. Find the number of stores in each country.
```sql
   SELECT country, COUNT(*) AS cnt FROM stores
   GROUP BY country
   ORDER BY cnt DESC ;
```
   
2. Calculate the total number of units sold by each store.
 ```sql
   SELECT stores.store_name,stores.store_id,SUM(quantity) AS QNT 
   FROM sales
   INNER JOIN stores
   ON sales.store_id = stores.store_id
   GROUP BY stores.store_id, stores.store_name
   ORDER BY QNT DESC ;
```
3. Identify how many sales occurred in December 2023.
 ```sql
 SELECT count(sale_id) AS s
FROM sales
WHERE EXTRACT( YEAR  FROM sale_date) = 2023 AND EXTRACT(Month FROM sale_date) = 12
```
- OR
 ```sql
 SELECT COUNT(sale_id) AS s
 FROM sales
 WHERE TO_CHAR(sale_date,'MM-YYYY') = '12-2023'
```
4. Determine how many stores have never had a warranty claim filed.
```sql
SELECT COUNT(*) FROM stores 
WHERE store_id NOT IN(
                SELECT DISTINCT store_id
                FROM sales
                RIGHT JOIN warranty 
                ON sales.sale_id=warranty.sale_id)
```
5. Calculate the percentage of warranty claims marked as "Warranty Void".
```sql
SELECT repair_status,
   (count(repair_status)* 1.0 / (SELECT count(repair_status) FROM warranty))*100 AS percent
FROM warranty
WHERE repair_status='Warranty Void'
GROUP BY repair_status
```

OR

```sql
SELECT ROUND(COUNT(*)/ (SELECT COUNT(*) FROM warranty) :: numeric *100,2) AS percentage
FROM warranty
WHERE repair_status='Warranty Void'
```

6. Identify which store had the highest total units sold in the last year.
```sql
SELECT store_name, store_id
FROM stores
WHERE store_id = (SELECT store_id
FROM sales
WHERE EXTRACT(YEAR FROM sale_date) = (EXTRACT(YEAR FROM CURRENT_DATE)-1)
GROUP BY store_id
ORDER BY SUM(quantity) DESC
LIMIT 1)
```

OR

```sql
SELECT
SL.store_id, ST.store_name,SUM(SL.quantity) 
FROM sales AS SL
JOIN stores as ST
ON SL.store_id=ST.store_id
WHERE sale_date >= (CURRENT_DATE- INTERVAL'1 YEAR')
GROUP BY 1,2
ORDER BY 2 DESC
LIMIT 1;
```
To find the top store of the last full calendar year, use the first query or the top store based on the last rolling 12 months, use the second query.

7. Count the number of unique products sold in the last year.
```sql
SELECT COUNT( DISTINCT product_id)
FROM sales
WHERE sale_date>=(CURRENT_DATE - INTERVAL '1 year')
```
8. Find the average price of products in each category.
```sql
SELECT c.category_name,p.category_id,AVG(price)
FROM products as p
JOIN category as c
ON p.category_id=c.category_id
GROUP BY 1,2
ORDER BY 2
```
9. How many warranty claims were filed in 2020?
```sql
SELECT COUNT(claim_id)
FROM warranty
WHERE EXTRACT(YEAR FROM claim_date) = 2020
```
10. For each store, identify the best-selling day based on highest quantity sold.
```sql
SELECT store_id,SUM(quantity),TO_CHAR(sale_date,'Day') as day_name,
RANK() OVER (PARTITION BY store_id ORDER BY SUM(quantity)DESC) as rank
FROM sales
GROUP BY 1,3

```
11. Identify the least selling product in each country for each year based on total units sold.
```sql
WITH product_rank
AS(
SELECT ST.country,PD.product_name,SUM(SL.quantity) as QNT,
RANK() OVER(PARTITION BY  ST.country ORDER BY SUM(Sl.quantity)) AS ranks
FROM sales AS SL
JOIN stores AS ST
ON SL.store_id=ST.store_id
JOIN Products as PD
ON SL.product_id=PD.product_id
GROUP BY 1,2
)
SELECT*
FROM products
WHERE ranks=1
```
12. Calculate how many warranty claims were filed within 180 days of a product sale.
```sql
SELECT COUNT(*)
FROM warranty AS W
LEFT JOIN sales AS S
ON W.sale_id=S.sale_id
WHERE W.claim_date - S.sale_date <=180
```
13. Determine how many warranty claims were filed for products launched in the last two years.
```sql
SELECT PD.product_name, COUNT(W.claim_id) AS CC, COUNT(SL.sale_id) AS CS
FROM warranty AS W
RIGHT JOIN  sales AS SL
ON W.sale_id=SL.sale_id
JOIN products AS PD
ON PD.product_id=SL.product_id
WHERE PD.launch_date >= CURRENT_DATE -INTERVAL '2 years'
GROUP BY 1
HAVING COUNT(W.claim_id) >0
```
14. List the months in the last three years where sales exceeded 5,000 units in the USA.
```sql

SELECT  TO_CHAR(sale_date,'MM-YYYY') as month,SUM(SL.quantity)
FROM sales AS SL
JOIN stores AS ST
ON SL.store_id =ST.store_id
WHERE ST.country ='USA'  AND SL.sale_date >= CURRENT_DATE- INTERVAL'3 years'
GROUP BY 1
HAVING SUM(SL.quantity) >5000
```
15. Identify the product category with the most warranty claims filed in the last two years.
```sql
SELECT
	C.category_name,
	COUNT(W.claim_id) AS total_claims
FROM warranty as W
LEFT JOIN
sales as S
ON W.sale_id= S.sale_id
JOIN products as PD
ON PD.product_id=S.product_id
JOIN category as C
ON C.category_id= PD.category_id
WHERE W.claim_date >= CURRENT_DATE -INTERVAL '2 years'
GROUP BY 1
```
16. Determine the percentage chance of receiving warranty claims after each purchase for each country.
```sql
SELECT country,QT,CT,
	   COALESCE(CT::numeric/QT::numeric *100,0) AS risk
FROM
(SELECT ST.country,SUM(SL.quantity) AS QT,COUNT(W.claim_id) AS CT
FROM sales AS SL
JOIN stores  AS ST
ON SL.store_id=ST.store_id
LEFT JOIN warranty AS W
ON W.sale_id=SL.sale_id
GROUP BY 1)  t1
ORDER BY 4 DESC
```
17. Analyze the year-by-year growth ratio for each store.
```sql
WITH year_sales 
AS(
SELECT SL.store_id,
ST.store_name,
EXTRACT(YEAR FROM sale_date) as year,
SUM(SL.quantity* P.price) as total_sale
FROM sales AS SL
JOIN products AS P
ON SL.product_id=P.product_id
JOIN stores AS ST
ON ST.store_id=SL.store_id
GROUP BY 1,2,3
ORDER BY 2,3),
growth_ratio 
AS(
SELECT store_name,year,
LAG(total_sale,1) OVER(PARTITION BY store_name ORDER BY year) AS last_year_sale,
total_sale as current_year_sale

FROM year_sales 
)
SELECT store_name, last_year_sale,current_year_sale  ,(current_year_sale-last_year_sale)::numeric/last_year_sale::numeric * 100
FROM growth_ratio
WHERE last_year_sale IS NOT NULL AND
YEAR<> EXTRACT(YEAR FROM CURRENT_DATE)
```
18. Calculate the correlation between product price and warranty claims for products sold in the last five years, segmented by price range.
```sql
SELECT
	CASE
		WHEN P.price < 500 THEN 'Less Expensive Products'
		WHEN P.price BETWEEN 500 AND 1000 THEN 'Medium Expensive Products'
		ELSE 'Expensive Product'
		END AS price_segemnt,
		W.claim_id
FROM warranty as W
LEFT JOIN sales as S
ON W.sale_id=S.sale_id
JOIN products as P
ON P.product_id=S.product_id
WHEN claim_date >= CURRENT_DATE - INTERVAL '5 year'
GROUP BY 2
```
19. Identify the store with the highest percentage of "Paid Repaired" claims relative to total claims filed.
```sql
WITH 
total_repaired
AS(SELECT store_id, COUNT(claim_id) as total_repaired
FROM sales as s
JOIN warranty as w
ON w.sale_id = s.sale_id
GROUP BY 1
),
paid_repaired
AS(SELECT  store_id, COUNT(claim_id) as paid_repaired
FROM sales as s
JOIN warranty as w
ON w.sale_id = s.sale_id
WHERE w.repair_status = 'Paid Repaired'
GROUP BY 1)
SELECT (paid_repaired::numeric/total_repaired::numeric)*100
FROM total_repaired AS T
JOIN paid_repaired AS P
ON T.store_id=P.store_id
```
20. Write a query to calculate the monthly running total of sales for each store over the past four years and compare trends during this period.
```sql
WITH monthly_sales
AS(SELECT store_id,
		EXTRACT(MONTH FROM sale_date) AS years,
		EXTRACT(YEAR FROM sale_date) AS months,
		SUM(P.price*S.quantity) AS total_revenue
FROM sales as S
JOIN
products AS P
ON S.product_id=P.product_id
GROUP BY 1,2,3
ORDER BY 1,2,3)
SELECT store_id,months,years,total_revenue,
		SUM(total_revenue) OVER(PARTITION BY store_id ORDER BY years,months) AS running_total
FROM monthly_sales
```
21. Analyze product sales trends over time, segmented into key periods: from launch to 6 months, 6-12 months, 12-18 months, and beyond 18 months.
```sql
SELECT P.product_name,
CASE
	WHEN S.sale_date BETWEEN P.launch_date AND P.launch_date+INTERVAL'6 month' THEN '0-6'
	WHEN S.sale_date BETWEEN P.launch_date AND P.launch_date+INTERVAL'12 month' THEN '6-12'
	WHEN S.sale_date BETWEEN P.launch_date AND P.launch_date+INTERVAL'18 month' THEN '12-18'
	ELSE '18+'
END AS plc,
SUM(S.quantity) AS total_qnt
FROM sales AS S
JOIN  products AS P
ON S.product_id=P.product_id
GROUP BY 1,2
ORDER BY 1,3
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

## Conclusion

By completing this project, you will develop advanced SQL querying skills, improve your ability to handle large datasets, and gain practical experience in solving complex data analysis problems that are crucial for business decision-making. This project is an excellent addition to your portfolio and will demonstrate your expertise in SQL to potential employers.

---
