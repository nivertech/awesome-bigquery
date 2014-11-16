## Classic Ecommerce

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
SELECT hits.item.productName AS prod_name, 
count(*) AS transactions
FROM (TABLE_DATE_RANGE([6191731.ga_sessions_], 
	TIMESTAMP('2014-06-01'), 
                       TIMESTAMP('2014-06-14'))) 
WHERE hits.item.productName IS NOT NULL
GROUP BY prod_name ORDER BY transactions DESC;
```
