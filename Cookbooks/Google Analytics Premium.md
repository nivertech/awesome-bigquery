## Marketing

#### Source / Medium

```sql
SELECT 
trafficSource.source + ' / ' + trafficSource.medium AS source_medium, count(*) as sessions
FROM [78667059.ga_sessions_20150103]
WHERE hits.hitNumber = 1
GROUP BY source_medium
ORDER BY sessions DESC
```

## Behaviour

#### Add Column To State If Page Is Last Hit

```sql
SELECT 
  unique_visit_id,
  page,
  hit_number,
  hit_type,
  max_hit,
  IF(hit_number = max_hit, 'yes', 'no') as last_page
FROM (SELECT CONCAT(fullVisitorId, STRING(visitId)) AS unique_visit_id, hits.hitNumber AS hit_number, hits.type AS hit_type, hits.page.pagePath AS page, MAX(hit_number) OVER (PARTITION BY unique_visit_id) AS max_hit
FROM [google.com:analytics-bigquery:LondonCycleHelmet.ga_sessions_20130910]
WHERE hits.type = 'PAGE'
GROUP BY unique_visit_id, hit_number, hit_type, page
ORDER BY unique_visit_id, hit_number)
```

#### Pageviews, Exits and Exit Rate

```sql
SELECT 
  page,
  COUNT(page) as pageviews,
  SUM(IF(hit_number = max_hit, 1, 0)) as exits,
  (SUM(IF(hit_number = max_hit, 1, 0))/COUNT(page)) * 100 AS exit_rate
FROM (SELECT CONCAT(fullVisitorId, STRING(visitId)) AS unique_visit_id, hits.hitNumber AS hit_number, hits.type AS hit_type, hits.page.pagePath AS page, MAX(hit_number) OVER (PARTITION BY unique_visit_id) AS max_hit
FROM [google.com:analytics-bigquery:LondonCycleHelmet.ga_sessions_20130910]
WHERE hits.type = 'PAGE'
GROUP BY unique_visit_id, hit_number, hit_type, page
ORDER BY unique_visit_id, hit_number)
GROUP BY page
ORDER BY pageviews DESC
```

## GA + CRM data

#### Join Traffic Source, Landing Page and Customer Type using Order ID

One of the issues with the BigQuery export is that you do not get the landing page as a traffic source so you need to do a JOIN using fullVisitorId and VisitId. This is time consuming but is the only way to get landing page next to transaction ID.
You can then join CRM data based on the transaction ID.

```sql
SELECT c.mdm, c.landpg, c.trid, d.custtype
FROM 
  (SELECT 
  a.user_session_id as uid,
  a.hit as hit,
  a.transaction_id as trid,
  b.medium as mdm,
  b.hit_number as hitnum,
  b.landing_page as landpg
FROM 
  (SELECT STRING(fullVisitorId) + '-' + STRING(visitId) as user_session_id, hits.hitNumber as hit, hits.transaction.transactionId AS transaction_id 
  FROM FLATTEN([6191731.ga_sessions_20141110], hits.transaction.transactionId)) AS a
JOIN EACH
  (SELECT STRING(fullVisitorId) + '-' + STRING(visitId) as user_session_id, trafficSource.medium AS medium, hits.page.pagePath AS landing_page, hits.hitNumber AS hit_number 
  FROM FLATTEN([6191731.ga_sessions_20141110],hits.hitNumber) WHERE hits.hitNumber = 1)  AS b
ON a.user_session_id = b.user_session_id ) as c
JOIN EACH 
  (SELECT OrderID as ordid, customerType as custtype 
  FROM [Customer_Type.Data] ) AS d
ON c.trid = d.ordid
```


## Classic Ecommerce

#### Top 5 Products Launched In Certain Month

todo- can this be done without the join using WINDOW function? 
Also do we need date and just have the MIN date and build queries on that? << this will work if we are just looking at latest month not pevious month but you can limite the daterange to end of that month

