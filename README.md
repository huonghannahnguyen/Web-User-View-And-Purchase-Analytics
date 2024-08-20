# eCommerce Electronics Store Analytics - PostgreSQL
-- eCommerce events history in electronics store dataset link: https://www.kaggle.com/datasets/mkechinov/ecommerce-events-history-in-electronics-store

-- Tableau Dashboard: https://public.tableau.com/app/profile/huong6399/viz/Website-ViewsandPurchasesReport/Dashboard1

---------------------------- DATA CLEANING ----------------------------------------

-- Create table webstore

    CREATE TABLE webstore (event_day timestamp, action text, product_id int,
	                        category_id bigint, category_code varchar, brand varchar, price decimal, 
	                        user_id bigint, user_session varchar);
                         
-- Split event_day into 2 components: date and time

    ALTER TABLE webstore
    ADD COLUMN date date,
    ADD COLUMN time time;

    UPDATE webstore
    SET date = TO_CHAR(event_day, 'YYYY/MM/DD')::date,
	      time = TO_CHAR(event_day, 'HH24:MI:SS')::time;
       
-- Split category_code into 3 components: category, subcategory, and groups

    ALTER TABLE webstore
    ADD COLUMN category text,
    ADD COLUMN subcategory text,
    ADD COLUMN groups text;

    UPDATE webstore
    SET category = SPLIT_PART(category_code, '.',1),
	      subcategory = SPLIT_PART(category_code, '.',2),
	      groups = SPLIT_PART(category_code, '.',3);
       
-- Capitalize brand

    UPDATE webstore
    SET brand = UPPER(brand);

    ALTER TABLE webstore
    DROP COLUMN event_day,
    DROP COLUMN category_code;
    
