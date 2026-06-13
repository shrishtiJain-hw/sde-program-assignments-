# MODULE-8 SQL Assignment 2

## 5.1 Shipping Addresses for October 2023 Orders

**Business Problem:** Customer Service might need to verify addresses for orders placed or completed in October 2023. This helps ensure shipments are delivered correctly and prevents address-related issues.

**Fields to Retrieve:**
- `ORDER_ID`
- `PARTY_ID`
- `CUSTOMER_NAME`
- `STREET_ADDRESS`
- `CITY`
- `STATE_PROVINCE`
- `POSTAL_CODE`
- `COUNTRY_CODE`
- `ORDER_STATUS`
- `ORDER_DATE`

```sql
select
oh.ORDER_ID,
orl.PARTY_ID,
concat(p.FIRST_NAME, ' ', p.LAST_NAME) as CUSTOMER_NAME,
pa.ADDRESS1 as STREET_ADDRESS,
pa.CITY,
pa.STATE_PROVINCE_GEO_ID as STATE_PROVINCE,
pa.POSTAL_CODE,
pa.COUNTRY_GEO_ID as COUNTRY_CODE,
oh.STATUS_ID as ORDER_STATUS,
oh.ORDER_DATE
from order_header oh
join order_role orl on oh.ORDER_ID = orl.ORDER_ID
and orl.ROLE_TYPE_ID = 'PLACING_CUSTOMER'
join person p on orl.PARTY_ID = p.PARTY_ID
join order_contact_mech ocm on oh.ORDER_ID = ocm.ORDER_ID
and ocm.CONTACT_MECH_PURPOSE_TYPE_ID = 'SHIPPING_LOCATION'
join postal_address pa on ocm.CONTACT_MECH_ID = pa.CONTACT_MECH_ID
where oh.ORDER_DATE >= '2023-10-01'
and oh.ORDER_DATE < '2023-11-01';
```

---

## 5.2 Orders From New York

**Business Problem:** Companies often want region-specific analysis to plan local marketing, staffing, or promotions in certain areas—here, specifically, New York.

**Fields to Retrieve:**
- `ORDER_ID`
- `CUSTOMER_NAME`
- `STREET_ADDRESS`
- `CITY`
- `STATE_PROVINCE`
- `POSTAL_CODE`
- `TOTAL_AMOUNT`
- `ORDER_DATE`
- `ORDER_STATUS`

```sql
select
oh.ORDER_ID,
concat(p.FIRST_NAME, ' ', p.LAST_NAME) as CUSTOMER_NAME,
pa.ADDRESS1 as STREET_ADDRESS,
pa.CITY,
pa.STATE_PROVINCE_GEO_ID as STATE_PROVINCE,
pa.POSTAL_CODE,
oh.GRAND_TOTAL as TOTAL_AMOUNT,
oh.ORDER_DATE,
oh.STATUS_ID as ORDER_STATUS
from order_header oh
join order_role orl on oh.ORDER_ID = orl.ORDER_ID
and orl.ROLE_TYPE_ID = 'PLACING_CUSTOMER'
join person p on orl.PARTY_ID = p.PARTY_ID
join order_contact_mech ocm on oh.ORDER_ID = ocm.ORDER_ID
and ocm.CONTACT_MECH_PURPOSE_TYPE_ID = 'SHIPPING_LOCATION'
join postal_address pa on ocm.CONTACT_MECH_ID = pa.CONTACT_MECH_ID
where pa.STATE_PROVINCE_GEO_ID = 'NY';
```

---

## 5.3 Top-Selling Product in New York

**Business Problem:** Merchandising teams need to identify the best-selling product(s) in a specific region (New York) for targeted restocking or promotions.

**Fields to Retrieve:**
- `PRODUCT_ID`
- `INTERNAL_NAME`
- `TOTAL_QUANTITY_SOLD`
- `REVENUE`

