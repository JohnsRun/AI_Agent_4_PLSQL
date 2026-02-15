# #1 Dependency Analysis: *get_date()* in *jta*

#### Upstream Procedures
1. **`get_profits_for()`**
    - **Reads**: `billed_items`, `customer_bills`, `cost_sales_tracker`
    - **Call Line**: `jta.get_date(v_current_time);` [JTA_Packages.sql:951](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L951)
    - **Procedure Body**: [JTA_Packages.sql:904-963](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L904-L963)
2. **`stock_check()`**
    - **Reads**: `inventory_by_location`
    - **Call Line**: `jta.get_date(v_current_time);` [JTA_Packages.sql:1782](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1782)
    - **Procedure Body**: [JTA_Packages.sql:1734-1797](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1734-L1797)

#### Downstream Procedures
1. **`get_hours()`**
    - **Body Location**: [JTA_Packages.sql:255](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L255)
    - **Call Line**: [JTA_Packages.sql:1141](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1141)
    - **Reads**: `work_hours`
2. **`jta_error.log_error()`**
    - **Body Location**: [JTA_Packages.sql:58](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L58)
    - **Call Line**: [JTA_Packages.sql:1151](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1151)
    - **Reads**: `--None--`
----
# #2 Mechanism Analysis for *get_date()*
## 1.Preparation
- Input Parameters: `p_local_time OUT NOCOPY DATE`; local variables initialized for hours and date-window calculation (`v_basic`, `v_overtime`, `v_doubletime`, `v_staff_id`, `v_start_date`, `v_end_date`). [JTA_Packages.sql:1119-1128](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1119-L1128)

## 2.Load Data
- Loads database server current datetime into `p_local_time` from `DUAL` using `SYSDATE`. [JTA_Packages.sql:1131](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1131)

## 3.Transform and Logic Condition
- Emits current local time via `dbms_output.put_line`. [JTA_Packages.sql:1134](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1134)
- Computes current-week boundaries: `v_start_date := TRUNC(p_local_time, 'DAY')` and `v_end_date := v_start_date + 6`. [JTA_Packages.sql:1137-1138](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1137-L1138)
- Invokes `get_hours(...)` for `v_staff_id = 1` over computed range, then prints hour breakdown. [JTA_Packages.sql:1141-1146](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1141-L1146)

## 4.Final Output
- Normal path returns populated `p_local_time` and console output only (no DML in this procedure). [JTA_Packages.sql:1131-1146](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1131-L1146)
- Exception path logs error with `jta_error.log_error`, nulls `p_local_time`, and prints failure message. [JTA_Packages.sql:1148-1153](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1148-L1153)
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