![image](https://github.com/user-attachments/assets/858042ce-e95d-4cc0-8e6d-e703d49ea04c)

---------------------------- DATA ANALYTICS ----------------------------------------

-- Total views and unique visitors

    SELECT COUNT(user_id) FILTER (WHERE action ='view') AS total_views,
	         COUNT(DISTINCT user_id) AS unique_visitors,
	         COUNT(DISTINCT user_id) FILTER (WHERE dense_rank > 1) AS returned_visitors
    FROM (
           SELECT user_id, user_session, action, DENSE_RANK()OVER(PARTITION BY user_id ORDER BY user_session)
           FROM webstore
	  ) AS a;

![image](https://github.com/user-attachments/assets/8443511a-c057-40fb-9e04-c182d016c680)

-- Total views, unique visitors, and new visitors

    SELECT date, COUNT(user_id) FILTER (WHERE action = 'view') AS total_views, 
	         COUNT(DISTINCT user_id) AS unique_visitors,
	         COUNT(DISTINCT user_id) FILTER (WHERE dense_rank = 1) AS new_visitors,
	         COUNT(DISTINCT user_id) FILTER (WHERE dense_rank > 1) AS returned_visitors
    FROM (
           SELECT date, user_id, user_session, action, DENSE_RANK() OVER (PARTITION BY user_id 
	                ORDER BY user_session)
           FROM webstore
    ) AS a
    GROUP BY date;

![image](https://github.com/user-attachments/assets/11e5df85-8e7b-4e8c-bb34-bc87c6bb5bf2)

-- Conversion rate

    SELECT 100.0 * COUNT(DISTINCT user_id) FILTER (WHERE action = 'purchase')::DECIMAL/
           COUNT(DISTINCT user_id) AS conversion_rate
    FROM webstore;

![image](https://github.com/user-attachments/assets/d2f95967-2b16-4fec-966b-33200f3e7afe)

-- Products viewed per session

    SELECT AVG(products_viewed) AS products_viewed_per_session
    FROM (
           SELECT user_session, COUNT(*) AS products_viewed
           FROM webstore
           WHERE action = 'view'
           GROUP BY user_session
	  ) AS a;

![image](https://github.com/user-attachments/assets/85e7ac1c-bc9e-4570-a7a0-1971832cecec)

-- Bounce rate

    SELECT 100.0* COUNT(user_session) FILTER(WHERE count = 1)::DECIMAL / 
	        COUNT(user_session) AS bounce_rate
    FROM (
          SELECT user_session, COUNT(user_session)
          FROM webstore
          WHERE action = 'view'
          GROUP BY user_session
    ) AS a;

![image](https://github.com/user-attachments/assets/9618be40-cd53-4a51-b90d-180969d19a6f)

-- Average order value

    SELECT AVG(products_purchase) AS products_purchased_per_session,
	         AVG(total_sales) AS sales_per_session
    FROM (
           SELECT user_session, COUNT(*) AS products_purchase, 
	                SUM(price) AS total_sales
           FROM webstore
           WHERE action = 'purchase'
           GROUP BY user_session
	  ) AS a;

![image](https://github.com/user-attachments/assets/9db58fb3-dd1a-45a9-bf4c-7e32157d831d)

-- Sales

    SELECT SUM(price) AS total_sales
    FROM webstore
    WHERE action = 'purchase';

![image](https://github.com/user-attachments/assets/8d327733-15c7-4a69-97b9-e4e335a87f6b)
    
-- Sales over time

    SELECT date, SUM(price) AS total_sales
    FROM webstore
    WHERE action = 'purchase'
    GROUP BY date;

![image](https://github.com/user-attachments/assets/d29861a3-6e8c-4a46-86f9-bdf1b28eba3b)
    
-- Cart abandonment rate

    SELECT 100.0 * (COUNT(*) FILTER (WHERE action = 'cart')::DECIMAL -
	   COUNT(*) FILTER (WHERE action IN ('cart', 'view') AND lead = 'purchase'))/
           COUNT(*) FILTER(WHERE action = 'cart') AS cart_abandonment_rate
    FROM (
	   SELECT user_session, user_id, action, 
                  LEAD(action)OVER(PARTITION BY user_session, user_id ORDER BY date, time), 
                  date, time
	   FROM webstore
	  ) AS a;

![image](https://github.com/user-attachments/assets/b7517744-0ee0-4a0f-8516-25f4c438e63d)

-- Day in week views

    SELECT TO_CHAR(date,'DAY') AS day, COUNT(user_id)
    FROM webstore
    WHERE action = 'view'
    GROUP BY day;

![image](https://github.com/user-attachments/assets/f4197993-22cb-457b-b78d-9f58bb0acf08)

-- Day in week views by hour

    SELECT day, time, SUM(count) AS total_views
    FROM (
          SELECT TO_CHAR(date,'DAY') AS day, DATE_TRUNC('hour', time) AS time, COUNT(*)
          FROM webstore
          WHERE action = 'view'
          GROUP BY day, time
	  ) AS a
    GROUP BY day, time;

![image](https://github.com/user-attachments/assets/1bc1efdd-0a30-4567-b470-e21b291ba25d)
    
-- Day in week purchases

    SELECT TO_CHAR(date,'DAY') AS day, COUNT(*)
    FROM webstore
    WHERE action = 'purchase'
    GROUP BY day;

![image](https://github.com/user-attachments/assets/bf475a54-d93c-4897-adfd-28234b13254b)
    
-- Day in week purchases by hour

    SELECT day, time, SUM(count) AS total_purchase
    FROM (
          SELECT TO_CHAR(date,'DAY') AS day, DATE_TRUNC('hour', time) AS time, COUNT(*)
          FROM webstore
          WHERE action = 'purchase'
          GROUP BY day, time
	  ) AS a
    GROUP BY day, time;

![image](https://github.com/user-attachments/assets/1bfce15f-f030-430a-9443-2cc90de0c306)

-- Sales by Category

	WITH a AS (SELECT category, SUM(price)
	FROM webstore
	WHERE action = 'purchase'
	GROUP BY category)
	
	SELECT COALESCE(UPPER(category), 'NON-CATEGORIZED') AS category, sum
	FROM a;

![image](https://github.com/user-attachments/assets/aa8d2ff6-7aeb-4f0c-afb7-24f4ba05ed91)

-- Sales by Brand per Category

	WITH a AS (
 	SELECT category, brand, SUM(price)
	FROM webstore
	WHERE action = 'purchase'
	GROUP BY category, brand
	ORDER BY category
 	),

	b AS (
 	SELECT category, brand, sum, DENSE_RANK()OVER(PARTITION BY category ORDER BY sum DESC)
	FROM a
 	),

	c AS (
 	SELECT category, brand, sum
	FROM b
	WHERE dense_rank <= 5
 	)

	SELECT COALESCE(UPPER(category), 'NON-CATEGORIZED') AS category, 
	       COALESCE(brand, 'OFF BRAND') AS brand, sum AS sales
	FROM c
![image](https://github.com/user-attachments/assets/b5f87790-6250-47ff-8c99-411b5f6e812e)

![image](https://github.com/user-attachments/assets/a7850508-e521-4dd0-9408-df8cc2289dac)
