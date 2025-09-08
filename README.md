# **Olist E‑Commerce Data Analysis by Adrian Langberg**

**Goal:** Identify factors affecting customer satisfaction and delivery performance in Brazilian e‑commerce orders.  


🔗 Project Links

- **Dataset:** [Brazilian E‑Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)  
- **SQL Code & Analysis:** *(this repository)*  
- **Interactive Tableau Dashboard:** [View Dashboard](https://public.tableau.com/shared/ZBTD6HJQ3?:display_count=n&:origin=viz_share_link)  
- **Final Presentation:** [View PowerPoint](https://1drv.ms/p/c/2eea264b2751569d/EfihWiY1ZvNFlQd9hOOg_i0BtYPjxFD3Cln8KftUM6Gc-Q?e=Am8xQT)


🛠 Tools & Skills Used
SQL • Excel • Tableau • Data Cleaning • Data Visualization • Business Insights
📊 Key Insights
- Rio de Janeiro ranked among the lowest average review scores and highest late delivery rates.
- Delivery delays strongly correlated with lower customer satisfaction.
- Freight‑to‑price ratio impacted review scores in certain product categories.
📂 Project Structure
- Data Cleaning & Preparation – SQL scripts to join, filter, and clean datasets.
- Exploratory Data Analysis – State‑level summaries, delivery performance metrics, and review score trends.
- Visualization – Tableau dashboards for interactive exploration.
- Presentation – Executive summary with recommendations for operational improvements.


## **SQL Code & Analysis**


/******************************************************************
 STEP 1: STATE-LEVEL OVERVIEW
 Goal: Identify states with low review scores, long delivery, high late %, and volume
******************************************************************/

-- 1A. State-level summary: avg review, avg delivery days, percent late, volume

SELECT c.customer_state, ROUND(AVG(r.review_score),2) AS avg_review_score, ROUND(AVG(EXTRACT(EPOCH FROM eo.order_delivered_customer_date - eo.order_purchase_timestamp)/86400.0),2) AS avg_delivery_days, ROUND(COUNT() FILTER (WHERE eo.delivery_status = 'Late') * 100.0 / COUNT(),2) AS pct_late, COUNT() AS total_orders, COUNT() FILTER (WHERE r.review_score IS NOT NULL) AS total_reviews FROM enriched_orders eo JOIN customers c ON eo.customer_id = c.customer_id LEFT JOIN reviews r ON eo.order_id = r.order_id WHERE eo.order_delivered_customer_date IS NOT NULL GROUP BY c.customer_state ORDER BY avg_review_score ASC; -- low scoring states first

-- Rio de janeiro is a top state priority to analyze
-- it is in the top 5 of lowest avg review scores (3.88) and has a total of 13, 936
-- reviews, lets run queries to respond to the problems


/******************************************************************
 STEP 2: RJ BASE TABLES
 Goal: Establish RJ-scoped base CTEs for orders, reviews, and items
******************************************************************/

-- RJ base: one row per (order, review, item) for customers in RJ

WITH rj_orders AS ( SELECT o.* FROM orders o JOIN customers c ON c.customer_id = o.customer_id WHERE c.customer_state = 'RJ' ), rj_reviews AS ( SELECT r., o.customer_id, o.order_purchase_timestamp, o.order_delivered_customer_date, o.order_estimated_delivery_date FROM reviews r JOIN rj_orders o ON o.order_id = r.order_id WHERE r.review_score BETWEEN 1 AND 5 ), rj_items AS ( SELECT oi., p.product_category_name, s.seller_id, s.seller_city, s.seller_state FROM order_items oi JOIN products p ON p.product_id = oi.product_id JOIN sellers s ON s.seller_id = oi.seller_id JOIN rj_orders o ON o.order_id = oi.order_id ) SELECT 1;


/******************************************************************
 STEP 3: REVIEW DISTRIBUTIONS (RJ)
 Goal: Volume by score; presence of comments by score/threshold
******************************************************************/

-- Bar chart: count of 1–5 star reviews (RJ) using enriched_orders

WITH rj_orders AS ( SELECT eo.* FROM enriched_orders eo JOIN customers c ON c.customer_id = eo.customer_id WHERE c.customer_state = 'RJ' ), rj_reviews AS ( SELECT r.* FROM reviews r JOIN rj_orders eo ON eo.order_id = r.order_id WHERE r.review_score BETWEEN 1 AND 5 ) SELECT review_score, COUNT() AS review_count, ROUND( COUNT() * 100.0 / SUM(COUNT(*)) OVER (), 2 ) AS percentage FROM rj_reviews GROUP BY review_score ORDER BY review_score;

-- review score breakdown

WITH review_comment_stats AS ( SELECT r.review_score, COUNT() AS total_reviews, COUNT() FILTER (WHERE review_comment_message IS NOT NULL AND review_comment_message <> '') AS reviews_with_comments FROM reviews r JOIN enriched_orders eo ON eo.order_id = r.order_id GROUP BY r.review_score ) SELECT review_score, total_reviews, reviews_with_comments, ROUND(reviews_with_comments * 100.0 / total_reviews, 2) AS pct_with_comments FROM review_comment_stats ORDER BY review_score;

-- Low review scores %

WITH low_reviews AS ( SELECT r.review_id, r.review_score, r.review_comment_message FROM reviews r JOIN enriched_orders eo ON eo.order_id = r.order_id WHERE r.review_score <= 2 ) SELECT COUNT() AS total_low_reviews, COUNT() FILTER (WHERE review_comment_message IS NOT NULL AND review_comment_message <> '') AS reviews_with_comments, ROUND( COUNT() FILTER (WHERE review_comment_message IS NOT NULL AND review_comment_message <> '') * 100.0 / COUNT(), 2 ) AS pct_with_comments FROM low_reviews;

-- mid to low reviews

WITH low_reviews AS ( SELECT r.review_id, r.review_score, r.review_comment_message FROM reviews r JOIN enriched_orders eo ON eo.order_id = r.order_id WHERE r.review_score <= 3 ) SELECT COUNT() AS total_low_reviews, COUNT() FILTER (WHERE review_comment_message IS NOT NULL AND review_comment_message <> '') AS reviews_with_comments, ROUND( COUNT() FILTER (WHERE review_comment_message IS NOT NULL AND review_comment_message <> '') * 100.0 / COUNT(), 3 ) AS pct_with_comments FROM low_reviews;


/******************************************************************
 STEP 4: DELIVERY PERFORMANCE VS REVIEW SCORES (RJ)
 Goal: Link delivery metrics (avg days, % late, days late) with score
******************************************************************/

-- Part 2

WITH rj_orders AS ( SELECT eo., r.review_score FROM enriched_orders eo JOIN customers c ON c.customer_id = eo.customer_id LEFT JOIN reviews r ON eo.order_id = r.order_id WHERE c.customer_state = 'RJ' AND r.review_score BETWEEN 1 AND 5 AND eo.order_delivered_customer_date IS NOT NULL ) SELECT review_score, COUNT() AS reviews, ROUND(AVG(EXTRACT(EPOCH FROM (order_delivered_customer_date - order_purchase_timestamp))/86400.0)::numeric,2) AS avg_delivery_days, ROUND(COUNT() FILTER (WHERE delivery_status = 'Late') * 100.0 / COUNT(),2) AS pct_late, ROUND(AVG(EXTRACT(EPOCH FROM (order_delivered_customer_date - order_estimated_delivery_date))/86400.0) FILTER (WHERE delivery_status = 'Late')::numeric,2) AS avg_days_late FROM rj_orders GROUP BY review_score ORDER BY review_score;

-- Low Review scores and comment message

SELECT r.review_score, r.review_comment_message, eo.delivery_status, ROUND(EXTRACT(EPOCH FROM (eo.order_delivered_customer_date - eo.order_purchase_timestamp)) / 86400.0, 2) AS delivery_days, ROUND(EXTRACT(EPOCH FROM (eo.order_delivered_customer_date - eo.order_estimated_delivery_date)) / 86400.0, 2) AS days_early_late FROM reviews r JOIN enriched_orders eo ON eo.order_id = r.order_id WHERE r.review_score <= 2 AND r.review_comment_message IS NOT NULL AND r.review_comment_message <> '';


/******************************************************************
 STEP 5: CATEGORY ANALYSIS (RJ)
 Goal: Identify categories tied to low scores and delivery issues
******************************************************************/

-- Categories most associated with low reviews (1–2★)

WITH rj_orders AS ( SELECT o.* FROM orders o JOIN customers c ON c.customer_id = o.customer_id WHERE c.customer_state = 'RJ' ), rj AS ( SELECT r.review_score, r.order_id, p.product_category_name FROM reviews r JOIN rj_orders o ON o.order_id = r.order_id JOIN order_items oi ON oi.order_id = r.order_id JOIN products p ON p.product_id = oi.product_id WHERE r.review_score BETWEEN 1 AND 5 ) SELECT product_category_name, COUNT() AS review_items, ROUND(AVG(review_score)::numeric,2) AS avg_score, SUM(CASE WHEN review_score <= 2 THEN 1 ELSE 0 END) AS low_reviews, ROUND(100.0 * SUM(CASE WHEN review_score <= 2 THEN 1 ELSE 0 END) / COUNT(), 1) AS pct_low FROM rj GROUP BY product_category_name HAVING COUNT(*) >= 50 -- adjust threshold to stabilize ORDER BY pct_low DESC, review_items DESC LIMIT 20;

-- Enhanced: adds avgh delivery days, avg deays eealy / late, pct late
-- Categories most associated with low reviews (1–2★) for RJ customers

WITH rj_orders AS ( SELECT eo.order_id, eo.customer_id, eo.order_purchase_timestamp, eo.order_delivered_customer_date, eo.order_estimated_delivery_date, eo.delivery_status FROM enriched_orders eo JOIN customers c ON c.customer_id = eo.customer_id WHERE c.customer_state = 'RJ' ), rj AS ( SELECT r.review_score, r.order_id, p.product_category_name, o.order_purchase_timestamp, o.order_delivered_customer_date, o.order_estimated_delivery_date, o.delivery_status FROM reviews r JOIN rj_orders o ON o.order_id = r.order_id JOIN order_items oi ON oi.order_id = r.order_id JOIN products p ON p.product_id = oi.product_id WHERE r.review_score BETWEEN 1 AND 5 ) SELECT product_category_name, COUNT() AS review_items, ROUND(AVG(review_score)::numeric, 2) AS avg_score, SUM(CASE WHEN review_score <= 2 THEN 1 ELSE 0 END) AS low_reviews, ROUND(100.0 * SUM(CASE WHEN review_score <= 2 THEN 1 ELSE 0 END) / COUNT(), 1) AS pct_low, -- delivery metrics ROUND( AVG(EXTRACT(EPOCH FROM (order_delivered_customer_date - order_purchase_timestamp)) / 86400.0)::numeric, 2 ) AS avg_delivery_days, ROUND( AVG(EXTRACT(EPOCH FROM (order_delivered_customer_date - order_estimated_delivery_date)) / 86400.0)::numeric, 2 ) AS avg_days_early_late, ROUND( COUNT() FILTER (WHERE delivery_status = 'Late') * 100.0 / COUNT(), 2 ) AS pct_late, MIN(order_purchase_timestamp) AS example_order_purchase_timestamp FROM rj GROUP BY product_category_name HAVING COUNT(*) >= 50 -- adjust threshold to stabilize ORDER BY pct_low DESC, review_items DESC LIMIT 20;

-- mega enhanced: adds most common order purchas timespamp

-- Categories most associated with low reviews (1–2★) for RJ customers

WITH rj_orders AS ( SELECT eo.order_id, eo.customer_id, eo.order_purchase_timestamp, eo.order_delivered_customer_date, eo.order_estimated_delivery_date, eo.delivery_status FROM enriched_orders eo JOIN customers c ON c.customer_id = eo.customer_id WHERE c.customer_state = 'RJ' ), rj AS ( SELECT r.review_score, r.order_id, p.product_category_name, o.order_purchase_timestamp, o.order_delivered_customer_date, o.order_estimated_delivery_date, o.delivery_status FROM reviews r JOIN rj_orders o ON o.order_id = r.order_id JOIN order_items oi ON oi.order_id = r.order_id JOIN products p ON p.product_id = oi.product_id WHERE r.review_score BETWEEN 1 AND 5 ), category_month_counts AS ( SELECT product_category_name, to_char(date_trunc('month', order_purchase_timestamp), 'YYYY-MM') AS month_year, COUNT() AS orders_in_month, RANK() OVER ( PARTITION BY product_category_name ORDER BY COUNT() DESC ) AS month_rank FROM rj GROUP BY product_category_name, date_trunc('month', order_purchase_timestamp) ) SELECT main.product_category_name, COUNT() AS review_items, ROUND(AVG(review_score)::numeric, 2) AS avg_score, SUM(CASE WHEN review_score <= 2 THEN 1 ELSE 0 END) AS low_reviews, ROUND(100.0 * SUM(CASE WHEN review_score <= 2 THEN 1 ELSE 0 END) / COUNT(), 1) AS pct_low, -- delivery metrics ROUND( AVG(EXTRACT(EPOCH FROM (order_delivered_customer_date - order_purchase_timestamp)) / 86400.0)::numeric, 2 ) AS avg_delivery_days, ROUND( AVG(EXTRACT(EPOCH FROM (order_delivered_customer_date - order_estimated_delivery_date)) / 86400.0)::numeric, 2 ) AS avg_days_early_late, ROUND( COUNT() FILTER (WHERE delivery_status = 'Late') * 100.0 / COUNT(), 2 ) AS pct_late, cm.month_year AS top_month_year_for_orders FROM rj AS main JOIN category_month_counts cm ON main.product_category_name = cm.product_category_name AND cm.month_rank = 1 GROUP BY main.product_category_name, cm.month_year HAVING COUNT(*) >= 50 -- adjust threshold to stabilize ORDER BY pct_low DESC, review_items DESC LIMIT 20;


/******************************************************************
 STEP 6: SELLER ANALYSIS (RJ)
 Goal: Identify sellers tied to low scores and their delivery metrics
******************************************************************/

-- Sellers most associated with low reviews (1–2★)

WITH rj_orders AS ( SELECT o.* FROM orders o JOIN customers c ON c.customer_id = o.customer_id WHERE c.customer_state = 'RJ' ), rj AS ( SELECT r.review_score, r.order_id, oi.seller_id FROM reviews r JOIN rj_orders o ON o.order_id = r.order_id JOIN order_items oi ON oi.order_id = r.order_id WHERE r.review_score BETWEEN 1 AND 5 ) SELECT seller_id, COUNT() AS review_items, ROUND(AVG(review_score)::numeric,2) AS avg_score, SUM(CASE WHEN review_score <= 2 THEN 1 ELSE 0 END) AS low_reviews, ROUND(100.0 * SUM(CASE WHEN review_score <= 2 THEN 1 ELSE 0 END) / COUNT(), 1) AS pct_low FROM rj GROUP BY seller_id HAVING COUNT(*) >= 50 ORDER BY pct_low DESC, review_items DESC LIMIT 20;

-- Enhanced/ uses enriched orders to add information
-- Sellers most associated with low reviews (1–2★) for RJ customers

WITH rj_orders AS ( SELECT eo.order_id, eo.customer_id, eo.order_purchase_timestamp, eo.order_delivered_customer_date, eo.order_estimated_delivery_date, eo.delivery_status FROM enriched_orders eo JOIN customers c ON c.customer_id = eo.customer_id WHERE c.customer_state = 'RJ' ), rj AS ( SELECT r.review_score, r.order_id, oi.seller_id, o.order_purchase_timestamp, o.order_delivered_customer_date, o.order_estimated_delivery_date, o.delivery_status FROM reviews r JOIN rj_orders o ON o.order_id = r.order_id JOIN order_items oi ON oi.order_id = r.order_id WHERE r.review_score BETWEEN 1 AND 5 ) SELECT seller_id, COUNT() AS review_items, ROUND(AVG(review_score)::numeric, 2) AS avg_score, SUM(CASE WHEN review_score <= 2 THEN 1 ELSE 0 END) AS low_reviews, ROUND(100.0 * SUM(CASE WHEN review_score <= 2 THEN 1 ELSE 0 END) / COUNT(), 1) AS pct_low, -- delivery metrics ROUND( AVG(EXTRACT(EPOCH FROM (order_delivered_customer_date - order_purchase_timestamp)) / 86400.0)::numeric, 2 ) AS avg_delivery_days, ROUND( AVG(EXTRACT(EPOCH FROM (order_delivered_customer_date - order_estimated_delivery_date)) / 86400.0)::numeric, 2 ) AS avg_days_early_late, ROUND( COUNT() FILTER (WHERE delivery_status = 'Late') * 100.0 / COUNT(), 2 ) AS pct_late FROM rj GROUP BY seller_id HAVING COUNT(*) >= 50 ORDER BY pct_low DESC, review_items DESC LIMIT 20;


/******************************************************************
 STEP 7: FREIGHT & PRICE CHECKS (RJ)
 Goal: Sanity check whether price/freight relate to scores
******************************************************************/

-- Freight & price stress vs reviews - no difference

WITH rj_orders AS ( SELECT eo.order_id, eo.customer_id, eo.order_purchase_timestamp, eo.order_delivered_customer_date, eo.order_estimated_delivery_date, eo.delivery_status FROM enriched_orders eo JOIN customers c ON c.customer_id = eo.customer_id WHERE c.customer_state = 'RJ' ), rj AS ( SELECT r.review_score, oi.price, oi.freight_value FROM reviews r JOIN rj_orders o ON o.order_id = r.order_id JOIN order_items oi ON oi.order_id = r.order_id WHERE r.review_score BETWEEN 1 AND 5 ) SELECT review_score, ROUND(AVG(price)::numeric, 2) AS avg_price, ROUND(AVG(freight_value)::numeric, 2) AS avg_freight, ROUND(AVG(CASE WHEN price > 0 THEN freight_value / price END)::numeric, 3) AS avg_freight_to_price FROM rj GROUP BY review_score ORDER BY review_score;


/******************************************************************
 STEP 8: LOW REVIEW COMMENTS (RJ)
 Goal: Pull low-score comments and basic RJ late/early review counts
******************************************************************/

-- 1. Count total reviews for Late vs Early orders

SELECT eo.delivery_status, COUNT(DISTINCT r.review_id) AS total_reviews FROM enriched_orders eo JOIN customers c ON eo.customer_id = c.customer_id JOIN reviews r ON eo.order_id = r.order_id WHERE c.customer_state = 'RJ' AND eo.delivery_status IN ('Late', 'Early') AND r.review_score <= 2 GROUP BY eo.delivery_status ORDER BY eo.delivery_status;

-- Late orders — reviews vs comments

SELECT 'Late' AS delivery_status, COUNT(DISTINCT r.review_id) AS total_reviews, COUNT(DISTINCT r.review_id) FILTER ( WHERE e.english_comment IS NOT NULL AND e.english_comment <> '' ) AS reviews_with_comments FROM enriched_orders eo JOIN customers c ON eo.customer_id = c.customer_id JOIN reviews r ON eo.order_id = r.order_id LEFT JOIN english e ON r.review_id = e.review_id WHERE c.customer_state = 'RJ' AND eo.delivery_status = 'Late' AND r.review_score <= 2;

-- Early orders — reviews vs comments

SELECT COUNT(DISTINCT eo.order_id) AS total_orders_score_le_2_rj,
COUNT(DISTINCT eo.order_id) FILTER (
    WHERE r.review_comment_message IS NOT NULL
      AND r.review_comment_message <> ''
) AS total_orders_score_le_2_with_comment,
COUNT(DISTINCT eo.order_id) FILTER (
    WHERE eo.delivery_status ILIKE '%late%'
      AND r.review_comment_message IS NOT NULL
      AND r.review_comment_message <> ''
) AS total_orders_score_le_2_late_with_comment,
COUNT(DISTINCT eo.order_id) FILTER (
    WHERE eo.delivery_status ILIKE '%early%'
      AND r.review_comment_message IS NOT NULL
      AND r.review_comment_message <> ''
) AS total_orders_score_le_2_early_with_comment
FROM enriched_orders eo JOIN customers c ON eo.customer_id = c.customer_id JOIN reviews r ON eo.order_id = r.order_id WHERE c.customer_state = 'RJ' AND r.review_score <= 2;


/******************************************************************
 STEP 9: ISSUE KEYWORDS CLASSIFICATION (RJ)
 Goal: Build keyword-to-category map and classify low-score comments
******************************************************************/

DROP TABLE IF EXISTS issue_keywords; CREATE TABLE issue_keywords ( keyword TEXT, category TEXT );


INSERT INTO issue_keywords (keyword, category) VALUES
-- A — Late / Not delivered
('not received', 'late / not delivered'),
('did not receive', 'late / not delivered'),
('didnt receive', 'late / not delivered'),
('not delivered', 'late / not delivered'),
('product not delivered', 'late / not delivered'),
('still waiting', 'late / not delivered'),
('deadline', 'late / not delivered'),
('delivery deadline', 'late / not delivered'),
('delivery date', 'late / not delivered'),
('deadline has passed', 'late / not delivered'),
('delivery was scheduled', 'late / not delivered'),
('overdue', 'late / not delivered'),
('late delivery', 'late / not delivered'),
('long delay', 'late / not delivered'),
('delivery deadline expired', 'late / not delivered'),
('not arrived', 'late / not delivered'),
('waiting for product', 'late / not delivered'),
('never arrived', 'late / not delivered'),
('stolen', 'late / not delivered'),
('missing (by carrier)', 'late / not delivered'),
('pick it up at the post office', 'late / not delivered'),
('post office protocol', 'late / not delivered'),
('tracking stalled', 'late / not delivered'),
('stuck in', 'late / not delivered'),

-- B — Missing / Wrong item / Partial delivery
('missing', 'missing / wrong item'),
('received one', 'missing / wrong item'),
('received only one', 'missing / wrong item'),
('only one arrived', 'missing / wrong item'),
('one missing', 'missing / wrong item'),
('bought two and received one', 'missing / wrong item'),
('wrong item', 'missing / wrong item'),
('different product', 'missing / wrong item'),
('wrong size', 'missing / wrong item'),
('wrong color', 'missing / wrong item'),
('part missing', 'missing / wrong item'),
('some parts missing', 'missing / wrong item'),

-- C — Damaged / Defective
('broken', 'damaged / defective'),
('arrived broken', 'damaged / defective'),
('defective', 'damaged / defective'),
('damaged', 'damaged / defective'),
('cracked', 'damaged / defective'),
('doesn''t work', 'damaged / defective'),
('doesnt work', 'damaged / defective'),
('malfunction', 'damaged / defective'),
('stopped working', 'damaged / defective'),
('returned it', 'damaged / defective'),

-- D — Quality
('poor quality', 'quality'),
('bad quality', 'quality'),
('weak', 'quality'),
('not as described', 'quality'),
('terrible quality', 'quality'),
('disappointed with the product', 'quality'),
('wouldn''t recommend', 'quality'),

-- E — Refund / Return / Billing / Cancellations
('refund', 'refund / return / billing'),
('money back', 'refund / return / billing'),
('voucher', 'refund / return / billing'),
('cancel', 'refund / return / billing'),
('reimbursement', 'refund / return / billing'),
('charged', 'refund / return / billing'),
('invoice', 'refund / return / billing'),
('paid and not received', 'refund / return / billing'),
('cancelled but charged', 'refund / return / billing'),

-- F — Customer service / Communication
('no response', 'customer service / communication'),
('no one responds', 'customer service / communication'),
('did not respond', 'customer service / communication'),
('no contact', 'customer service / communication'),
('cannot contact', 'customer service / communication'),
('no update', 'customer service / communication'),
('no explanation', 'customer service / communication'),
('no answer', 'customer service / communication'),
('ignored', 'customer service / communication'),

-- G — Carrier / Post office
('post office', 'carrier / post office'),
('carrier', 'carrier / post office'),
('pac', 'carrier / post office'),
('tracking', 'carrier / post office'),
('transit', 'carrier / post office'),
('stuck', 'carrier / post office'),
('returned to sender', 'carrier / post office'),
('lost by post office', 'carrier / post office'),
('delivered to post office', 'carrier / post office'),

-- H — Other / Misc
('misleading advertising', 'other / misc'),
('not trustworthy', 'other / misc'),
('refund request not processed', 'other / misc'),
('cancelled order', 'other / misc'),
('duplicate charge', 'other / misc'),
('gift didn''t arrive', 'other / misc');

WITH cleaned_reviews AS ( SELECT r.review_id, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> '' ) SELECT ik.category, COUNT(DISTINCT cr.review_id) AS review_count FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%' GROUP BY ik.category ORDER BY review_count DESC;



-- Master query Template

WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> '' ), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%' ), review_items AS ( SELECT ci.review_id, oi.seller_id, p.product_category_name FROM categorized_reviews ci JOIN order_items oi ON ci.order_id = oi.order_id JOIN products p ON oi.product_id = p.product_id ) SELECT ri.seller_id, ri.product_category_name, cr.category, COUNT(DISTINCT ri.review_id) AS review_count FROM review_items ri JOIN categorized_reviews cr ON ri.review_id = cr.review_id GROUP BY cr.category, ri.seller_id, ri.product_category_name ORDER BY cr.category, review_count DESC;


/******************************************************************
 STEP 10: RJ DEEP DIVES — LATE / NOT DELIVERED (RJ)
 Goal: Late/not delivered by seller, category, city, timeline, month
******************************************************************/

-- Sellers: Late / Not Delivered


WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> ''), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%'), review_items AS ( SELECT ci.review_id, oi.seller_id, p.product_category_name FROM categorized_reviews ci JOIN order_items oi ON ci.order_id = oi.order_id JOIN products p ON oi.product_id = p.product_id) SELECT ri.seller_id, COUNT(DISTINCT ri.review_id) AS review_count FROM review_items ri JOIN categorized_reviews cr ON ri.review_id = cr.review_id WHERE cr.category = 'late / not delivered' GROUP BY ri.seller_id ORDER BY review_count DESC;