```sql
SELECT a.date_month, b.first_month, a.product_name, a.qty
FROM (
  SELECT MONTH(date) as date_month, hits.item.productName as product_name, COUNT(hits.item.productName) as qty
  FROM (TABLE_DATE_RANGE([6191731.ga_sessions_], TIMESTAMP('2014-04-01'), TIMESTAMP('2014-09-30'))) 
  GROUP BY product_name, date_month ) as a
JOIN (
  SELECT MIN(MONTH(date)) as first_month, hits.item.productName as product_name
  FROM (TABLE_DATE_RANGE([6191731.ga_sessions_], TIMESTAMP('2014-04-01'), TIMESTAMP('2014-09-30'))) 
  GROUP BY product_name ) as b
ON a.product_name = b.product_name
WHERE b.first_month = 9
ORDER BY a.qty DESC
LIMIT 5
```

#### All Full Visitor IDs that purchased one of top 10 products

```sql
SELECT fullVisitorId, hits.item.productName
  FROM (FLATTEN([6191731.ga_sessions_20140601], hits.item))
  WHERE hits.item.productName IN (
    SELECT hits.item.productName FROM (
    SELECT hits.item.productName, sum( hits.item.itemRevenue ) as total
    FROM (FLATTEN([6191731.ga_sessions_20140601], hits.item))
    WHERE hits.item.productName IS NOT NULL
    GROUP BY hits.item.productName
    order by total DESC
    LIMIT 10
  ))
   AND totals.transactions>=1
  GROUP BY fullVisitorId, hits.item.productName
```

#### Users that purchased product A also purchased

```sql
SELECT hits.item.productName AS other_purchased_products, COUNT(hits.item.productName) AS quantity
FROM [6191731.ga_sessions_20140601]
WHERE fullVisitorId IN (
  SELECT fullVisitorId
  FROM [6191731.ga_sessions_20140601]
  WHERE hits.item.productName = 'PRODUCT A'
   AND totals.transactions>=1
  GROUP BY fullVisitorId )
 AND hits.item.productName IS NOT NULL
 AND hits.item.productName !='PRODUCT A'
GROUP BY other_purchased_products
ORDER BY quantity DESC;
```


#### What is the average number of user interactions before a purchase?

```sql
SELECT one.hits.item.productName AS product_name, product_revenue, ( sum_of_hit_number / total_hits ) AS avg_hit_number
FROM (
  SELECT hits.item.productName, SUM(hits.hitNumber) AS sum_of_hit_number, SUM( hits.item.itemRevenue ) AS product_revenue
  FROM (TABLE_DATE_RANGE([6191731.ga_sessions_], TIMESTAMP('2014-06-01'), TIMESTAMP('2014-06-14'))) 
  WHERE hits.item.productName IS NOT NULL
    AND totals.transactions>=1
  GROUP BY hits.item.productName
  ORDER BY sum_of_hit_number DESC ) as one
JOIN (
  SELECT hits.item.productName, COUNT( fullVisitorId ) AS total_hits
  FROM (TABLE_DATE_RANGE([6191731.ga_sessions_], TIMESTAMP('2014-06-01'), TIMESTAMP('2014-06-14'))) 
  WHERE hits.item.productName IS NOT NULL
    AND totals.transactions>=1
  GROUP BY hits.item.productName ) as two
ON one.hits.item.productName = two.hits.item.productName
ORDER BY product_revenue DESC
```

## Enhanced Ecommerce Notes

The hits.type values 'ITEM' and 'TRANSACTION' are no longer used as the new tagging must send the data with the hits.type 'PAGE' or 'EVENT'.

There is a new way to identify the ecommerce flow

Ecommerce Tag Action | BigQuery Column Name | BigQuery Value
---------------------|----------------------|-----------------
Checkout             |hits.eCommerceAction.action_type | 5
Purchase             |hits.eCommerceAction.action_type | 6


#### Enhanced Ecommerce - Products - Unique Purchases as Rows

```sql
SELECT 
  hits.product.v2ProductName as Product_Name,
  hits.page.pagePath
FROM [6191731.ga_sessions_20141101]
WHERE hits.product.v2ProductName IS NOT NULL 
  AND hits.type = 'PAGE' 
  AND hits_eCommerceAction_action_type = '6'
  AND hits.page.pagePath CONTAINS 'confirmation'
```

