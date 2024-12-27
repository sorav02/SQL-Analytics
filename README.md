# SQL-Analytics
Financial &amp; Sales Analytics using SQL

Problem Statement
#### Atliq Hardware is a manufacturer of Computer Peripherals and sells its product through its direct stores and through online & offline distribution platforms. The company uses SQL database to store its data. The company wants to use data insight to better understand its customer base and assess the sales of the products being sold. The management has given a list of requirements which needs to be fulfilled using SQL.





1.	Gross Sales Report for customer ‘Croma India’ The report should include columns: - 
a.	Date
b.	Product_code
c.	Product
d.	Variant
e.	Sold_quantity
f.	Gross_price


SELECT s.date, 
            s.product_code, 
            p.product, 
            p.variant, 
            s.sold_quantity, 
            g.gross_price,
            ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total
	FROM fact_sales_monthly s
	JOIN dim_product p
            ON s.product_code=p.product_code
	JOIN fact_gross_price g
            ON g.fiscal_year=get_fiscal_year(s.date)
    	AND g.product_code=s.product_code
	WHERE 
    	    customer_code=90002002 AND 
            get_fiscal_year(s.date)=2021     
	LIMIT 1000000;
