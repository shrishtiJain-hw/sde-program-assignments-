# MODULE-8 SQL Assignment 3

## 1. Completed Sales Orders (Physical Items)

**Business Problem:** Merchants need to track only physical items requiring shipping and fulfillment for logistics and shipping-cost analysis.

**Fields to Retrieve:**
- `ORDER_ID`
- `ORDER_ITEM_SEQ_ID`
- `PRODUCT_ID`
- `PRODUCT_TYPE_ID`
- `SALES_CHANNEL_ENUM_ID`
- `ORDER_DATE`
- `ENTRY_DATE`
- `STATUS_ID`
- `STATUS_DATETIME`
- `ORDER_TYPE_ID`
- `PRODUCT_STORE_ID`

```sql
SELECT
    OH.ORDER_ID,
    OI.ORDER_ITEM_SEQ_ID,
    P.PRODUCT_ID,
    P.PRODUCT_TYPE_ID,
    OH.SALES_CHANNEL_ENUM_ID,
    OH.ORDER_DATE,
    OH.ENTRY_DATE,
    OS.STATUS_ID,
    OS.STATUS_DATETIME,
    OH.ORDER_TYPE_ID,
    OH.PRODUCT_STORE_ID
FROM 
    ORDER_HEADER OH
JOIN 
    ORDER_ITEM OI ON OH.ORDER_ID = OI.ORDER_ID
JOIN 
    ORDER_STATUS OS ON OH.ORDER_ID = OS.ORDER_ID 
                    AND OI.ORDER_ITEM_SEQ_ID = OS.ORDER_ITEM_SEQ_ID
JOIN 
    PRODUCT P ON OI.PRODUCT_ID = P.PRODUCT_ID
JOIN 
    PRODUCT_TYPE PT ON P.PRODUCT_TYPE_ID = PT.PRODUCT_TYPE_ID
WHERE 
    PT.IS_PHYSICAL = 'Y'
    AND OH.STATUS_ID = 'ORDER_COMPLETED'
    AND OH.ORDER_TYPE_ID = 'SALES_ORDER';

```

---

## 2. Completed Return Items

**Business Problem:** Customer service and finance teams need insights into returned items to manage refunds, replacements, and inventory restocking.

**Fields to Retrieve:**
- `RETURN_ID`
- `ORDER_ID`
- `PRODUCT_STORE_ID`
- `STATUS_DATETIME`
- `ORDER_NAME`
- `FROM_PARTY_ID`
- `RETURN_DATE`
- `ENTRY_DATE`
- `RETURN_CHANNEL_ENUM_ID`

```sql
select
rh.RETURN_ID,
ri.ORDER_ID,
oh.PRODUCT_STORE_ID,
rs.STATUS_DATETIME,
oh.ORDER_NAME,
rh.FROM_PARTY_ID,
rh.RETURN_DATE,
rh.ENTRY_DATE,
rh.RETURN_CHANNEL_ENUM_ID
from return_header rh
join return_item ri on ri.RETURN_ID = rh.RETURN_ID
join order_header oh on oh.ORDER_ID = ri.ORDER_ID
join return_status rs on rs.RETURN_ID = rh.RETURN_ID
where rs.STATUS_ID = 'RETURN_COMPLETED'
and rs.RETURN_ITEM_SEQ_ID is null;
```

---

## 3. Single-Return Orders (Last Month)

**Business Problem:** The merchandising team needs a list of customers whose orders had exactly one return during the previous month.

**Fields to Retrieve:**
- `PARTY_ID`
- `FIRST_NAME`

```sql
select distinct
p.PARTY_ID,
p.FIRST_NAME
from person p
join order_role orl on p.PARTY_ID = orl.PARTY_ID
join (
select
ri.ORDER_ID
from return_item ri
join return_header rh on rh.RETURN_ID = ri.RETURN_ID
where rh.RETURN_DATE >= date_format(curdate() - interval 1 month, '%Y-%m-01')
and rh.RETURN_DATE < date_format(curdate(), '%Y-%m-01')
group by ri.ORDER_ID
having count(distinct ri.RETURN_ID) = 1
) r on r.ORDER_ID = orl.ORDER_ID
where orl.ROLE_TYPE_ID = 'PLACING_CUSTOMER';
```

---

## 4. Returns and Appeasements

**Business Problem:** The retailer needs total return volume and total appeasements issued.

**Fields to Retrieve:**
- `TOTAL_RETURNS`
- `RETURN_DOLLAR_TOTAL`
- `TOTAL_APPEASEMENTS`
- `APPEASEMENTS_DOLLAR_TOTAL`

