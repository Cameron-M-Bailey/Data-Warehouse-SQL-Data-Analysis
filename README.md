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
# Data Warehouse ERD
<img width="560" alt="Screenshot 2023-01-28 at 4 19 52 PM" src="https://user-images.githubusercontent.com/104586192/215291657-3b1df6d5-630d-4e1f-b10a-464ced66fa3a.png">
- A simple data warehouse with our fact table containing the original retail price and sales price and our dimensions containing customer, date, and product data.<br>
- The static dimension DIM_STATE_POP contains state population data in order to calculate sales per capita.<br>

# Data Analysis using SQL 

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
FROM
(SELECT sneaker_name, AVG(sale_price) AS average_sales
FROM fact_sales
JOIN dim_product USING(product_id)
GROUP BY sneaker_name) AS t1
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
