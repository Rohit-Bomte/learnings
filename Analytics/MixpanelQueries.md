# Ecommerce Analytics: Advanced SQL Queries

This document contains high-performance, complex SQL queries designed to replicate Mixpanel's internal logic for ecommerce data.

## Database Schema (BigQuery/ClickHouse Style)

We assume two main tables:
1. `events`: A tall table containing all user actions.
2. `users`: A table containing user metadata (demographics).

**Common Columns for `events`**:
- `event_name` (STRING): e.g., 'Product Viewed', 'Product Added'.
- `distinct_id` (STRING): Unique user identifier.
- `timestamp` (TIMESTAMP): When the event happened.
- `properties` (JSON): Contains `product_id`, `category`, `price`, etc.

**Common Columns for `users`**:
- `distinct_id` (STRING): To join with events.
- `country` (STRING)
- `age` (INT)
- `gender` (STRING)

---

## Query 1: Funnel Conversion (View to Add to Cart) - Last 7 Days

**Objective**: Calculate how many people viewed a product and then added it to their cart within 7 days of the view, segmented by demographics.

```sql
WITH product_views AS (
    -- Step 1: Extract all product views
    SELECT 
        distinct_id,
        timestamp AS view_time,
        JSON_EXTRACT_SCALAR(properties, '$.product_id') AS product_id
    FROM `project.dataset.events`
    WHERE event_name = 'Product Viewed'
),
product_adds AS (
    -- Step 2: Extract all cart additions
    SELECT 
        distinct_id,
        timestamp AS add_time,
        JSON_EXTRACT_SCALAR(properties, '$.product_id') AS product_id
    FROM `project.dataset.events`
    WHERE event_name = 'Product Added'
),
joined_funnel AS (
    -- Step 3: Join views and adds efficiently using window-like logic or targeted joins
    SELECT 
        v.distinct_id,
        v.product_id,
        v.view_time,
        MIN(a.add_time) AS first_add_after_view
    FROM product_views v
    LEFT JOIN product_adds a ON v.distinct_id = a.distinct_id 
        AND v.product_id = a.product_id
        AND a.add_time BETWEEN v.view_time AND TIMESTAMP_ADD(v.view_time, INTERVAL 7 DAY)
    GROUP BY 1, 2, 3
)
-- Step 4: Aggregate by demographics
SELECT 
    u.country,
    u.gender,
    CASE 
        WHEN u.age < 18 THEN 'Under 18'
        WHEN u.age BETWEEN 18 AND 35 THEN '18-35'
        ELSE 'Over 35' 
    END AS age_group,
    COUNT(DISTINCT f.distinct_id) AS users_who_viewed,
    COUNT(DISTINCT CASE WHEN f.first_add_after_view IS NOT NULL THEN f.distinct_id END) AS users_who_added,
    SAFE_DIVIDE(
        COUNT(DISTINCT CASE WHEN f.first_add_after_view IS NOT NULL THEN f.distinct_id END),
        COUNT(DISTINCT f.distinct_id)
    ) * 100 AS conversion_rate
FROM joined_funnel f
JOIN `project.dataset.users` u ON f.distinct_id = u.distinct_id
GROUP BY 1, 2, 3
ORDER BY users_who_viewed DESC;
```

---

## Query 2: Cohort Retention (Repeat Purchase Rate)

**Objective**: Track users who made an initial purchase and see if they return for a second purchase within 30 days.

```sql
WITH first_purchases AS (
    -- Identify the very first purchase for every user
    SELECT 
        distinct_id,
        MIN(timestamp) AS first_purchase_time
    FROM `project.dataset.events`
    WHERE event_name = 'Order Completed'
    GROUP BY 1
),
subsequent_actions AS (
    -- Find any event after the first purchase
    SELECT 
        e.distinct_id,
        e.timestamp AS return_time,
        f.first_purchase_time
    FROM `project.dataset.events` e
    JOIN first_purchases f ON e.distinct_id = f.distinct_id
    WHERE e.timestamp > f.first_purchase_time
    AND e.timestamp <= TIMESTAMP_ADD(f.first_purchase_time, INTERVAL 30 DAY)
)
SELECT 
    FORMAT_TIMESTAMP('%Y-%m', f.first_purchase_time) AS cohort_month,
    u.country,
    COUNT(DISTINCT f.distinct_id) AS total_new_customers,
    COUNT(DISTINCT s.distinct_id) AS returning_customers,
    ROUND(COUNT(DISTINCT s.distinct_id) * 100.0 / COUNT(DISTINCT f.distinct_id), 2) AS retention_rate_30d
FROM first_purchases f
LEFT JOIN subsequent_actions s ON f.distinct_id = s.distinct_id
JOIN `project.dataset.users` u ON f.distinct_id = u.distinct_id
GROUP BY 1, 2
ORDER BY cohort_month DESC;
```

