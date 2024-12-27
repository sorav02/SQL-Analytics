#1.	Get the Gross Sales Report for customer ‘Croma India’ The report should include columns: - 
	##Date
	##Product_code
	##Product
	##Variant
	##Sold_quantity
	##Gross_price


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






#2.	Generate Monthly Total Sales Report for ‘Croma’. The report should have 2 columns – 
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



#3.	Create a stored procedure to determine market badge. If total sold qty > 5 million then market is considered Gold else Silver.

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



#4.	Generate report for Top Markets, Top Products and Top Customers by Net Sales for FY 2021.


CREATE DATABASE VIEW for gross_sales and net_invoice_sales

## Top customers
	select ns.customer, round(sum(((1 - (po.discounts_pct + po.other_deductions_pct))*net_invoice_sales)/1000000),2) as 	net_sales
	from net_invoice_sales ns
	join fact_post_invoice_deductions po on
	ns.customer_code = po.customer_code and ns.product_code = po.product_code and ns.date = po.date 
	where ns.fiscal_year = 2021
	group by ns.customer
	order by net_sales desc;


## Top Markets
	select ns.market, round(sum(((1 - (po.discounts_pct + po.other_deductions_pct))*net_invoice_sales)/1000000),2) as 	net_sales
	from net_invoice_sales ns
	join fact_post_invoice_deductions po on
	ns.customer_code = po.customer_code and ns.product_code = po.product_code and ns.date = po.date 
	where ns.fiscal_year = 2021
	group by ns.market
	order by net_sales desc
	limit 3;

## Top Product
	select  p.product, round(sum(((1 - (po.discounts_pct + po.other_deductions_pct))*net_invoice_sales)/1000000),2) as 	net_sales
	from net_invoice_sales ns
	join fact_post_invoice_deductions po on
	ns.customer_code = po.customer_code and ns.product_code = po.product_code and ns.date = po.date
	join dim_product p on ns.product_code = p.product_code 
	where ns.fiscal_year = 2021
	group by p.product
	order by net_sales desc
	limit 3;


OR  

	CREATE PROCEDURE get_top_n_products_by_net_sales(
              in_fiscal_year int,
              in_top_n int
	)
	BEGIN
            select
                 product,
                 round(sum(net_sales)/1000000,2) as net_sales_mln
            from gdb041.net_sales
            where fiscal_year=in_fiscal_year
            group by product
            order by net_sales_mln desc
            limit in_top_n;
	END




#5.    For FY 2021, top 10 market by %net sales. 

	with cte1 as(
	select c.customer, round((sum(nsl.net_sales)/1000000),2) as total_sales from net_sales nsl
	join dim_customer c on nsl.customer_code = c.customer_code  
	where nsl.fiscal_year = 2021
	group by c.customer)

	select *, total_sales*100/sum(total_sales) over () as pct_sales
	from cte1
	order by total_sales desc;



#6.    Region wise % net sales breakdown by customers:

	with cte1 as(
	select c.customer, c.region, round((sum(nsl.net_sales)/1000000),2) as total_sales from net_sales nsl
	join dim_customer c on nsl.customer_code = c.customer_code  
	where nsl.fiscal_year = 2021
	group by c.customer, c.region)

	select *, total_sales*100/sum(total_sales) over (partition by region) as pct_sales
	from cte1
	order by region, total_sales desc;


