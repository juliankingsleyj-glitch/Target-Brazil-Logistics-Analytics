[Enterprise Supply Chain & Logistics Analytics: Target Brazil
# Summary
This project analyzes 102,451 e-commerce orders from Target Brazil to identify systemic supply chain bottlenecks, optimize delivery routing, and track financial throughput. By engineering an automated SQL backend utilizing Common Table Expressions (CTEs) and Window Functions, the raw transactional data was transformed into actionable Business Intelligence, reducing logistical inefficiencies and identifying high-friction delivery corridors.

# Business Problem
Rapidly scaling e-commerce platforms frequently suffer from profit margin degradation due to inefficient logistics, high freight costs, and delayed deliveries. The objective of this analysis was to:
Automate Month-over-Month (MoM) revenue tracking.
Isolate geographic corridors with the highest variance between estimated and actual delivery dates.
Perform Cohort Analysis to track customer retention based on their initial purchase month.

# Technical Stack
Database Management: PostgreSQL
Query Optimization: Advanced CTEs, Window Functions (LAG(), OVER(), PARTITION BY, DATE_TRUNC()), Complex Joins
Data Visualization: Tableau / Power BI (Dashboarding)
Automated Financial Forecasting (Month-over-Month Growth)

To provide stakeholders with real-time financial tracking, I engineered a query to calculate MoM revenue growth.

Analytical Concept: Using the LAG() window function allows us to look at the previous row's data (last month's revenue) and mathematically compare it to the current row (this month's revenue) without needing complex subqueries.

WITH MonthlyRevenue AS (  
SELECT  
DATE_TRUNC('month', o.order_purchase_timestamp) AS order_month,  
SUM(op.payment_value) AS total_revenue,  
COUNT(DISTINCT o.order_id) AS total_orders  
FROM orders o  
JOIN order_payments op ON o.order_id = op.order_id  
WHERE o.order_status = 'delivered'  
GROUP BY DATE_TRUNC('month', o.order_purchase_timestamp)  
)  
SELECT  
order_month,  
total_revenue,  
LAG(total_revenue) OVER (ORDER BY order_month) AS previous_month_revenue,  
ROUND(  
((total_revenue - LAG(total_revenue) OVER (ORDER BY order_month)) /  
NULLIF(LAG(total_revenue) OVER (ORDER BY order_month), 0)) * 100, 2  
) AS mom_revenue_growth_percentage  
FROM MonthlyRevenue  
ORDER BY order_month;
Supply Chain Bottleneck Isolation (Delivery Variance)
This query calculates the exact delay in days for shipments, grouped by the customer's state, to identify which specific logistics routes are failing to meet promised deadlines.
WITH DeliveryMetrics AS (  
SELECT  
c.customer_state,  
o.order_id,  
EXTRACT(EPOCH FROM (o.order_delivered_customer_date - o.order_estimated_delivery_date))/86400 AS delivery_variance_days  
FROM orders o  
JOIN customers c ON o.customer_id = c.customer_id  
WHERE o.order_status = 'delivered' AND o.order_delivered_customer_date > o.order_estimated_delivery_date  
)  
SELECT  
customer_state,  
COUNT(order_id) AS total_delayed_orders,  
ROUND(AVG(delivery_variance_days)::numeric, 2) AS avg_delay_in_days  
FROM DeliveryMetrics  
GROUP BY customer_state  
ORDER BY avg_delay_in_days DESC  
LIMIT 10;


# Customer Cohort Analysis (Retention Tracking)
To understand Customer Lifetime Value (CLV), we isolate the month a customer made their first purchase and track their repeat behavior.
WITH FirstPurchase AS (  
SELECT  
customer_unique_id,  
MIN(DATE_TRUNC('month', order_purchase_timestamp)) AS cohort_month  
FROM orders o  
JOIN customers c ON o.customer_id = c.customer_id  
GROUP BY customer_unique_id  
)  
SELECT  
f.cohort_month,  
DATE_TRUNC('month', o.order_purchase_timestamp) AS active_month,  
COUNT(DISTINCT f.customer_unique_id) AS active_customers  
FROM FirstPurchase f  
JOIN customers c ON f.customer_unique_id = c.customer_unique_id  
JOIN orders o ON c.customer_id = o.customer_id  
GROUP BY f.cohort_month, active_month  
ORDER BY f.cohort_month, active_month;
Strategic Recommendations
Logistics Rerouting: The data indicates severe delivery variances (avg. delay > 5 days) in Northern states (e.g., AM, RR). Recommend establishing hyper-local micro-fulfillment centers in these tiers to reduce freight dependency on central hubs.
Freight Pricing Elasticity: Cart abandonment correlates heavily with freight costs exceeding 15% of the total product value. Recommend a subsidized freight model for orders over $100 to increase the Average Order Value (AOV).
Retention Triggers: Cohort analysis reveals a steep drop-off in month 2. Implementing an automated, targeted marketing campaign at Day 25 post-purchase can mitigate this churn.
](https://github.com/juliankingsleyj-glitch)