```sql
select
  (select coalesce(sum(ri.RETURN_QUANTITY), 0)
   from return_item ri
   join return_header rh on ri.RETURN_ID = rh.RETURN_ID
   where rh.STATUS_ID = 'RETURN_COMPLETED') as TOTAL_RETURNS,

  (select coalesce(sum(ri.RETURN_QUANTITY * ri.RETURN_PRICE), 0)
   from return_item ri
   join return_header rh on ri.RETURN_ID = rh.RETURN_ID
   where rh.STATUS_ID = 'RETURN_COMPLETED') as RETURN_DOLLAR_TOTAL,

  (select count(*)
   from return_adjustment
   where RETURN_ADJUSTMENT_TYPE_ID = 'APPEASEMENT') as TOTAL_APPEASEMENTS,

  (select coalesce(sum(AMOUNT), 0)
   from return_adjustment
   where RETURN_ADJUSTMENT_TYPE_ID = 'APPEASEMENT') as APPEASEMENTS_DOLLAR_TOTAL;

```

---

## 5. Detailed Return Information

**Business Problem:** Operations and finance teams require detailed return information for return analysis and refund tracking.

**Fields to Retrieve:**
- `RETURN_ID`
- `ENTRY_DATE`
- `RETURN_ADJUSTMENT_TYPE_ID`
- `AMOUNT`
- `COMMENTS`
- `ORDER_ID`
- `ORDER_DATE`
- `RETURN_DATE`
- `PRODUCT_STORE_ID`

```sql
select
  rh.RETURN_ID,
  rh.ENTRY_DATE,
  ra.RETURN_ADJUSTMENT_TYPE_ID,
  ra.AMOUNT,
  ra.COMMENTS,
  ro.ORDER_ID,
  oh.ORDER_DATE,
  rh.RETURN_DATE,
  oh.PRODUCT_STORE_ID
from return_header rh
left join (
  select RETURN_ID, min(ORDER_ID) as ORDER_ID
  from return_item
  group by RETURN_ID
) ro on rh.RETURN_ID = ro.RETURN_ID
left join order_header oh on oh.ORDER_ID = ro.ORDER_ID
left join return_adjustment ra on rh.RETURN_ID = ra.RETURN_ID;

```

---

## 6. Orders with Multiple Returns

**Business Problem:** Orders with multiple returns may indicate fraud, product quality issues, or fulfillment problems.

**Fields to Retrieve:**
- `ORDER_ID`
- `RETURN_ID`
- `RETURN_DATE`
- `RETURN_REASON`
- `RETURN_QUANTITY`

```sql
select
ri.ORDER_ID,
rh.RETURN_ID,
rh.RETURN_DATE,
ri.RETURN_REASON_ID as RETURN_REASON,
ri.RETURN_QUANTITY
from return_header rh
join return_item ri on rh.RETURN_ID = ri.RETURN_ID
where ri.ORDER_ID in (
select ORDER_ID
from return_item
group by ORDER_ID
having count(distinct RETURN_ID) > 1
)
order by ri.ORDER_ID;
```

---

## 7. Store with Most One-Day Shipped Orders (Last Month)

**Business Problem:** Identify facilities handling the highest volume of one-day shipped orders.

**Fields to Retrieve:**
- `FACILITY_ID`
- `FACILITY_NAME`
- `TOTAL_ONE_DAY_SHIP_ORDERS`
- `REPORTING_PERIOD`

```sql
select
  s.ORIGIN_FACILITY_ID as FACILITY_ID,
  f.FACILITY_NAME,
  count(distinct s.PRIMARY_ORDER_ID) as TOTAL_ONE_DAY_SHIP_ORDERS,
  concat(
    date_format(curdate() - interval 1 month, '%Y-%m'),
    ' Last Month'
  ) as REPORTING_PERIOD
from shipment s
join facility f on f.FACILITY_ID = s.ORIGIN_FACILITY_ID
where timestampdiff(day, s.ESTIMATED_READY_DATE, s.ESTIMATED_SHIP_DATE) <= 1
  and s.ESTIMATED_SHIP_DATE >= date_format(curdate() - interval 1 month, '%Y-%m-01')
  and s.ESTIMATED_SHIP_DATE < date_format(curdate(), '%Y-%m-01')
group by
  s.ORIGIN_FACILITY_ID,
  f.FACILITY_NAME
order by TOTAL_ONE_DAY_SHIP_ORDERS desc;

```

---

## 8. List of Warehouse Pickers

**Business Problem:** Warehouse managers need employee assignments for picking and packing operations.

**Fields to Retrieve:**

-`PARTY_ID (or Employee ID)`
-`NAME (First/Last)`
-`ROLE_TYPE_ID (e.g., “WAREHOUSE_PICKER”)`
-`FACILITY_ID (assigned warehouse)`
-`STATUS (active or inactive employee)`