#### Enhanced Ecommerce - Products - Revenue, Uniq & Qty

```sql
SELECT 
  hits.product.v2ProductName as Product_Name,
  (sum( hits.product.productRevenue ) * 0.000001) as Product_Revenue,
  count(hits.product.v2ProductName) as Unique_Purchases,
  sum( hits.product.productQuantity ) as Quantity
FROM [6191731.ga_sessions_20141101]
WHERE hits.product.v2ProductName IS NOT NULL 
  AND hits.eCommerceAction.action_type = '6'
GROUP BY Product_Name
ORDER BY Product_Revenue DESC
```
#### Enhanced Ecommerce - Product with Rank In Category

```sql
SELECT
   hits.product.v2ProductCategory as Product_Category,
   hits.product.v2ProductName as Product_Name,
   (sum( hits.product.productRevenue ) * 0.000001) as Product_Revenue,
   RANK() OVER (PARTITION BY Product_Category ORDER BY Product_Revenue DESC) Category_Rank,
FROM
   [6191731.ga_sessions_20141101]
WHERE hits.product.v2ProductName IS NOT NULL 
  AND hits.eCommerceAction.action_type = '6'
GROUP BY Product_Category, Product_Name
ORDER BY Product_Revenue DESC
```

#### Enhanced Ecommerce - Top 5 Products Per Category

```sql
SELECT Product_Category, Product_Name, Product_Revenue, Category_Rank FROM (
  SELECT
     hits.product.v2ProductCategory as Product_Category,
     hits.product.v2ProductName as Product_Name,
     (sum( hits.product.productRevenue ) * 0.000001) as Product_Revenue,
     RANK() OVER (PARTITION BY Product_Category ORDER BY Product_Revenue DESC) Category_Rank,
  FROM
     [6191731.ga_sessions_20141101]
  WHERE hits.product.v2ProductName IS NOT NULL 
    AND hits.eCommerceAction.action_type = '6'
  GROUP BY Product_Category, Product_Name
  ORDER BY Product_Revenue DESC
)
WHERE Category_Rank < 6
ORDER BY Product_Category, Category_Rank ASC

```

#### Enhanced Ecommerce - Top 5 Products Per Category with Overall Rank

Todo - same as 'Enhanced Ecommerce - Top 5 Products Per Category' but join with overall rank.

#### Enhanced Ecommerce - Products Bring In 80% Of Revenue

Todo - change example below to work with GA data

```sql
SELECT 
  word, 
  ratio_running,
  IF(ratio_running < 0.8, 'TRUE', 'FALSE') as top_80_percent
FROM (
SELECT 
  word, 
  word_count, 
  word_count_running, 
  ratio, 
  SUM(ratio) OVER(ORDER BY ratio DESC, word) AS ratio_running, 
  (word_count / ratio) as total
FROM (
  SELECT 
     word, 
     word_count, 
     SUM(word_count) OVER(ORDER BY word_count DESC, word) AS word_count_running, 
     RATIO_TO_REPORT(word_count) OVER() AS ratio
  FROM [publicdata:samples.shakespeare]
  WHERE corpus  = 'hamlet'
  AND word > 'a' LIMIT 30
  ))
```

#### Enhanced Ecommerce

To-do - Top New Products (Not Available Previous Month)

```sql
SELECT a.date_month, b.first_month, a.product_name, a.qty
FROM (
  SELECT MONTH(date) as date_month, hits.item.productName as product_name, COUNT(hits.item.productName) as qty
  FROM (TABLE_DATE_RANGE([6191731.ga_sessions_], TIMESTAMP('2014-04-01'), TIMESTAMP('2014-09-30'))) 
  GROUP BY product_name, date_month ) as a
JOIN (
  SELECT MIN(MONTH(date)) as first_month, hits.item.productName as product_name
  FROM (TABLE_DATE_RANGE([6191731.ga_sessions_], TIMESTAMP('2014-04-01'), TIMESTAMP('2014-09-30'))) 
  GROUP BY product_name ) as b
ON a.product_name = b.product_name
WHERE b.first_month = 9
ORDER BY a.qty DESC
LIMIT 5;
```