-- enhanced

WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> '' ), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%' ), review_items AS ( SELECT ci.review_id, oi.seller_id, p.product_category_name FROM categorized_reviews ci JOIN order_items oi ON ci.order_id = oi.order_id JOIN products p ON oi.product_id = p.product_id ), -- Count reviews per seller & category category_counts AS ( SELECT ri.seller_id, ri.product_category_name, COUNT(DISTINCT ri.review_id) AS category_review_count FROM review_items ri JOIN categorized_reviews cr ON ri.review_id = cr.review_id WHERE cr.category = 'late / not delivered' GROUP BY ri.seller_id, ri.product_category_name ), -- Rank categories per seller ranked_categories AS ( SELECT seller_id, product_category_name, category_review_count, ROW_NUMBER() OVER ( PARTITION BY seller_id ORDER BY category_review_count DESC ) AS rn FROM category_counts ) -- Final output: seller, total review count, top category SELECT ri.seller_id, COUNT(DISTINCT ri.review_id) AS review_count, rc.product_category_name AS top_product_category_name FROM review_items ri JOIN categorized_reviews cr ON ri.review_id = cr.review_id JOIN ranked_categories rc ON ri.seller_id = rc.seller_id AND rc.rn = 1 WHERE cr.category = 'late / not delivered' GROUP BY ri.seller_id, rc.product_category_name ORDER BY review_count DESC;

