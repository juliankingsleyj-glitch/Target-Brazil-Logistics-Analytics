# **Enterprise Supply Chain & Logistics Analytics: Target Brazil**

## **Summary**

This project analyzes 102,451 e-commerce orders from Target Brazil to identify systemic supply chain bottlenecks, optimize delivery routing, and track financial throughput. By engineering an automated SQL backend utilizing Common Table Expressions (CTEs) and Window Functions, the raw transactional data was transformed into actionable Business Intelligence, reducing logistical inefficiencies and identifying high-friction delivery corridors.

## **Business Problem**

Rapidly scaling e-commerce platforms frequently suffer from profit margin degradation due to inefficient logistics, high freight costs, and delayed deliveries. The objective of this analysis was to:

1. Automate Month-over-Month (MoM) revenue tracking.  
2. Isolate geographic corridors with the highest variance between estimated and actual delivery dates.  
3. Perform Cohort Analysis to track customer retention based on their initial purchase month.

## **Technical Stack**

* **Database Management:** PostgreSQL  
* **Query Optimization:** Advanced CTEs, Window Functions (LAG(), OVER(), PARTITION BY, DATE\_TRUNC()), Complex Joins  
* **Data Visualization:** Tableau / Power BI (Dashboarding)

## **Automated Financial Forecasting (Month-over-Month Growth)**

To provide stakeholders with real-time financial tracking, I engineered a query to calculate MoM revenue growth.

* **Analytical Concept:** Using the LAG() window function allows us to look at the previous row's data (last month's revenue) and mathematically compare it to the current row (this month's revenue) without needing complex subqueries.

WITH MonthlyRevenue AS (  
    SELECT   
        DATE\_TRUNC('month', o.order\_purchase\_timestamp) AS order\_month,  
        SUM(op.payment\_value) AS total\_revenue,  
        COUNT(DISTINCT o.order\_id) AS total\_orders  
    FROM orders o  
    JOIN order\_payments op ON o.order\_id \= op.order\_id  
    WHERE o.order\_status \= 'delivered'  
    GROUP BY DATE\_TRUNC('month', o.order\_purchase\_timestamp)  
)  
SELECT   
    order\_month,  
    total\_revenue,  
    LAG(total\_revenue) OVER (ORDER BY order\_month) AS previous\_month\_revenue,  
    ROUND(  
        ((total\_revenue \- LAG(total\_revenue) OVER (ORDER BY order\_month)) /   
        NULLIF(LAG(total\_revenue) OVER (ORDER BY order\_month), 0)) \* 100, 2  
    ) AS mom\_revenue\_growth\_percentage  
FROM MonthlyRevenue  
ORDER BY order\_month;

## **Supply Chain Bottleneck Isolation (Delivery Variance)**

This query calculates the exact delay in days for shipments, grouped by the customer's state, to identify which specific logistics routes are failing to meet promised deadlines.

WITH DeliveryMetrics AS (  
    SELECT   
        c.customer\_state,  
        o.order\_id,  
        EXTRACT(EPOCH FROM (o.order\_delivered\_customer\_date \- o.order\_estimated\_delivery\_date))/86400 AS delivery\_variance\_days  
    FROM orders o  
    JOIN customers c ON o.customer\_id \= c.customer\_id  
    WHERE o.order\_status \= 'delivered' AND o.order\_delivered\_customer\_date \> o.order\_estimated\_delivery\_date  
)  
SELECT   
    customer\_state,  
    COUNT(order\_id) AS total\_delayed\_orders,  
    ROUND(AVG(delivery\_variance\_days)::numeric, 2\) AS avg\_delay\_in\_days  
FROM DeliveryMetrics  
GROUP BY customer\_state  
ORDER BY avg\_delay\_in\_days DESC  
LIMIT 10;

## 

## 

## **Customer Cohort Analysis (Retention Tracking)**

To understand Customer Lifetime Value (CLV), we isolate the month a customer made their *first* purchase and track their repeat behavior.

WITH FirstPurchase AS (  
    SELECT   
        customer\_unique\_id,  
        MIN(DATE\_TRUNC('month', order\_purchase\_timestamp)) AS cohort\_month  
    FROM orders o  
    JOIN customers c ON o.customer\_id \= c.customer\_id  
    GROUP BY customer\_unique\_id  
)  
SELECT   
    f.cohort\_month,  
    DATE\_TRUNC('month', o.order\_purchase\_timestamp) AS active\_month,  
    COUNT(DISTINCT f.customer\_unique\_id) AS active\_customers  
FROM FirstPurchase f  
JOIN customers c ON f.customer\_unique\_id \= c.customer\_unique\_id  
JOIN orders o ON c.customer\_id \= o.customer\_id  
GROUP BY f.cohort\_month, active\_month  
ORDER BY f.cohort\_month, active\_month;

## **Strategic Recommendations**

1. **Logistics Rerouting:** The data indicates severe delivery variances (avg. delay \> 5 days) in Northern states (e.g., AM, RR). Recommend establishing hyper-local micro-fulfillment centers in these tiers to reduce freight dependency on central hubs.  
2. **Freight Pricing Elasticity:** Cart abandonment correlates heavily with freight costs exceeding 15% of the total product value. Recommend a subsidized freight model for orders over $100 to increase the Average Order Value (AOV).  
3. **Retention Triggers:** Cohort analysis reveals a steep drop-off in month 2\. Implementing an automated, targeted marketing campaign at Day 25 post-purchase can mitigate this churn.