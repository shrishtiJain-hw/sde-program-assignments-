# MODULE-8 SQL Assignment 1

## 1. New Customers Acquired in June 2023

**Business Problem:** The marketing team ran a campaign in June 2023 and wants to see how many new customers signed up during that period.

**Fields to Retrieve:**
- `PARTY_ID`
- `FIRST_NAME`
- `LAST_NAME`
- `EMAIL`
- `PHONE`
- `ENTRY_DATE`

```sql
select
p.PARTY_ID,
p.FIRST_NAME,
p.LAST_NAME,
max(case when cm.CONTACT_MECH_TYPE_ID = 'EMAIL_ADDRESS' then cm.INFO_STRING end) as EMAIL,
max(tn.CONTACT_NUMBER) as PHONE,
p.CREATED_STAMP as ENTRY_DATE
from person p
join party_role pr on p.PARTY_ID = pr.PARTY_ID
and pr.ROLE_TYPE_ID = 'CUSTOMER'
left join party_contact_mech pcm on p.PARTY_ID = pcm.PARTY_ID
left join contact_mech cm on pcm.CONTACT_MECH_ID = cm.CONTACT_MECH_ID
left join telecom_number tn on pcm.CONTACT_MECH_ID = tn.CONTACT_MECH_ID
where p.CREATED_STAMP >= '2023-06-01'
and p.CREATED_STAMP < '2023-07-01'
group by
p.PARTY_ID,
p.FIRST_NAME,
p.LAST_NAME,
p.CREATED_STAMP;


```

---

## 2. List All Active Physical Products

**Business Problem:** Merchandising teams often need a list of all physical products to manage logistics, warehousing, and shipping.

**Fields to Retrieve:**
- `PRODUCT_ID`
- `PRODUCT_TYPE_ID`
- `INTERNAL_NAME`

```sql
select
p.PRODUCT_ID,
p.PRODUCT_TYPE_ID,
p.INTERNAL_NAME
from product p
join product_type pt on p.PRODUCT_TYPE_ID = pt.PRODUCT_TYPE_ID
where pt.IS_PHYSICAL = 'Y'
and p.SALES_DISCONTINUATION_DATE is null;
```

---

## 3. Products Missing NetSuite ID

**Business Problem:** A product cannot sync to NetSuite unless it has a valid NetSuite ID. The OMS needs a list of all products that still need to be created or updated in NetSuite.

**Fields to Retrieve:**
- `PRODUCT_ID`
- `PRODUCT_TYPE_ID`
- `INTERNAL_NAME`
- `NETSUITE_ID`

```sql
select
p.PRODUCT_ID,
p.PRODUCT_TYPE_ID,
p.INTERNAL_NAME,
gi.ID_VALUE as NETSUITE_ID
from product p
left join good_identification gi on p.PRODUCT_ID = gi.PRODUCT_ID
and gi.GOOD_IDENTIFICATION_TYPE_ID = 'ERP_ID'
where gi.ID_VALUE is null
or gi.ID_VALUE = '';
```

---

## 4. Product IDs Across Systems

**Business Problem:** To sync an order or product across multiple systems (e.g., Shopify, HotWax, ERP/NetSuite), the OMS needs to know each system’s unique identifier for that product. This query retrieves the Shopify ID, HotWax ID, and ERP ID (NetSuite ID) for all products.

**Fields to Retrieve:**
- `PRODUCT_ID`
- `SHOPIFY_ID`
- `HOTWAX_ID`
- `ERP_ID`

```sql
select
p.PRODUCT_ID,
max(case
when gi.GOOD_IDENTIFICATION_TYPE_ID = 'SHOPIFY_PROD_ID'
then gi.ID_VALUE
end) as SHOPIFY_ID,
p.PRODUCT_ID as HOTWAX_ID,
max(case
when gi.GOOD_IDENTIFICATION_TYPE_ID = 'ERP_ID'
then gi.ID_VALUE
end) as ERP_ID
from product p
left join good_identification gi on p.PRODUCT_ID = gi.PRODUCT_ID
group by p.PRODUCT_ID;
```

---

## 5. Completed Orders in August 2023

**Business Problem:** After running similar reports for a previous month, you now need all completed orders in August 2023 for analysis.

**Fields to Retrieve:**
- `PRODUCT_ID`
- `PRODUCT_TYPE_ID`
- `PRODUCT_STORE_ID`
- `TOTAL_QUANTITY`
- `INTERNAL_NAME`
- `FACILITY_ID`
- `EXTERNAL_ID`
- `FACILITY_TYPE_ID`
- `ORDER_HISTORY`
- `ORDER_ID`
- `ORDER_ITEM_SEQ_ID`
- `SHIP_GROUP_SEQ_ID`