-- Top Product Category Name

WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> ''), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%'), review_items AS ( SELECT ci.review_id, oi.seller_id, p.product_category_name FROM categorized_reviews ci JOIN order_items oi ON ci.order_id = oi.order_id JOIN products p ON oi.product_id = p.product_id) SELECT ri.product_category_name, COUNT(DISTINCT ri.review_id) AS review_count FROM review_items ri JOIN categorized_reviews cr ON ri.review_id = cr.review_id WHERE cr.category = 'late / not delivered' GROUP BY ri.product_category_name ORDER BY review_count DESC;

WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> '' ), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%' ), review_items AS ( SELECT ci.review_id, oi.seller_id, p.product_category_name FROM categorized_reviews ci JOIN order_items oi ON ci.order_id = oi.order_id JOIN products p ON oi.product_id = p.product_id ) SELECT ri.seller_id, ri.product_category_name, COUNT(DISTINCT ri.review_id) AS review_count FROM review_items ri JOIN categorized_reviews cr ON ri.review_id = cr.review_id WHERE cr.category = 'late / not delivered' GROUP BY ri.seller_id, ri.product_category_name ORDER BY review_count DESC;

-- Cities

WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> '' ), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%' ), review_cities AS ( SELECT ci.review_id, c.customer_city FROM categorized_reviews ci JOIN enriched_orders eo ON ci.order_id = eo.order_id JOIN customers c ON eo.customer_id = c.customer_id WHERE c.customer_state = 'RJ' ) SELECT rc.customer_city, COUNT(DISTINCT rc.review_id) AS review_count FROM review_cities rc JOIN categorized_reviews cr ON rc.review_id = cr.review_id WHERE cr.category = 'late / not delivered' GROUP BY rc.customer_city ORDER BY review_count DESC;


