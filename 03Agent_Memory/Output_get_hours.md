# #1 Dependency Analysis: *get_date()* in *jta*

#### Upstream Procedures
1. **`get_profits_for`**
    - **Reads**: `billed_items`, `customer_bills`, `cost_sales_tracker`
    - **Call Line**: `jta.get_date(v_current_time);` [JTA_Packages.sql:951](02Development_Zone/Oracle_Package/JTA_Packages.sql#L951)
2. **`stock_check`**
    - **Reads**: `inventory_by_location`
    - **Call Line**: `jta.get_date(v_current_time);` [JTA_Packages.sql:1782](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1782)

#### Downstream Procedures
1. **`get_hours()`**
    - **Body Location**: [JTA_Packages.sql:255-288](02Development_Zone/Oracle_Package/JTA_Packages.sql#L255-L288)
    - **Call Line**: [JTA_Packages.sql:1141](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1141)
    - **Reads**: `work_hours`
2. **`jta_error.log_error()`**
    - **Body Location**: [JTA_Packages.sql:58-72](02Development_Zone/Oracle_Package/JTA_Packages.sql#L58-L72)
    - **Call Line**: [JTA_Packages.sql:1151](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1151)
    - **Reads**: `None`
----
# #2 Mechanism Analysis for *get_date()*
## 1.Preparation
- Input Parameters: `p_local_time`; Output Parameters: `p_local_time`. [JTA_Packages.sql:1119-1121](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1119-L1121)
- Initializes working variables `v_basic`, `v_overtime`, `v_doubletime`, `v_staff_id`, `v_start_date`, `v_end_date`. [JTA_Packages.sql:1123-1128](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1123-L1128)
 ## 2.Load Data
- Selects server date-time using `SELECT SYSDATE INTO p_local_time FROM DUAL`. [JTA_Packages.sql:1131](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1131)
- Computes weekly date range using `TRUNC(p_local_time, 'DAY')` and `v_start_date + 6`. [JTA_Packages.sql:1137-1138](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1137-L1138)
## 3.Transform and Logic Condition 
- 1) Invokes `get_hours(v_staff_id, v_start_date, v_end_date, v_basic, v_overtime, v_doubletime)` to load worked-hour buckets. [JTA_Packages.sql:1141](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1141)
- 2) Emits diagnostic output for current time, staff id, period, and hour totals via `dbms_output.put_line`. [JTA_Packages.sql:1134-1146](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1134-L1146)
- 3) On exception, logs error through `jta_error.log_error(SQLCODE, SQLERRM)` and sets `p_local_time := NULL`. [JTA_Packages.sql:1149-1152](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1149-L1152)
## 4.Final Output
- Returns `p_local_time` to caller; no table writes occur in `get_date` body. [JTA_Packages.sql:1131-1154](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1131-L1154)
- Target procedure table access summary: **Reads** `DUAL`; **Writes** `None`. [JTA_Packages.sql:1131](02Development_Zone/Oracle_Package/JTA_Packages.sql#L1131)
----
# #3 Body Script of *get_date()*
```sql
PROCEDURE get_date (
    p_local_time OUT NOCOPY DATE
)
IS
    v_basic INTEGER;
    v_overtime INTEGER;
    v_doubletime INTEGER;
    v_staff_id staff.staff_id%TYPE := 1; -- example staff_id
    v_start_date DATE;
    v_end_date DATE;
BEGIN
    -- retrieve current date and time from the database server
    SELECT SYSDATE INTO p_local_time FROM DUAL;
    
    -- display current date and time
    dbms_output.put_line('Current local time: ' || TO_CHAR(p_local_time, 'yyyy-Mon-dd HH24:MI:SS'));
    
    -- set date range (example: current week)
    v_start_date := TRUNC(p_local_time, 'DAY');
    v_end_date := v_start_date + 6;
    
    -- call get_hours to retrieve and display hours worked
    get_hours(v_staff_id, v_start_date, v_end_date, v_basic, v_overtime, v_doubletime);
    
    -- display hours information
    dbms_output.put_line('Staff ID: ' || v_staff_id);
    dbms_output.put_line('Period: ' || TO_CHAR(v_start_date, 'yyyy-Mon-dd') || ' to ' || TO_CHAR(v_end_date, 'yyyy-Mon-dd'));
    dbms_output.put_line('Basic hours: ' || v_basic || ', Overtime: ' || v_overtime || ', Double time: ' || v_doubletime);
    
EXCEPTION
    WHEN OTHERS THEN
        -- log error and set output to NULL
        jta_error.log_error(SQLCODE, SQLERRM);
        p_local_time := NULL;
        dbms_output.put_line('Error retrieving date and hours information');
END get_date;
```