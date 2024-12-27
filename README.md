# SQL-Analytics
Financial &amp; Sales Analytics using SQL

Problem Statement
#### Atliq Hardware is a manufacturer of Computer Peripherals and sells its product through its direct stores and through online & offline distribution platforms. The company uses SQL database to store its data. The company wants to use data insight to better understand its customer base and assess the sales of the products being sold. The management has given a list of requirements which needs to be fulfilled using SQL.

1.	Get the Gross Sales Report for customer ‘Croma India’ The report should include columns: - 
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


2.	Generate Monthly Total Sales Report for ‘Croma’. The report should have 2 columns – 
a.	Month
b.	Total Gross Sales Amount in that month.
Using stored procedure, monthly gross sales report can be generated for any customer	

CREATE PROCEDURE `get_monthly_gross_sales_for_customer`(
        	in_customer_codes TEXT
	)
	BEGIN
        	SELECT 
                    s.date, 
                    SUM(ROUND(s.sold_quantity*g.gross_price,2)) as monthly_sales
        	FROM fact_sales_monthly s
        	JOIN fact_gross_price g
               	    ON g.fiscal_year=get_fiscal_year(s.date)
                    AND g.product_code=s.product_code
        	WHERE 
                    FIND_IN_SET(s.customer_code, in_customer_codes) > 0
        	GROUP BY s.date
        	ORDER BY s.date DESC;
	END


 
3.	Create a stored procedure to determine market badge. If total sold qty > 5 million then market is considered Gold else Silver.

	CREATE PROCEDURE `get_market_badge`(
        	IN in_market VARCHAR(45),
        	IN in_fiscal_year YEAR,
        	OUT out_level VARCHAR(45)
	)
	BEGIN
             DECLARE qty INT DEFAULT 0;
    
    	     # Default market is India
    	     IF in_market = "" THEN
                  SET in_market="India";
             END IF;
    
    	     # Retrieve total sold quantity for a given market in a given year
             SELECT 
                  SUM(s.sold_quantity) INTO qty
             FROM fact_sales_monthly s
             JOIN dim_customer c
             ON s.customer_code=c.customer_code
             WHERE 
                  get_fiscal_year(s.date)=in_fiscal_year AND
                  c.market=in_market;
        
             # Determine Gold vs Silver status
             IF qty > 5000000 THEN
                  SET out_level = 'Gold';
             ELSE
                  SET out_level = 'Silver';
             END IF;
	END





4.	Generate report for Top Markets, Top Products and Top Customers by Net Sales.


CREATE DATABASE VIEW for gross_sales and net_invoice_sales

top customers:

select ns.customer, round(sum(((1 - (po.discounts_pct + po.other_deductions_pct))*net_invoice_sales)/1000000),2) as net_sales
from net_invoice_sales ns
join fact_post_invoice_deductions po on
ns.customer_code = po.customer_code and ns.product_code = po.product_code and ns.date = po.date 
where ns.fiscal_year = 2021
group by ns.customer
order by net_sales desc;