-- Timeline

WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, r.review_creation_date, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> '' ), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, cr.review_creation_date, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%' ) SELECT DATE_TRUNC('month', cr.review_creation_date) AS month, COUNT(DISTINCT cr.review_id) AS review_count FROM categorized_reviews cr WHERE cr.category = 'late / not delivered' GROUP BY DATE_TRUNC('month', cr.review_creation_date) ORDER BY review_count DESC;

-- Template for month

WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, r.review_creation_date, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> '' ), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, cr.review_creation_date, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%' WHERE ik.category = 'late / not delivered' ), review_items AS ( SELECT cr.review_id, cr.review_creation_date, oi.seller_id, p.product_category_name FROM categorized_reviews cr JOIN order_items oi ON cr.order_id = oi.order_id JOIN products p ON oi.product_id = p.product_id WHERE DATE_TRUNC('month', cr.review_creation_date) = DATE '2018-03-01' -- change month here ) -- Top sellers SELECT seller_id, COUNT(DISTINCT review_id) AS review_count FROM review_items GROUP BY seller_id ORDER BY review_count DESC LIMIT 10;

-- Top product categories

SELECT product_category_name, COUNT(DISTINCT review_id) AS review_count FROM review_items GROUP BY product_category_name ORDER BY review_count DESC LIMIT 10;