```sql
SELECT
    oi.PRODUCT_ID,
    p.PRODUCT_TYPE_ID,
    oh.PRODUCT_STORE_ID,
    oi.QUANTITY AS TOTAL_QUANTITY,
    p.INTERNAL_NAME,
    oisg.FACILITY_ID,
    oh.EXTERNAL_ID,
    f.FACILITY_TYPE_ID,
    os.ORDER_STATUS_ID AS ORDER_HISTORY_ID,
    oh.ORDER_ID,
    oi.ORDER_ITEM_SEQ_ID,
    oisg.SHIP_GROUP_SEQ_ID
FROM order_header oh
JOIN order_item oi ON oh.ORDER_ID = oi.ORDER_ID
JOIN order_item_ship_group oisg ON oh.ORDER_ID = oisg.ORDER_ID
    AND oi.SHIP_GROUP_SEQ_ID = oisg.SHIP_GROUP_SEQ_ID
JOIN product p ON oi.PRODUCT_ID = p.PRODUCT_ID
LEFT JOIN facility f ON oisg.FACILITY_ID = f.FACILITY_ID
JOIN order_status os ON oh.ORDER_ID = os.ORDER_ID
    AND os.STATUS_ID = 'ORDER_COMPLETED'
WHERE oh.ORDER_TYPE_ID = 'SALES_ORDER'
  AND os.STATUS_DATETIME >= '2023-08-01 00:00:00'
  AND os.STATUS_DATETIME < '2023-09-01 00:00:00';
```

---

## 7. Newly Created Sales Orders and Payment Methods

**Business Problem:** Finance teams need to see new orders and their payment methods for reconciliation and fraud checks.

**Fields to Retrieve:**
- `ORDER_ID`
- `TOTAL_AMOUNT`
- `PAYMENT_METHOD_TYPE_ID`
- `SHOPIFY_ORDER_ID`

```sql
SELECT
    oh.order_id AS ORDER_ID,
    oh.grand_total AS TOTAL_AMOUNT,
    opp.payment_method_type_id AS PAYMENT_METHOD,
    oh.external_id AS SHOPIFY_ORDER_ID
FROM order_header oh
LEFT JOIN order_payment_preference opp ON oh.order_id = opp.order_id
WHERE oh.order_type_id = 'SALES_ORDER'
ORDER BY oh.entry_date DESC;
```

---

## 8. Payment Captured but Not Shipped

**Business Problem:** Finance teams want to ensure revenue is recognized properly. If payment is captured but no shipment has occurred, it warrants further review.

**Fields to Retrieve:**
- `ORDER_ID`
- `ORDER_STATUS`
- `PAYMENT_STATUS`
- `SHIPMENT_STATUS`

```sql
SELECT 
    oh.order_id, 
    oh.status_id AS order_status, 
    opp.status_id AS payment_status, 
    COALESCE(s.status_id, 'NOT_SHIPPED') AS shipping_status 
FROM order_header oh
JOIN order_payment_preference opp ON oh.order_id = opp.order_id
LEFT JOIN shipment s ON oh.order_id = s.primary_order_id
WHERE opp.status_id IN ('PAYMENT_SETTLED', 'PAYMENT_RECEIVED')
  AND (s.status_id IS NULL 
       OR s.status_id NOT IN ('SHIPMENT_SHIPPED', 'SHIPMENT_CANCELLED'));
```

---

## 9. Orders Completed Hourly

**Business Problem:** Operations teams may want to see how orders complete across the day to schedule staffing.

**Fields to Retrieve:**
- `TOTAL_ORDERS`
- `HOUR`

```sql
select
count(distinct oh.ORDER_ID) as TOTAL_ORDERS,
hour(os.STATUS_DATETIME) as HOUR
from order_header oh
join order_status os on oh.ORDER_ID = os.ORDER_ID
where oh.STATUS_ID = 'ORDER_COMPLETED'
and os.STATUS_ID = 'ORDER_COMPLETED'
group by hour(os.STATUS_DATETIME)
order by HOUR asc;
```

---

## 10. BOPIS Orders Revenue (Last Year)

**Business Problem:** BOPIS (Buy Online, Pickup In Store) is a key retail strategy. Finance wants to know the revenue from BOPIS orders for the previous year.

**Fields to Retrieve:**
- `TOTAL_RECORDS`
- `TOTALS_REVENUE`

```sql
select
count(oh.ORDER_ID) as TOTAL_RECORDS,
sum(oh.GRAND_TOTAL) as TOTALS_REVENUE
from order_header oh
where oh.STATUS_ID = 'ORDER_COMPLETED'
and year(oh.ORDER_DATE) = year(curdate()) - 1
and exists (
select 1
from order_item_ship_group oisg
where oisg.ORDER_ID = oh.ORDER_ID
and oisg.SHIPMENT_METHOD_TYPE_ID = 'STOREPICKUP'
);
```

---

## 11. Canceled Orders (Last Month)

**Business Problem:** The merchandising team needs to know how many orders were canceled in the previous month and their reasons.

**Fields to Retrieve:**
- `TOTAL_ORDERS`
- `CANCELLATION_REASON`

```sql
select
count(distinct oh.ORDER_ID) as TOTAL_ORDERS,
os.CHANGE_REASON as CANCELLATION_REASON
from order_header oh
join order_status os on oh.ORDER_ID = os.ORDER_ID
where oh.STATUS_ID = 'ORDER_CANCELLED'
and os.STATUS_ID = 'ORDER_CANCELLED'
and month(os.STATUS_DATETIME) = month(current_date - interval 1 month)
and year(os.STATUS_DATETIME) = year(current_date - interval 1 month)
group by os.CHANGE_REASON
order by TOTAL_ORDERS desc;
```

---

## 12. Product Threshold Value

**Business Problem:** The retailer has set a threshold value for products that are sold online, in order to avoid overselling.

**Fields to Retrieve:**
- `PRODUCT_ID`
- `THRESHOLD`

```sql
SELECT pf.product_id,
       pf.minimum_stock as threshold
FROM product_facility pf
LEFT JOIN facility f
ON pf.facility_id = f.facility_id
WHERE facility_type_id = 'CONFIGURATION';
```