---

## Query 3: Product Affinity (Basket Analysis)

**Objective**: Find products that are frequently bought together ("Users also bought...").

```sql
WITH order_items AS (
    -- Extract items within the same order
    SELECT 
        JSON_EXTRACT_SCALAR(properties, '$.order_id') AS order_id,
        JSON_EXTRACT_SCALAR(properties, '$.product_name') AS product_name
    FROM `project.dataset.events`
    WHERE event_name = 'Order Completed'
)
SELECT 
    a.product_name AS product_a,
    b.product_name AS product_b,
    COUNT(*) AS frequency
FROM order_items a
JOIN order_items b ON a.order_id = b.order_id AND a.product_name < b.product_name
GROUP BY 1, 2
HAVING frequency > 5
ORDER BY frequency DESC
LIMIT 10;
```

---

## Industry Use Cases

1. **Conversion Funnels**: Used by Marketing to identify "leaks" in the bucket. If mobile users have a 50% lower conversion than desktop, the team investigates the mobile UX.
2. **Retention Analysis**: Critical for Subscription models. Predicting when a user might "churn" allows teams to send "We miss you" coupons before it's too late.
3. **Basket Analysis**: Powers recommendation engines. Amazon's "Customers who bought this also bought..." is essentially a real-time affinity query.

---

## Query 4: Attribution Rewards Logic (First vs. Last Touch)

**Objective**: Determine whether the "Discovery" (First View) or the "Closer" (Final Visit) gets the reward for a product addition.

```sql
WITH user_journey AS (
    -- Get all touchpoints for a user-product pair
    SELECT 
        distinct_id,
        timestamp,
        event_name,
        JSON_EXTRACT_SCALAR(properties, '$.product_id') AS product_id,
        JSON_EXTRACT_SCALAR(properties, '$.campaign_id') AS campaign_id,
        -- Generate sequence of events
        ROW_NUMBER() OVER(PARTITION BY distinct_id, JSON_EXTRACT_SCALAR(properties, '$.product_id') ORDER BY timestamp ASC) as first_touch_rank,
        ROW_NUMBER() OVER(PARTITION BY distinct_id, JSON_EXTRACT_SCALAR(properties, '$.product_id') ORDER BY timestamp DESC) as last_touch_rank
    FROM `project.dataset.events`
    WHERE event_name IN ('Product Viewed', 'Product Added')
),
conversions AS (
    -- Identify the actual "Added" event
    SELECT * FROM user_journey WHERE event_name = 'Product Added'
),
first_touch_rewards AS (
    -- Give credit to the VERY FIRST view of that product
    SELECT 
        c.distinct_id,
        c.product_id,
        j.campaign_id AS reward_recipient
    FROM conversions c
    JOIN user_journey j ON c.distinct_id = j.distinct_id AND c.product_id = j.product_id
    WHERE j.first_touch_rank = 1
),
last_touch_rewards AS (
    -- Give credit to the view IMMEDIATELY preceding the addition
    SELECT 
        c.distinct_id,
        c.product_id,
        j.campaign_id AS reward_recipient
    FROM conversions c
    JOIN user_journey j ON c.distinct_id = j.distinct_id AND c.product_id = j.product_id
    WHERE j.event_name = 'Product Viewed'
    AND j.timestamp < c.timestamp
    -- Get the view with the largest timestamp before the add (rank 1 in desc order filter)
    QUALIFY ROW_NUMBER() OVER(PARTITION BY c.distinct_id, c.product_id ORDER BY j.timestamp DESC) = 1
)
-- Final Comparison
SELECT 
    'First-Touch' as model, reward_recipient, COUNT(*) as conversion_count
FROM first_touch_rewards GROUP BY 1, 2
UNION ALL
SELECT 
    'Last-Touch' as model, reward_recipient, COUNT(*) as conversion_count
FROM last_touch_rewards GROUP BY 1, 2;
```

---

## The "Reward" Logic Explained (Attribution)

When a user views a product today but only adds it 5 days later, industry experts use **Attribution Models** to decide who "wins":

1.  **First-Touch**: Rewards the **Discovery**. If an Instagram Ad brought them to the site 5 days ago, Instagram gets the full reward. This is best for Brand Awareness.
2.  **Last-Touch**: Rewards the **Closer**. If they came back via a Direct Search today, the "Direct" channel gets the reward. This is best for evaluating conversion efficiency.
3.  **Linear**: Splits the reward. Both the Ad and the Direct visit get 50% credit.
4.  **Time Decay**: The closer the event is to the conversion, the more reward it gets.

### Which one should you use?
In e-commerce, **Last-Touch** is the default because it's easier to track, but **First-Touch** is what marketing teams use to prove that their expensive ads are actually working in the long run.