-- March 2018

WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, r.review_creation_date, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> '' ), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, cr.review_creation_date, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%' WHERE ik.category = 'late / not delivered' ), review_items AS ( SELECT cr.review_id, cr.review_creation_date, oi.seller_id, p.product_category_name FROM categorized_reviews cr JOIN order_items oi ON cr.order_id = oi.order_id JOIN products p ON oi.product_id = p.product_id -- Month filter: March 2018 WHERE DATE_TRUNC('month', cr.review_creation_date) = DATE '2018-03-01' ) -- Top sellers in March 2018 SELECT seller_id, COUNT(DISTINCT review_id) AS review_count FROM review_items GROUP BY seller_id ORDER BY review_count DESC LIMIT 10;

-- Top product categories in March 2018

SELECT product_category_name, COUNT(DISTINCT review_id) AS review_count FROM review_items GROUP BY product_category_name ORDER BY review_count DESC LIMIT 10;

-- Category Name (March 2018)

WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, r.review_creation_date, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> '' ), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, cr.review_creation_date, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%' WHERE ik.category = 'late / not delivered' ), review_items AS ( SELECT cr.review_id, cr.review_creation_date, oi.seller_id, p.product_category_name FROM categorized_reviews cr JOIN order_items oi ON cr.order_id = oi.order_id JOIN products p ON oi.product_id = p.product_id -- Month filter: March 2018 WHERE DATE_TRUNC('month', cr.review_creation_date) = DATE '2018-03-01' ) SELECT product_category_name, COUNT(DISTINCT review_id) AS review_count FROM review_items GROUP BY product_category_name ORDER BY review_count DESC LIMIT 10;

