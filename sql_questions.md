# NYC Airbnb — SQL Practice Queries

10 SQL questions written against the cleaned `airbnbny` table in PostgreSQL, following the same
dataset explored in the Python EDA notebook. Ordered roughly by increasing complexity.

---

### 1. How many listings does each borough have?

```sql
SELECT neighbourhood_group, COUNT(*) AS total_listings
FROM airbnbny
GROUP BY neighbourhood_group
ORDER BY total_listings DESC;
```

### 2. What is the median price per borough?

Postgres has no built-in `MEDIAN()` function — `PERCENTILE_CONT` is the standard way to get one.

```sql
SELECT
    neighbourhood_group,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY price) AS median_price
FROM airbnbny
GROUP BY neighbourhood_group
ORDER BY median_price DESC;
```

### 3. What are the 10 most expensive listings, and what room type are they?

```sql
SELECT name, neighbourhood_group, room_type, price
FROM airbnbny
ORDER BY price DESC
LIMIT 10;
```

### 4. Does license status relate to the 30-night minimum-stay pattern?

Tests the Local Law 18 hypothesis: unlicensed/exempt listings should cluster more heavily at a
30-night minimum than licensed ones.

```sql
SELECT
    license_status,
    COUNT(*) AS total_listings,
    ROUND(100.0 * SUM(CASE WHEN is_30_night_min THEN 1 ELSE 0 END) / COUNT(*), 1) AS pct_30_night_min
FROM airbnbny
GROUP BY license_status
ORDER BY pct_30_night_min DESC;
```

### 5. What share of listings are fully blocked (0 days) vs. fully open (365 days) all year?

```sql
SELECT
    ROUND(100.0 * SUM(CASE WHEN availability_365 = 0 THEN 1 ELSE 0 END) / COUNT(*), 1) AS pct_fully_blocked,
    ROUND(100.0 * SUM(CASE WHEN availability_365 = 365 THEN 1 ELSE 0 END) / COUNT(*), 1) AS pct_fully_open
FROM airbnbny;
```

### 6. Which hosts have the most listings, and what's their average price?

```sql
SELECT
    host_id,
    host_name,
    COUNT(*) AS listing_count,
    ROUND(AVG(price)::numeric, 2) AS avg_price
FROM airbnbny
GROUP BY host_id, host_name
ORDER BY listing_count DESC
LIMIT 10;
```

### 7. How does average price-per-bed compare across boroughs and room types?

```sql
SELECT
    neighbourhood_group,
    room_type,
    ROUND(AVG(price_per_bed)::numeric, 2) AS avg_price_per_bed,
    COUNT(*) AS listing_count
FROM airbnbny
WHERE price_flag = FALSE   -- exclude flagged extreme/placeholder prices
GROUP BY neighbourhood_group, room_type
ORDER BY neighbourhood_group, avg_price_per_bed DESC;
```

### 8. Within each borough, how does each listing rank by price? (window function)

```sql
SELECT
    name,
    neighbourhood_group,
    price,
    RANK() OVER (PARTITION BY neighbourhood_group ORDER BY price DESC) AS price_rank
FROM airbnbny
WHERE price_flag = FALSE
ORDER BY neighbourhood_group, price_rank
LIMIT 30;
```

### 9. Which neighbourhoods have an average price above their own borough's average? (CTE)

```sql
WITH borough_avg AS (
    SELECT neighbourhood_group, AVG(price) AS borough_avg_price
    FROM airbnbny
    WHERE price_flag = FALSE
    GROUP BY neighbourhood_group
),
neighbourhood_avg AS (
    SELECT neighbourhood_group, neighbourhood, AVG(price) AS neighbourhood_avg_price
    FROM airbnbny
    WHERE price_flag = FALSE
    GROUP BY neighbourhood_group, neighbourhood
)
SELECT
    n.neighbourhood_group,
    n.neighbourhood,
    ROUND(n.neighbourhood_avg_price::numeric, 2) AS neighbourhood_avg_price,
    ROUND(b.borough_avg_price::numeric, 2) AS borough_avg_price
FROM neighbourhood_avg n
JOIN borough_avg b ON n.neighbourhood_group = b.neighbourhood_group
WHERE n.neighbourhood_avg_price > b.borough_avg_price
ORDER BY n.neighbourhood_group, neighbourhood_avg_price DESC;
```

### 10. What does the review-activity distribution look like — how many listings have never been reviewed, vs. reviewed frequently?

```sql
SELECT
    CASE
        WHEN number_of_reviews = 0 THEN 'No reviews'
        WHEN reviews_per_month < 1 THEN 'Low activity (<1/month)'
        WHEN reviews_per_month < 3 THEN 'Medium activity (1-3/month)'
        ELSE 'High activity (3+/month)'
    END AS review_activity_tier,
    COUNT(*) AS listing_count,
    ROUND(AVG(price)::numeric, 2) AS avg_price
FROM airbnbny
GROUP BY review_activity_tier
ORDER BY listing_count DESC;
```

---

**Note on `price_flag`:** several queries exclude rows where `price_flag = TRUE` — these mark listings
priced at suspicious round-number extremes (e.g. $10,000+), identified during cleaning as likely data
entry errors rather than genuine luxury listings. See the Python notebook for the full reasoning.