```sql
SELECT
    oi.PRODUCT_ID,
    p.INTERNAL_NAME,
    SUM(oi.QUANTITY) AS TOTAL_QUANTITY_SOLD,
    SUM(oi.QUANTITY * oi.UNIT_PRICE) AS REVENUE
FROM order_header oh
JOIN order_item oi ON oh.ORDER_ID = oi.ORDER_ID
JOIN product p ON oi.PRODUCT_ID = p.PRODUCT_ID
JOIN order_contact_mech ocm ON oh.ORDER_ID = ocm.ORDER_ID 
    AND ocm.CONTACT_MECH_PURPOSE_TYPE_ID = 'SHIPPING_LOCATION'
JOIN postal_address pa ON ocm.CONTACT_MECH_ID = pa.CONTACT_MECH_ID
WHERE pa.STATE_PROVINCE_GEO_ID = 'NY'
  AND oh.STATUS_ID = 'ORDER_COMPLETED' 
GROUP BY
    oi.PRODUCT_ID,
    p.INTERNAL_NAME
ORDER BY 
    TOTAL_QUANTITY_SOLD DESC, 
    REVENUE DESC
LIMIT 1;
```

---

## 7.3 Store-Specific (Facility-Wise) Revenue

**Business Problem:** Different physical or online stores (facilities) may have varying levels of performance. The business wants to compare revenue across facilities for sales planning and budgeting.

**Fields to Retrieve:**
- `FACILITY_ID`
- `FACILITY_NAME`
- `TOTAL_ORDERS`
- `TOTAL_REVENUE`
- `DATE_RANGE`

```sql
select
f.FACILITY_ID,
f.FACILITY_NAME,
count(distinct oh.ORDER_ID) as TOTAL_ORDERS,
sum(oi.QUANTITY * oi.UNIT_PRICE) as TOTAL_REVENUE,
concat(
min(date(oh.ORDER_DATE)),
' to ',
max(date(oh.ORDER_DATE))
) as DATE_RANGE
from order_item oi
join order_item_ship_group oisg on oi.ORDER_ID = oisg.ORDER_ID
and oi.SHIP_GROUP_SEQ_ID = oisg.SHIP_GROUP_SEQ_ID
join facility f on oisg.FACILITY_ID = f.FACILITY_ID
join order_header oh on oi.ORDER_ID = oh.ORDER_ID
group by
f.FACILITY_ID,
f.FACILITY_NAME
order by TOTAL_REVENUE desc;
```

---

## 8.1 Lost and Damaged Inventory

**Business Problem:** Warehouse managers need to track shrinkage such as lost, damaged, or stolen inventory to reconcile physical versus system counts.

**Fields to Retrieve:**
- `INVENTORY_ITEM_ID`
- `PRODUCT_ID`
- `FACILITY_ID`
- `QUANTITY_LOST_OR_DAMAGED`
- `REASON_CODE`
- `TRANSACTION_DATE`

```sql
select
ii.INVENTORY_ITEM_ID,
ii.PRODUCT_ID,
ii.FACILITY_ID,
abs(iid.QUANTITY_ON_HAND_DIFF) as QUANTITY_LOST_OR_DAMAGED,
iid.REASON_ENUM_ID as REASON_CODE,
iid.EFFECTIVE_DATE as TRANSACTION_DATE
from inventory_item_detail iid
join inventory_item ii on iid.INVENTORY_ITEM_ID = ii.INVENTORY_ITEM_ID
where iid.REASON_ENUM_ID in (
'VAR_LOST',
'VAR_DAMAGED',
'VAR_STOLEN',
'REJ_RSN_DAMAGED',
'WORN_DISPLAY'
);
```

---

## 8.2 Low Stock or Out of Stock Items Report

**Business Problem:** Avoiding out-of-stock situations is critical. This report flags items that have fallen below a reorder threshold or have zero available stock.

**Fields to Retrieve:**
- `PRODUCT_ID`
- `PRODUCT_NAME`
- `FACILITY_ID`
- `QOH`
- `ATP`
- `REORDER_THRESHOLD`
- `DATE_CHECKED`

```sql
select
ii.PRODUCT_ID,
p.INTERNAL_NAME as PRODUCT_NAME,
ii.FACILITY_ID,
ii.QUANTITY_ON_HAND as QOH,
ii.AVAILABLE_TO_PROMISE as ATP,
pf.REORDER_POINT as REORDER_THRESHOLD,
now() as DATE_CHECKED
from inventory_item ii
join product p on ii.PRODUCT_ID = p.PRODUCT_ID
left join product_facility pf on ii.PRODUCT_ID = pf.PRODUCT_ID
and ii.FACILITY_ID = pf.FACILITY_ID
where ii.QUANTITY_ON_HAND <= ifnull(pf.REORDER_POINT, 0)
or ii.QUANTITY_ON_HAND = 0;
```

