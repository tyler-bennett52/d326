
```sql
-- Assuming we have the following tables:
-- employees (employee_id, store_id)
-- sales (sale_id, employee_id, revenue)

-- Step 1: Aggregate the revenue by employee
WITH EmployeeRevenue AS (
    SELECT 
        employee_id,
        SUM(revenue) AS total_revenue
    FROM 
        sales
    GROUP BY 
        employee_id
)

-- Step 2: Join the aggregated revenue with the employees to get store-wise revenue
SELECT 
    e.store_id,
    SUM(er.total_revenue) AS TotalRevenue
FROM 
    EmployeeRevenue er
JOIN 
    employees e ON er.employee_id = e.employee_id
GROUP BY 
    e.store_id;
``t