-- Date analysis
-- Spike days

WITH daily_counts AS ( SELECT r.review_creation_date::date AS review_date, COUNT(DISTINCT r.review_id) AS review_count FROM reviews r JOIN english e ON r.review_id = e.review_id JOIN issue_keywords ik ON LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) ILIKE '%' || ik.keyword || '%' WHERE r.review_score <= 2 AND ik.category = 'late / not delivered' GROUP BY r.review_creation_date::date ) SELECT * FROM daily_counts ORDER BY review_count DESC LIMIT 1000; -- adjust to see top spike days

-- Pull seller/product details for those days

WITH cleaned_reviews AS ( SELECT r.review_id, r.order_id, r.review_creation_date::date AS review_date, LOWER(regexp_replace(e.english_comment, '[^\w\s]+', '', 'g')) AS cleaned_comment FROM reviews r JOIN english e ON r.review_id = e.review_id WHERE r.review_score <= 2 AND e.english_comment IS NOT NULL AND e.english_comment <> '' ), categorized_reviews AS ( SELECT cr.review_id, cr.order_id, cr.review_date, ik.category FROM cleaned_reviews cr JOIN issue_keywords ik ON cr.cleaned_comment ILIKE '%' || ik.keyword || '%' WHERE ik.category = 'late / not delivered' ), review_details AS ( SELECT cr.review_id, cr.review_date, oi.seller_id, oi.product_id, p.product_category_name, p.product_name_length, p.product_weight_g FROM categorized_reviews cr JOIN order_items oi ON cr.order_id = oi.order_id JOIN products p ON oi.product_id = p.product_id ) SELECT review_date, seller_id, product_id, product_category_name, product_name_length, product_weight_g, COUNT(DISTINCT review_id) AS review_count FROM review_details WHERE review_date IN ( DATE '2017-12-15', DATE '2018-03-10', DATE '2018-03-15', DATE '2018-04-05', DATE '2018-04-20', DATE '2018-05-02' ) -- replace with your actual spike dates GROUP BY review_date, seller_id, product_id, product_category_name, product_name_length, product_weight_g ORDER BY review_date, review_count DESC;
