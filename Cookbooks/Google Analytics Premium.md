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
