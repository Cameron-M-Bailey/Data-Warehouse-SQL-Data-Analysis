# Data-Warehouse-SQL-Data-Analysis
### The development of a small-scale data warehouse in a star schema and analysis of StockX sales data using SQL.
# Objective 
The project builds a simple data warehouse in PostgreSQL in order to query and analyze StockX sales and product data for Yeezy and Nike x Off-White sneakers. SQL queries were written to accomplish a variety of tasks including to create custom functions, identify top performing products, and calculate sneakers with the best resale value. 
# Dataset 
This dataset is from the StockX 2019 Data Contest. It consists of sales provided by StockX with approximately 10,000 shoe sales from 50 different models of Nike x Off-White and Yeezy. Data includes:

<b>Order Date:</b> date of sneaker sale<br>
<b>Brand:</b> brand of sneaker<br>
<b>Sneaker Name:</b> name of sneaker<br>
<b>Sale Price:</b> StockX sale price of sneaker<br>
<b>Retail Price:</b> original retail price of sneaker<br>
<b>Release Date:</b> release date of sneaker<br>
<b>Shoe Size:</b> sneaker shoe size<br>
<b>Buyer Region:</b> customer location<br>
# Data Warehouse - Entity Relationship Diagram
A simple data warehouse with a fact table containing the original retail price and sales price as well as dimension tables containing customer, date, and product data. The static dimension DIM_STATE_POP contains state population data in order to calculate sales per capita.

<img width="560" alt="Screenshot 2023-01-28 at 4 19 52 PM" src="https://user-images.githubusercontent.com/104586192/215291657-3b1df6d5-630d-4e1f-b10a-464ced66fa3a.png">

# Data Analysis using SQL 

