
ERD Diagram  
![image](https://github.com/user-attachments/assets/77825153-7336-4c57-9f72-c48c9670ba6c)





















#1.	Get the Gross Sales Report for customer ‘Croma India’ The report should include columns: -   
a. Date  
b. Product_code  
c. Product  
d. Variant  
e. Sold_quantity  
f. Gross_price  


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
![image](https://github.com/user-attachments/assets/8958da21-8c77-4d32-ac0d-9994276fb0ee)






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
![image](https://github.com/user-attachments/assets/bf5eab86-e8ad-43fd-965b-277d47dda940)



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
![image](https://github.com/user-attachments/assets/da9f8c83-7e5e-4476-9394-04e7a61f1678)


## Top Markets
	select ns.market, round(sum(((1 - (po.discounts_pct + po.other_deductions_pct))*net_invoice_sales)/1000000),2) as 	net_sales
	from net_invoice_sales ns
	join fact_post_invoice_deductions po on
	ns.customer_code = po.customer_code and ns.product_code = po.product_code and ns.date = po.date 
	where ns.fiscal_year = 2021
	group by ns.market
	order by net_sales desc
	limit 3;
![image](https://github.com/user-attachments/assets/3d6621dd-a142-4f60-b422-301bd6ba252c)

## Top Products
	select  p.product, round(sum(((1 - (po.discounts_pct + po.other_deductions_pct))*net_invoice_sales)/1000000),2) as 	net_sales
	from net_invoice_sales ns
	join fact_post_invoice_deductions po on
	ns.customer_code = po.customer_code and ns.product_code = po.product_code and ns.date = po.date
	join dim_product p on ns.product_code = p.product_code 
	where ns.fiscal_year = 2021
	group by p.product
	order by net_sales desc
	limit 3;
![image](https://github.com/user-attachments/assets/4a4588be-87ea-4a53-a237-7e843eb97fed)


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




#5.    For FY 2021, top 10 market by %net sales. The output should look like: 
![image](https://github.com/user-attachments/assets/0f26b56e-9aed-4171-bccf-2ff975aeb6f7)


	with cte1 as(
	select c.customer, round((sum(nsl.net_sales)/1000000),2) as total_sales from net_sales nsl
	join dim_customer c on nsl.customer_code = c.customer_code  
	where nsl.fiscal_year = 2021
	group by c.customer)

	select *, total_sales*100/sum(total_sales) over () as pct_sales
	from cte1
	order by total_sales desc;
![Screenshot (111)](https://github.com/user-attachments/assets/fb99a414-989c-4390-90c1-1ee0f8c27e21)



#6.    Region wise % net sales breakdown by customers:

	with cte1 as(
	select c.customer, c.region, round((sum(nsl.net_sales)/1000000),2) as total_sales from net_sales nsl
	join dim_customer c on nsl.customer_code = c.customer_code  
	where nsl.fiscal_year = 2021
	group by c.customer, c.region)

	select *, total_sales*100/sum(total_sales) over (partition by region) as pct_sales
	from cte1
	order by region, total_sales desc;
![image](https://github.com/user-attachments/assets/2b66bc0d-d1e9-4ead-89cd-afc83d8998d0) 
![image](https://github.com/user-attachments/assets/d916729f-b7d9-4522-9e71-bd45f3e1444c) 
![image](https://github.com/user-attachments/assets/d233ca03-eb53-4c01-8d61-458aa77a5c8b) 
![image](https://github.com/user-attachments/assets/6053e2f1-a78b-4274-bae7-a9ca6f70e657)





