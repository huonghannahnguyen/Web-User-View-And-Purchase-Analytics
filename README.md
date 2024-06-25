# eCommerce Electronics Store Analytics - PostgreSQL
eCommerce events history in electronics store dataset link: https://www.kaggle.com/datasets/mkechinov/ecommerce-events-history-in-electronics-store

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

---------------------------- DATA ANALYTICS ----------------------------------------

-- Total views and unique visitors

    SELECT COUNT(user_id) FILTER (WHERE action ='view') AS total_views,
	         COUNT(DISTINCT user_id) AS unique_visitors,
	         COUNT(DISTINCT user_id) FILTER (WHERE dense_rank > 1) AS returned_visitors
    FROM (
           SELECT user_id, user_session, action, DENSE_RANK()OVER(PARTITION BY user_id ORDER BY user_session)
           FROM webstore
	  ) AS a;
   
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
    
-- Conversion rate

    SELECT 100.0 * COUNT(DISTINCT user_id) FILTER (WHERE action = 'purchase')::DECIMAL/
           COUNT(DISTINCT user_id) AS conversion_rate
    FROM webstore;
    
-- Products viewed per session

    SELECT AVG(products_viewed) AS products_viewed_per_session
    FROM (
           SELECT user_session, COUNT(*) AS products_viewed
           FROM webstore
           WHERE action = 'view'
           GROUP BY user_session
	  ) AS a;
   
-- Bounce rate

    SELECT 100.0* COUNT(user_session) FILTER(WHERE count = 1)::DECIMAL / 
	        COUNT(user_session) AS bounce_rate
    FROM (
          SELECT user_session, COUNT(user_session)
          FROM webstore
          WHERE action = 'view'
          GROUP BY user_session
    ) AS a;
    
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
   
-- Sales

    SELECT SUM(price) AS total_sales
    FROM webstore
    WHERE action = 'purchase';
    
-- Sales over time

    SELECT date, SUM(price) AS total_sales
    FROM webstore
    WHERE action = 'purchase'
    GROUP BY date;
    
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
   
-- Day in week views

    SELECT TO_CHAR(date,'DAY') AS day, COUNT(user_id)
    FROM webstore
    WHERE action = 'view'
    GROUP BY day;
    
-- Day in week views by hour

    SELECT day, time, SUM(count) AS total_views
    FROM (
          SELECT TO_CHAR(date,'DAY') AS day, DATE_TRUNC('hour', time) AS time, COUNT(*)
          FROM webstore
          WHERE action = 'view'
          GROUP BY day, time
	  ) AS a
    GROUP BY day, time;
    
-- Day in week purchases

    SELECT TO_CHAR(date,'DAY') AS day, COUNT(*)
    FROM webstore
    WHERE action = 'purchase'
    GROUP BY day;
    
-- Day in week purchases by hour

    SELECT day, time, SUM(count) AS total_purchase
    FROM (
          SELECT TO_CHAR(date,'DAY') AS day, DATE_TRUNC('hour', time) AS time, COUNT(*)
          FROM webstore
          WHERE action = 'purchase'
          GROUP BY day, time
	  ) AS a
    GROUP BY day, time;