#### Create a Custom Function to Calculate Per Capita Sales 
```sql
# Use for quick comparison of key markets  
CREATE OR REPLACE FUNCTION calculate_per_capita_sales(st text)
RETURNS NUMERIC
AS
$$

DECLARE
	  total_sales NUMERIC;
	  pop_calc NUMERIC;
	  per_capita_sales NUMERIC;
  BEGIN
		SELECT SUM(sale_price) INTO total_sales
		FROM fact_sales JOIN dim_customer USING (customer_id)
		WHERE buyer_region = st
		GROUP BY buyer_region;
		
		SELECT population INTO pop_calc
		FROM dim_state_pop
		WHERE state_name = st;
		
		per_capita_sales := ROUND((total_sales/pop_calc),4);
		RETURN per_capita_sales;
	END;
$$
LANGUAGE plpgsql;

SELECT calculate_per_capita_sales('California');
SELECT calculate_per_capita_sales('New York');
SELECT calculate_per_capita_sales('Texas');
SELECT calculate_per_capita_sales('Illinois');
```
#### Create Full Sales per Capita by State Report 
```sql
SELECT buyer_region, ROUND((SUM(sale_price)/population),4) AS per_capita FROM fact_sales
JOIN dim_customer USING(customer_id)
JOIN dim_state_pop ON
dim_customer.buyer_region = dim_state_pop.state_name
GROUP BY buyer_region, population
ORDER BY per_capita DESC, buyer_region ASC;
```
#### Identify Sneakers with Poor Selling Performance and Calculate Total Monthly Loss
```sql
# Calculate total loss from sneakers who sell under their original retail price 
WITH poor_performance AS 
(SELECT DATE_TRUNC('month', order_date) AS monthly, 
 (sale_price - retail_price) AS total_loss
 FROM fact_sales JOIN dim_product USING(product_id)
 JOIN dim_date USING(date_id)
 WHERE retail_price > sale_price AND EXTRACT ('year' FROM order_date) = 2018),
 
 # Group total loss by month 
 poor_performance_total AS
 (SELECT monthly, SUM(total_loss) AS monthly_total_loss
 FROM poor_performance
 GROUP BY monthly)
 
# Create report showcasing sneakers with poor selling performance by month 
SELECT * FROM poor_performance_total;
```
#### Average Sales Comparison by Quartile  
```sql
SELECT sneaker_name, average_sales,
NTILE(4) OVER (ORDER BY average_sales ASC)
FROM (SELECT sneaker_name, 
AVG(sale_price) AS average_sales
FROM fact_sales
JOIN dim_product USING(product_id)
GROUP BY sneaker_name) AS t1
```
#### Rank Products by Average Sales 
```sql 
SELECT sneaker_name, average_sales,
RANK() OVER (ORDER BY average_sales DESC)
FROM (SELECT sneaker_name, 
AVG(sale_price) AS average_sales
FROM fact_sales
JOIN dim_product USING(product_id)
GROUP BY sneaker_name) AS t1
```
#### Determine and Categorize Sneaker Performance Based on Average Markup Percent  
```sql
# Average Resale Markup is 100%
SELECT sneaker_name, average_resale_percent_change,
CASE
    WHEN average_resale_percent_change > 100 THEN 'High Demand'
    WHEN average_resale_percent_change < 100 AND average_resale_percent_change >= 50 THEN 'Average'
    ELSE 'Poor Performance'
END AS sneaker_performance
FROM 
(SELECT sneaker_name, ROUND((((AVG(sale_price)- retail_price)/retail_price)*100),2) 
AS average_resale_percent_change
FROM fact_sales JOIN dim_product USING(product_id)
JOIN dim_date USING(date_id)
WHERE EXTRACT ('year' FROM order_date) = 2018
GROUP BY sneaker_name, retail_price) AS t1
ORDER BY average_resale_percent_change DESC;
```
#### Create Rollup Report of Sneaker Demand 
```sql
SELECT brand, sneaker_name, shoe_size, COUNT(*)
FROM dim_product 
GROUP BY rollup(brand, sneaker_name,shoe_size)
ORDER BY COUNT(*) DESC;
```
#### On Average, Which Sneakers Return the Greatest Investment 
```sql 
SELECT sneaker_name, ROUND(AVG(total_profit),2)
FROM (SELECT sneaker_name, (sale_price - retail_price) AS total_profit
FROM fact_sales
JOIN dim_product 
ON fact_sales.product_id = dim_product.product_id) AS t1
GROUP BY sneaker_name
ORDER BY AVG(total_profit) DESC
LIMIT 20;
```
#### Calculate Each Sneaker Brand's Total Sales for 2019
```sql
SELECT brand, SUM(sale_price) AS total_sales 
FROM fact_sales 
JOIN dim_product 
ON fact_sales.product_id = dim_product.product_id
JOIN dim_date 
ON fact_sales.date_id = dim_date.date_id
WHERE EXTRACT('year' FROM order_date) = 2019
GROUP BY brand
ORDER BY total_sales DESC;
```
#### Identify if There is Correlation Between Average Sales and Longer Times From Release Date 
```sql
SELECT sneaker_name, ROUND(AVG(order_date - release_date),2) AS average_time_release_to_sale,
ROUND(AVG(sale_price),2) AS average_sales
FROM fact_sales 
JOIN dim_date USING(date_id)
JOIN dim_product USING(product_id)
GROUP BY sneaker_name
ORDER BY average_time_release_to_sale DESC;
```
#### Identify Top-Selling Sneakers 
```sql
SELECT sneaker_name, SUM(sale_price) AS top_sellers 
FROM public.fact_sales 
JOIN public.dim_product 
ON public.fact_sales.product_id = public.dim_product.product_id
GROUP BY sneaker_name
ORDER BY SUM(sale_price) DESC
LIMIT 10;
```
#### Average Sales for Nike x Off-White Sneakers in the 4th Quarter 
```sql
SELECT AVG(sale_price) AS off_white_average_sales 
FROM fact_sales JOIN dim_date USING(date_id)
JOIN dim_product USING(product_id)
WHERE EXTRACT ('year' FROM order_date) = 2018
AND order_date BETWEEN '2018-10-01' AND '2018-12-31'
AND brand = 'Off-White';
```
# Key Insights 
- Based on sales per capita, Oregan and Delaware greatly lead in product sales with key markets New York and California being 3rd and 5th, respectively.
- Total loss for 2018 accounts for only 0.01% of total sales, most sneakers are performing above original retail price.
- On average, high demand sneakers are selling 370% over their original market price.  
- The top performing sneakers are the Nike x Off-White Air Jordan 1 Retro High Tops in White and Chicao colorways; Adidas Yeezy Boost 350 V2 in Sesame and Butter colorways are selling the worst.
- Sneaker sizes 9 - 11 are the most in demand. 
- In 2019, Yeezy's total sales are close to double of Nike x Off-White sales. 
- There is no correlation between sneakers being on the market longer and higher sale prices. 

                                                                                                                                                                      

