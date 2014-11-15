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