```sql
SELECT
  pr.PARTY_ID,
  CONCAT(p.FIRST_NAME, ' ', p.LAST_NAME) AS NAME,
  pr.ROLE_TYPE_ID,
  fp.FACILITY_ID,
  pty.STATUS_ID AS STATUS
FROM party_role pr
JOIN person p ON p.PARTY_ID = pr.PARTY_ID
JOIN party pty ON pty.PARTY_ID = pr.PARTY_ID 
LEFT JOIN facility_party fp ON fp.PARTY_ID = pr.PARTY_ID
WHERE pr.ROLE_TYPE_ID = 'WAREHOUSE_PICKER';


```

---

## 9. Total Facilities That Sell the Product

**Business Problem:** Identify how many facilities sell each product and where those products are available.

**Fields to Retrieve:**
- `PRODUCT_ID`
- `PRODUCT_NAME`
- `FACILITY_COUNT`
- `FACILITY_LIST`

```sql
select
pf.PRODUCT_ID,
p.INTERNAL_NAME as PRODUCT_NAME,
count(distinct pf.FACILITY_ID) as FACILITY_COUNT,
group_concat(distinct pf.FACILITY_ID) as FACILITY_LIST
from product_facility pf
join product p on p.PRODUCT_ID = pf.PRODUCT_ID
group by
pf.PRODUCT_ID,
p.INTERNAL_NAME;
```

---

## 10. Total Items in Various Facilities

**Business Problem:** Analyze inventory levels across facilities and facility types.

**Fields to Retrieve:**
- `PRODUCT_ID`
- `FACILITY_ID`
- `FACILITY_TYPE_ID`
- `QOH`
- `ATP`

```sql
select
ii.PRODUCT_ID,
ii.FACILITY_ID,
f.FACILITY_TYPE_ID,
ii.QUANTITY_ON_HAND_TOTAL as QOH,
ii.AVAILABLE_TO_PROMISE_TOTAL as ATP
from inventory_item ii
join facility f on f.FACILITY_ID = ii.FACILITY_ID;
```

---

## 11. Transfer Orders Without Inventory Reservation

**Business Problem:** Identify transfer orders that do not have reserved inventory.

**Fields to Retrieve:**
- `TRANSFER_ORDER_ID`
- `FROM_FACILITY_ID`
- `TO_FACILITY_ID`
- `PRODUCT_ID`
- `REQUESTED_QUANTITY`
- `RESERVED_QUANTITY`
- `TRANSFER_DATE`
- `STATUS`

```sql
select
  oh.ORDER_ID as TRANSFER_ORDER_ID,
  s.ORIGIN_FACILITY_ID as FROM_FACILITY_ID,
  s.DESTINATION_FACILITY_ID as TO_FACILITY_ID,
  oi.PRODUCT_ID,
  oi.QUANTITY as REQUESTED_QUANTITY,
  coalesce(sum(oisgir.QUANTITY), 0) as RESERVED_QUANTITY,
  s.ESTIMATED_SHIP_DATE as TRANSFER_DATE,
  oh.STATUS_ID as STATUS
from order_header oh
join order_item oi on oi.ORDER_ID = oh.ORDER_ID
join shipment s on s.PRIMARY_ORDER_ID = oh.ORDER_ID
left join order_item_ship_grp_inv_res oisgir on oisgir.ORDER_ID = oi.ORDER_ID
  and oisgir.ORDER_ITEM_SEQ_ID = oi.ORDER_ITEM_SEQ_ID
where oh.ORDER_TYPE_ID = 'TRANSFER_ORDER' 
group by
  oh.ORDER_ID,
  s.ORIGIN_FACILITY_ID,
  s.DESTINATION_FACILITY_ID,
  oi.PRODUCT_ID,
  oi.QUANTITY,
  s.ESTIMATED_SHIP_DATE,
  oh.STATUS_ID
having RESERVED_QUANTITY = 0;

```

---

## 12. Orders Without Picklist

**Business Problem:** Orders missing picklists may experience fulfillment delays and require operational attention.

**Fields to Retrieve:**
- `ORDER_ID`
- `ORDER_DATE`
- `ORDER_STATUS`
- `FACILITY_ID`
- `DURATION_DAYS`

```sql
select distinct
oh.ORDER_ID,
oh.ORDER_DATE,
oh.STATUS_ID as ORDER_STATUS,
oisg.FACILITY_ID,
timestampdiff(day, oh.ORDER_DATE, current_timestamp) as DURATION_DAYS
from order_header oh
join order_item_ship_group oisg on oisg.ORDER_ID = oh.ORDER_ID
left join picklist_item pli on pli.ORDER_ID = oh.ORDER_ID
where pli.PICKLIST_BIN_ID is null
and oh.STATUS_ID not in ('ORDER_COMPLETED', 'ORDER_CANCELLED')
order by DURATION_DAYS desc;
```