---

## 8.3 Retrieve the Current Facility of Open Orders

**Business Problem:** The business wants to know where open orders are currently assigned, whether in a physical store or a virtual fulfillment location.

**Fields to Retrieve:**
- `ORDER_ID`
- `ORDER_STATUS`
- `FACILITY_ID`
- `FACILITY_NAME`
- `FACILITY_TYPE_ID`

```sql
select distinct
oh.ORDER_ID,
oh.STATUS_ID as ORDER_STATUS,
f.FACILITY_ID,
f.FACILITY_NAME,
f.FACILITY_TYPE_ID
from order_header oh
join order_item oi on oh.ORDER_ID = oi.ORDER_ID
join order_item_ship_group oisg on oi.ORDER_ID = oisg.ORDER_ID
and oi.SHIP_GROUP_SEQ_ID = oisg.SHIP_GROUP_SEQ_ID
join facility f on oisg.FACILITY_ID = f.FACILITY_ID
where oh.STATUS_ID in (
'ORDER_CREATED',
'ORDER_APPROVED'
);
```

---

## 8.4 Items Where QOH and ATP Differ

**Business Problem:** Sometimes Quantity on Hand (QOH) doesn’t match Available to Promise (ATP) due to reservations, pending orders, or inventory discrepancies.

**Fields to Retrieve:**
- `PRODUCT_ID`
- `FACILITY_ID`
- `QOH`
- `ATP`
- `DIFFERENCE`

```sql
select
PRODUCT_ID,
FACILITY_ID,
QUANTITY_ON_HAND as QOH,
AVAILABLE_TO_PROMISE as ATP,
(QUANTITY_ON_HAND - AVAILABLE_TO_PROMISE) as DIFFERENCE
from inventory_item
where QUANTITY_ON_HAND <> AVAILABLE_TO_PROMISE;
```

---

## 8.5 Order Item Current Status Changed Date-Time

**Business Problem:** Operations teams need to audit when an order item's status was last changed for tracking and dispute resolution.

**Fields to Retrieve:**
- `ORDER_ID`
- `ORDER_ITEM_SEQ_ID`
- `CURRENT_STATUS_ID`
- `STATUS_CHANGE_DATETIME`
- `CHANGED_BY`

```sql
select
os.ORDER_ID,
os.ORDER_ITEM_SEQ_ID,
os.STATUS_ID as CURRENT_STATUS_ID,
os.STATUS_DATETIME as STATUS_CHANGE_DATETIME,
os.STATUS_USER_LOGIN as CHANGED_BY
from order_status os
join (
select
ORDER_ID,
ORDER_ITEM_SEQ_ID,
max(STATUS_DATETIME) as MAX_STATUS_TIME
from order_status
where ORDER_ITEM_SEQ_ID is not null
group by ORDER_ID, ORDER_ITEM_SEQ_ID
) latest on os.ORDER_ID = latest.ORDER_ID
and os.ORDER_ITEM_SEQ_ID = latest.ORDER_ITEM_SEQ_ID
and os.STATUS_DATETIME = latest.MAX_STATUS_TIME;
```

---

## 8.6 Total Orders by Sales Channel

**Business Problem:** Marketing and sales teams want to see how many orders come from each sales channel and the revenue generated by each channel.

**Fields to Retrieve:**
- `SALES_CHANNEL`
- `TOTAL_ORDERS`
- `TOTAL_REVENUE`
- `REPORTING_PERIOD`

```sql
select
SALES_CHANNEL_ENUM_ID as SALES_CHANNEL,
count(*) as TOTAL_ORDERS,
sum(GRAND_TOTAL) as TOTAL_REVENUE,
concat(
min(date(ORDER_DATE)),
' to ',
max(date(ORDER_DATE))
) as REPORTING_PERIOD
from order_header
group by SALES_CHANNEL_ENUM_ID
order by TOTAL_REVENUE desc;
```
