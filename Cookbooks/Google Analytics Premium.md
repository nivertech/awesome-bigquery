
#### Enhanced Ecommerce - Unique Purchases as Rows

```sql
SELECT 
  hits.product.v2ProductName as Product_Name,
  hits.page.pagePath
FROM [6191731.ga_sessions_20141101]
WHERE hits.product.v2ProductName IS NOT NULL 
  AND hits.type = 'PAGE' 
  AND hits.page.pagePath CONTAINS 'confirmation'
```
