# #1 Dependency Analysis: *get_hours()* in *jta*

## 1.Upstream SP
- ***process_payroll***
  - Reads: `work_hours`, `staff`
  - Invoking: `get_hours(current_staff.staff_id, v_start_date, v_end_date, basic, overtime, doubletime);` [JTA_Packages.sql:353](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L353)
- ***get_date***
  - Reads: `DUAL`
  - Invoking: `get_hours(v_staff_id, v_start_date, v_end_date, v_basic, v_overtime, v_doubletime);` [JTA_Packages.sql:1141](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L1141)

## 2.Downstream SP
- None
----
# #2 Mechanism Analysis for *get_hours()*
## 1.Preparation
- Input Parameters: `p_staff_id`, `start_date`, `end_date`; Output Parameters: `basic`, `overtime`, `doubletime`. [JTA_Packages.sql:255-262](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L255-L262)
- Initializes `basic := 0`, `overtime := 0`, `doubletime := 0`. [JTA_Packages.sql:266-268](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L266-L268)
## 2.Load Data
- Aggregates Sunday hours into `doubletime` from `work_hours` (`to_char(work_date, 'd') = '1'`). [JTA_Packages.sql:270-274](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L270-L274)
- Aggregates non-Sunday hours into `basic` from `work_hours` (`to_char(work_date, 'd') != '1'`). [JTA_Packages.sql:276-280](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L276-L280)
## 3.Transform and Logic Condition
- If `basic > 40`, sets `overtime := basic - 40` and caps `basic := 40`. [JTA_Packages.sql:282-285](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L282-L285)
## 4.Final Output
- Returns computed values through OUT parameters; no table writes occur in this procedure body. [JTA_Packages.sql:255-288](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L255-L288)
- Exception handling is delegated to the caller (per in-body note). [JTA_Packages.sql:287](../02Development_Zone/Oracle_Package/JTA_Packages.sql#L287)
----
# #3 Body Script of *get_hours()*
```sql
PROCEDURE get_hours (
    p_staff_id IN staff.staff_id%TYPE,
    start_date IN DATE,
    end_date IN DATE,
    basic OUT NOCOPY INTEGER,
    overtime OUT NOCOPY INTEGER,
    doubletime OUT NOCOPY INTEGER
)
IS
BEGIN
    
    basic := 0;
    overtime := 0;
    doubletime := 0;
    
    SELECT nvl(SUM(hours_worked), 0) INTO doubletime
        FROM work_hours
        WHERE work_date BETWEEN start_date AND end_date 
        AND staff_id = p_staff_id 
        AND to_char(work_date, 'd') = '1';
        
    SELECT nvl(SUM(hours_worked), 0) INTO basic
        FROM work_hours
        WHERE work_date BETWEEN start_date AND end_date 
        AND staff_id = p_staff_id 
        AND to_char(work_date, 'd') != '1';
    
    IF basic > 40 THEN
        overtime := basic - 40;
        basic := 40;
    END IF;
        
-- excpetions will be handled in called procedure
END;
```