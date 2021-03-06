# **HR Analytics - PEAR**

## PEAR
  - P - Problem
  - E - Exploration
  - A - Analysis
  - R - Result

<br>

## **Problem**

* We have been asked by the HR Analytica to generate reusable datasets to power 2 HR analytics tools.
* Generate database views that HR Analytica will use for 2 key dashboards - reporting solutions and ad-hoc analytics requests. 
* Also fix the date related fields.

<br>

## Required Insights 

* The following insights must be generated for the 2 dashboards as requested by HR Analytica.

<br>

## Company Level

### Splitting by gender - current snapshot

* Total number of employees
* Average company tenure in years
* Average latest payrise percentage
* Statistical metrics for salary values including: ```MEAN```, ```MAX```, ```STDDEV```, Inter quartile range and median.


<br>

## Department Level

* Similar to company level, just at department level that includes additional department levek tenure metrics split by gender.

<br>


## Title level

* Siilar to department level but a little level of granularity.

<br>

## Deep diving on the employee relation

* Highlight recent event for every single employee form time to time.
* See all the various employment history ordered by effective date including salary, department, manager and title changes
* Calculate previous historic payrise percentages and value changes
* Calculate the previous position and department history in months with start and end dates
* Compare an employee’s current salary, total company tenure, department, position and gender to the average benchmarks for their current position

<br>

## Outputs

### Current snapshot reporting

<br>
<p align="center">
  <img width="498" height="746" src="Current_Snapshot.png">
</p>

### Historic Employee Deep Dive 

<br>
<p align="center">
  <img width="498" height="746" src="Historic_deep_dive.png">
</p>

<br>


## Exploration

### ER Diagram 

<br>
<p align="center">
  <img width="750" height="450" src="ER.png">
</p>

<br>


## Data Exploration

* Exploring the schema ```employees``` by accessing ```pg_indexes``` to get the index information.

<br>

```sql
SELECT *
FROM pg_indexes
WHERE schemaname = 'employees';

```

<br>

| schemaname	| tablename	| indexname	| indexdef  |
| :---:| :---:| :---:| :---:|
|employees|	employee|	idx_16988_primary|	CREATE UNIQUE INDEX idx_16988_primary ON employees.employee USING btree (id)|
|employees|	department_employee|	idx_16982_primary|	CREATE UNIQUE INDEX idx_16982_primary ON employees.department_employee USING btree (employee_id, department_id)|
|employees|	department_employee|	idx_16982_dept_no|	CREATE INDEX idx_16982_dept_no ON employees.department_employee USING btree (department_id)|
|employees|	department|	idx_16979_primary|	CREATE UNIQUE INDEX idx_16979_primary ON employees.department USING btree (id)|
|employees|	department|	idx_16979_dept_name|	CREATE UNIQUE INDEX idx_16979_dept_name ON employees.department USING btree (dept_name)|
|employees|	department_manager|	idx_16985_primary|	CREATE UNIQUE INDEX idx_16985_primary ON employees.department_manager USING btree (employee_id, department_id)|
|employees|	department_manager|	idx_16985_dept_no|	CREATE INDEX idx_16985_dept_no ON employees.department_manager USING btree (department_id)|
|employees|	salary|	idx_16991_primary|	CREATE UNIQUE INDEX idx_16991_primary ON employees.salary USING btree (employee_id, from_date)|
|employees|	title|	idx_16994_primary|	CREATE UNIQUE INDEX idx_16994_primary ON employees.title USING btree (employee_id, title, from_date)|

<br>

From the above table we can observe that:
1. The following tables have unique indexes on a single column:
    * ```employees.employee```
    * ```employees.department```

2. The rest of the tables have multiple records for ```employee_id``` values.
    * ```employees.department_employee```
    * ```employees.department_manager```
    * ```employee.salary```
    * ```employee.title```
 
##  Individual Table Analysis

### Employee Table

* Confirming whether there is a single record per employee.

<br>

```sql

with cte AS  
  (SELECT 
    id,
    COUNT(*) as row_count
  FROM employees.employee
  GROUP BY id)
SELECT
  row_count,
  COUNT(DISTINCT id) as employee_count
FROM cte 
GROUP BY 1
ORDER BY 1;

```

<br>

| row_count	| employee_count|
| :---:| :---:|
| 1 | 300024|

<br>

### Department Table 

* There are 9 unique departments 

<br>

```sql

SELECT 
  *
FROM employees.department;

```

| id	| dept_name|
| :---:| :---:|
|d001|	Marketing|
|d002|	Finance|
|d003|	Human Resources|
|d004|	Production|
|d005|	Development|
|d006|	Quality Management|
|d007|	Sales|
|d008|	Research|
|d009|	Customer Service|

<br>

### Department Employee table 

```sql

SELECT 
  *
FROM employees.department_employee
LIMIT 5;

```

| employee_id	| department_id| from_date	| to_date|
| :---:| :---:| :---:| :---:|
|10001|	d005|	1986-06-26|	9999-01-01|
|10002|	d007|	1996-08-03|	9999-01-01|
|10003|	d004|	1995-12-03|	9999-01-01|
|10004|	d004|	1986-12-01|	9999-01-01|
|10005|	d003|	1989-09-12|	9999-01-01|


<br>

* In the ```department_employee``` table we have a column named ```to_date = '9999-01-01'``` 
* Lets investigate the distribution of the ```to_date``` column.

<br>

```sql

SELECT 
  to_date,
  COUNT(*) as record_count
FROM employees.department_employee
GROUP BY 1
ORDER BY 1 DESC
LIMIT 5;

```

<br>

| to_date	| record_count|
| :---:| :---:|
|9999-01-01|	240124|
|2000-04-14|	48|
|2000-03-29|	46|
|2001-02-10|	46|
|1999-12-06|	45|

<br>

* We see that there are many values related to ```to_date```. We have many values for ```9999-01-01```. 
* Now let's confirm that we have a many-to-one relationship between ```department_employee``` and its ```employee_id```


```sql

with employee_id_cte AS 
  (SELECT 
    employee_id,
    COUNT(*) as row_count
  FROM employees.department_employee
  GROUP BY 1)
SELECT 
  row_count,
  COUNT(DISTINCT employee_id) as employee_count 
FROM employee_id_cte
GROUP BY 1 
ORDER BY 1 DESC;

```

* We can see that there are approximately 10% rows with 2 records. i.e. there are multiple records per ```employee_id```


### Department manager table

```sql 

SELECT 
FROM employees.department_manager
LIMIT 5;

```

<br>

| employee_id	| department_id| from_date	| to_date|
| :---:| :---:| :---:| :---:|
|110022|	d001|	1985-01-01|	1991-10-01|
|110039|	d001|	1991-10-01|	9999-01-01|
|110085|	d002|	1985-01-01|	1989-12-17|
|110114|	d002|	1989-12-17|	9999-01-01|
|110183|	d003|	1985-01-01|	1992-03-21|

<br>

* Investigating ```to_date``` column 

<br>

```sql

SELECT 
  to_date,
  COUNT(*) as record_count
FROM employees.department_manager
GROUP BY 1
ORDER BY 2 DESC;

```
<br>

| to_date	| record_count|
| :---:| :---:|
|9999-01-01|	9|
|1994-06-28|	1|
|1991-09-12|	1|
|1992-04-25|	1|
|1991-03-07|	1|
|1992-03-21|	1|
|1991-10-01|	1|
|1992-08-02|	1|
|1988-10-17|	1|
|1996-01-03|	1|
|1988-09-09|	1|
|1991-04-08|	1|
|1989-05-06|	1|
|1989-12-17|	1|
|1996-08-30|	1|
|1992-09-08|	1|

<br>

* Confirming rows per ```employee_id```

<br>

```sql

WITH employee_id_cte AS 
  (SELECT 
    employee_id,
    COUNT(*) as row_count
  FROM employees.department_manager
  GROUP BY 1)
SELECT 
  row_count,
  COUNT(DISTINCT employee_id) AS employee_count
FROM employee_id_cte 
GROUP BY 1
ORDER BY 1 DESC;

```

<br>

| to_date	| record_count|
| :---:| :---:|
| 1 | 24 |

* From the above query we can see that each ```employee_id``` that appears in ```empoyees.department_manager``` table will have only have a single record or a one-to-one relationship.


### Salary Table 

* Salary table comprises of salary of an employee over time. 

<br>

```sql 

SELECT 
  *
FROM employees.salary
WHERE employee_id = 100001
ORDER BY from_date DESC;

```

<br>

| employee_id	| amount| from_date | to_date |
| :---:| :---:| :---:| :---:|
|10001|	88958|	2002-06-22|	9999-01-01|
|10001|	85097|	2001-06-22|	2002-06-22|
|10001|	85112|	2000-06-22|	2001-06-22|
|10001|	84917|	1999-06-23|	2000-06-22|
|10001|	81097|	1998-06-23|	1999-06-23|
|10001|	81025|	1997-06-23|	1998-06-23|
|10001|	80013|	1996-06-23|	1997-06-23|
|10001|	76884|	1995-06-24|	1996-06-23|
|10001|	75994|	1994-06-24|	1995-06-24|
|10001|	75286|	1993-06-24|	1994-06-24|
|10001|	74333|	1992-06-24|	1993-06-24|
|10001|	71046|	1991-06-25|	1992-06-24|
|10001|	66961|	1990-06-25|	1991-06-25|
|10001|	66596|	1989-06-25|	1990-06-25|
|10001|	66074|	1988-06-25|	1989-06-25|
|10001|	62102|	1987-06-26|	1988-06-25|
|10001|	60117|	1986-06-26|	1987-06-26|

<br>

* Investigative the ```to_date``` column 

```sql

SELECT 
  to_date,
  COUNT(*) as record_count,
  COUNT(DISTINCT employee_id) as employee_count
FROM employees.salary
GROUP BY 1 
ORDER BY 1 DESC
LIMIT 5;

```

<br>

| employee_id	| amount| from_date |
| :---:| :---:| :---:|
|9999-01-01|	240124|	240124|
|2020-08-01|	686|	686|
|2020-07-31|	641|	641|
|2020-07-30|	673|	673|
|2020-07-29|	679|	679|

<br>

* Checking for duplicate ```employee_id```

<br>

```sql

WITH employee_id_cte AS 
  (SELECT 
    employee_id,
    COUNT(*) as row_count
  FROM employees.salary
  GROUP BY 1)
SELECT
  row_count,
  COUNT(DISTINCT employee_id) as employee_count
FROM employee_id_cte
GROUP BY 1
ORDER BY 1 DESC;

```

<br>

| row_count	| employee_count|
| :---:| :---:|
|18|	8180|
|17|	16106|
|16|	16331|
|15|	16799|
|14|	17193|
|13|	17318|
|12|	17530|
|11|	17952|
|10|	18474|
|9|	18815|
|8|	19281|
|7|	19715|
|6|	20403|
|5|	21335|
|4|	22038|
|3|	15784|
|2|	8361|
|1|	8409|

<br>

* From the above table we can observe that there are more than one record per employee for majority of the records. We need to be careful while joining.

## Analysis 

#### Splitting the SQL solution into following parts
   
   1. Data Cleaning and Date Adjustments
   2. Current Snapshot Analysis 
   3. Historical Analysis 
    

## 1. Data Cleaning

* First we will have to adjust all the date firlds identified by HR Analytica
* Incrementing all of the date fileds except the end date ```9999-01-01``` 
* Also cast the ```DATE``` to ```TIMESTAMP``` to keep the data original and as similar as possible.
* Creating materialized views with same original tables 

<br>


```sql

DROP SCHEMA IF EXISTS mv_employees CASCADE;
CREATE SCHEMA mv_employees;


-- department 

DROP MATERIALIZED VIEW IF EXISTS mv_employees.department;
CREATE MATERIALIZED VIEW mv_employees.department AS 
SELECT 
  *
FROM employees.department;



--department employee 


DROP MATERIALIZED VIEW IF EXISTS mv_employees.department_employee;
CREATE MATERIALIZED VIEW mv_employees.department_employee AS 
SELECT 
  employee_id,
  department_id,
  (from_date + interval '18 years')::DATE as from_date,
  CASE 
    WHEN to_date <> '9999-01-01' THEN (to_date + interval '18 years')::DATE
    ELSE to_date 
    END AS to_date
FROM employees.department_employee;


-- department_manager 


DROP MATERIALIZED VIEW IF EXISTS mv_employees.department_manager;
CREATE MATERIALIZED VIEW mv_employees.department_manager AS 
SELECT 
  employee_id,
  department_id,
  (from_date + interval '18 years')::DATE as from_date,
  CASE 
    WHEN to_date <> '9999-01-01' THEN (to_date + interval '18 years')::DATE
    ELSE to_date 
    END AS to_date
FROM employees.department_manager;


-- employee 

DROP MATERIALIZED VIEW IF EXISTS mv_employees.employee;
CREATE MATERIALIZED VIEW mv_employees.employee AS 
SELECT 
  id,
  (birth_date + interval '18 years')::DATE as birth_date,
  first_name,
  last_name,
  gender,
  (hire_date + interval '18 years')::DATE as hire_date
FROM employees.employee;


-- salary 


DROP MATERIALIZED VIEW IF EXISTS mv_employees.salary;
CREATE MATERIALIZED VIEW mv_employees.salary AS 
SELECT 
  employee_id,
  amount,
  (from_date + interval '18 years')::DATE AS from_date,
  CASE 
    WHEN to_date <> '9999-01-01' THEN (to_date + interval '18 years')::DATE 
    ELSE to_date 
    END AS to_date
FROM employees.salary;


-- title 

DROP MATERIALIZED VIEW IF EXISTS mv_employees.title;
CREATE MATERIALIZED VIEW mv_employees.title AS 
SELECT 
  employee_id,
  title,
  (from_date + interval '18 years' )::DATE AS from_date,
  CASE 
    WHEN to_date <> '9999-01-01' THEN (to_date + interval '18 years')::DATE 
    ELSE to_date 
    END AS to_date
FROM employees.title;


-- Creating indexes 


CREATE UNIQUE INDEX ON mv_employees.employee USING btree (id);
CREATE UNIQUE INDEX ON mv_employees.department_employee USING btree (employee_id, department_id);
CREATE INDEX        ON mv_employees.department_employee USING btree (department_id);
CREATE UNIQUE INDEX ON mv_employees.department USING btree (id);
CREATE UNIQUE INDEX ON mv_employees.department USING btree (dept_name);
CREATE UNIQUE INDEX ON mv_employees.department_manager USING btree (employee_id, department_id);
CREATE INDEX        ON mv_employees.department_manager USING btree (department_id);
CREATE UNIQUE INDEX ON mv_employees.salary USING btree (employee_id, from_date);
CREATE UNIQUE INDEX ON mv_employees.title USING btree (employee_id, title, from_date);

```

<br>

### Current Snapshot Analysis
    
* We will create a snapshot view whoch we will use as a base for each of the aggregated layers for different dashboard outputs.
    
#### Analysis Plan

1. Apply LAG window functions on the ```salary``` materialized view to obtain the latest ```previous_salary``` value, keeping only current valid records with ```to_date = '9999-01-01'```
2. Join previous salary and all other required information from the materialized views for the dashboard analysis (omitting the department_manager view)
3. Apply ```WHERE``` filter to keep only current records
4. Make sure to include the ```gender``` column from the ```employee``` view for all calculations
5. Use the ```hire_date``` column from the ```employee``` view to calculate the number of tenure years
6. Include the ```from_date``` columns from the ```title``` and ```department``` are included to calculate tenure
7. Use the ```salary``` table to calculate the current average salary
8. Include ```department``` and ```title``` information for additional group by aggregations
9. Implement the various statistical measures for the salary amount
10. Combine all of these elements into a single final current snapshot view
    

#### Current Employee Snapshot 


```sql

DROP VIEW IF EXISTS mv_employees.current_employee_snapshot CASCADE;
CREATE VIEW mv_employees.current_employee_snapshot AS 
WITH cte_previous_salary AS   
  (SELECT
    *
  FROM 
    (SELECT 
      employee_id,
      to_date,
      LAG(amount) OVER (PARTITION BY employee_id ORDER BY from_date) AS amount
    FROM mv_employees.salary) all_salaries
  WHERE to_date = '9999-01-01'),
cte_joined_data AS
  (SELECT 
    e.id as employee_id,
    e.gender,
    e.hire_date,
    t.title,
    s.amount as salary,
    p.amount as previous_salary,
    d.dept_name as department,
    t.from_date as title_from_date,
    dp.from_date as department_from_date
  FROM mv_employees.employee e  
  INNER JOIN mv_employees.title t 
  ON e.id = t.employee_id 
  INNER JOIN mv_employees.salary s 
  ON e.id = s.employee_id 
  INNER JOIN cte_previous_salary p 
  ON e.id = p.employee_id 
  INNER JOIN mv_employees.department_employee dp 
  ON e.id = dp.employee_id 
  INNER JOIN mv_employees.department d 
  ON dp.department_id = d.id 
  WHERE s.to_date = '9999-01-01' AND t.to_date = '9999-01-01' and dp.to_date = '9999-01-01'),
final_output AS 
(SELECT 
  employee_id,
  gender,
  title,
  salary,
  department,
  ROUND(100*(salary - previous_salary) / (previous_salary)::NUMERIC,2) AS salary_percentage_change,
  DATE_PART('year',now()) - DATE_PART('year',hire_date) AS company_tenure_years,
  DATE_PART('year',now()) - DATE_PART('year',title_from_date) AS title_tenure_years,
  DATE_PART('year',now()) - DATE_PART('year',department_from_date) AS department_tenure_years
FROM cte_joined_data)
SELECT 
  *
FROM final_output;


SELECT 
  * 
FROM mv_employees.current_employee_snapshot 
LIMIT 5;

```

<br>

| employee_id	| gender| title	| salary| department	| salary_percentage_change| company_tenure_years	| title_tenure_years| department_tenure_years	| 
| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:|
|10001|	M|	Senior Engineer|	88958|	Development|	4.54|	17|	17|	17|
|10002|	F|	Staff|	72527|	Sales|	0.78|	18|	7|	7|
|10003|	M|	Senior Engineer|	43311|	Production|	-0.89|	17|	8|	8|
|10004|	M|	Senior Engineer|	74057|	Production|	4.75|	17|	8|	17|
|10005|	M|	Senior Staff|	94692|	Human Resources|	3.54|	14|	7|	14|

<br>


### Dashboard Aggregation Views 

<br>

#### Company Level

<br>

```sql

DROP VIEW IF EXISTS mv_employees.company_level_dashboard;
CREATE VIEW mv_employees.company_level_dashboard AS 
SELECT 
  gender,
  COUNT(*) AS employee_count,
  ROUND(100 * COUNT(*)::NUMERIC/ SUM(COUNT(*)) OVER ()) AS employee_percentage,
  ROUND(AVG(salary)) AS avg_salary,
  ROUND(AVG(salary_percentage_change)) AS avg_salary_percentage_change,
  ROUND(MIN(salary)) AS min_salary,
  ROUND(MAX(salary)) AS max_salary,
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)) AS median_salary,
  ROUND(PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) - 
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary)) AS inter_quartile_range,
  ROUND(STDDEV(salary)) AS stddev_salary
FROM mv_employees.current_employee_snapshot
GROUP BY gender;

```

<br>

| gender	| employee_count| employee_percentage	| avg_salary| avg_salary_percentage_change	| min_salary| max_salary	| median_salary| inter_quartile_range	| stddev_salary|
| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:|
|M|	144114|	60|	13|	72045|	3|	38623|	158220|	69830|	23624|	17363|
|F|	96010|	40|	13|	71964|	3|	38936|	152710|	69764|	23326|	17230|

<br>

#### Department Level

<br>

```sql

DROP VIEW IF EXISTS mv_employees.department_level_dashboard;
CREATE VIEW mv_employees.department_level_dashboard AS 
SELECT 
  gender,
  department,
  COUNT(*) AS employee_count,
  ROUND(100 * COUNT(*)::NUMERIC/ SUM(COUNT(*)) OVER (PARTITION BY department)) AS employee_percentage,
  ROUND(AVG(department_tenure_years)) AS department_tenure,
  ROUND(AVG(salary)) AS avg_salary,
  ROUND(AVG(salary_percentage_change)) AS avg_salary_percentage_change,
  ROUND(MIN(salary)) AS min_salary,
  ROUND(MAX(salary)) AS max_salary,
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)) AS median_salary,
  ROUND(PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) - 
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary)) AS inter_quartile_range,
  ROUND(STDDEV(salary)) AS stddev_salary
FROM mv_employees.current_employee_snapshot
GROUP BY 1,2;


SELECT 
  *
FROM mv_employees.department_level_dashboard
LIMIT 5;

```

<br>

| gender	| department| employee_count| employee_percentage	|department_tenure| avg_salary| avg_salary_percentage_change	| min_salary| max_salary	| median_salary| inter_quartile_range	| stddev_salary|
| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:|
|M|	Customer Service|	10562|	60|	9|	67203|	3|	39373|	143950|	65100|	20097|	15921|
|M|	Development|	36853|	40|	11|	67713|	3|	39036|	140784|	66526|	19664|	14267|
|M|	Finance|	7423|	60|	11|	78433|	3|	39012|	142395|	77526|	24078	|17242|
|M|	Human Resources|	7751|	40|	11|	63777|	3|	39611|	141953|	62864|	17607|	12843|
|M|	Marketing|	8978|	60|	10|	80293|	3|	39821|	145128|	79481|	24990|	17480|

<br>

#### Title Level

<br>

```sql

DROP VIEW IF EXISTS mv_employees.title_level_dashboard;
CREATE VIEW mv_employees.title_level_dashboard AS 
SELECT 
  gender,
  title,
  COUNT(*) AS employee_count,
  ROUND(100 * COUNT(*)::NUMERIC/ SUM(COUNT(*)) OVER (PARTITION BY title)) AS employee_percentage,
  ROUND(AVG(title_tenure_years)) AS title_tenure,
  ROUND(AVG(salary)) AS avg_salary,
  ROUND(AVG(salary_percentage_change)) AS avg_salary_percentage_change,
  ROUND(MIN(salary)) AS min_salary,
  ROUND(MAX(salary)) AS max_salary,
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)) AS median_salary,
  ROUND(PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) - 
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary)) AS inter_quartile_range,
  ROUND(STDDEV(salary)) AS stddev_salary
FROM mv_employees.current_employee_snapshot
GROUP BY 1,2;

SELECT 
  *
FROM mv_employees.title_level_dashboard
LIMIT 5;

```

<br>


| gender	| title| employee_count| employee_percentage	|title_tenure| avg_salary| avg_salary_percentage_change	| min_salary| max_salary	| median_salary| inter_quartile_range	| stddev_salary|
| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:| :---:|
|M|	Assistant Engineer|	2148|	60|	6|	57198|	4|	39827|	117636|	54384|	14972|	11152|
|F|	Assistant Engineer|	1440|	40|	6|	57496|	4|	39469|	106340|	55234	|14679|	10805|
|M|	Engineer|	18571|	60|	6|	59593|	4|	38942|	130939|	56941|	17311|	12416|
|F|	Engineer|	12412|	40|	6|	59617|	4|	39519|	115444|	57220|	17223|	12211|
|M|	Manager|	5|	56|	9|	79351|	2|	56654|	106491|	72876|	43242|	23615|

<br>


### SQL solution - Historic Employee Results 

```sql

CREATE OR REPLACE VIEW mv_employees.current_employee_snapshot AS 
WITH cte_previous_salary AS (
  SELECT 
    * 
  FROM
    (SELECT 
      employee_id,
      to_date,
      LAG(amount) OVER (PARTITION BY employee_id ORDER BY from_date) AS amount
    FROM mv_employees.salary) all_salaries
  WHERE to_date = '9999-01-01'
  ),

cte_joined_data AS (
  SELECT 
    e.id AS employee_id,
    CONCAT_WS(' ',e.first_name, e.last_name) AS employee_name,
    e.gender,
    e.hire_date,
    t.title,
    s.amount AS salary,
    c.amount AS previous_salary,
    d.dept_name AS department,
    CONCAT_WS(' ',manager.first_name, manager.last_name) AS manager,
    t.from_date AS title_from_date,
    de.from_date AS department_from_date
  FROM mv_employees.employee e 
  INNER JOIN mv_employees.title t 
  ON e.id = t.employee_id 
  INNER JOIN mv_employees.salary s 
  ON e.id = s.employee_id 
  INNER JOIN cte_previous_salary c 
  ON e.id = c.employee_id 
  INNER JOIN mv_employees.department_employee de 
  ON e.id = de.employee_id 
  INNER JOIN mv_employees.department d 
  ON de.department_id = d.id 
  INNER JOIN mv_employees.department_manager dm 
  ON d.id = dm.department_id 
  INNER JOIN mv_employees.employee AS manager 
  ON dm.employee_id = manager.id 
  WHERE s.to_date = '9999-01-01' 
  AND t.to_date = '9999-01-01'
  AND de.to_date = '9999-01-01'
  AND dm.to_date = '9999-01-01'
)

SELECT 
  employee_id,
  gender,
  title,
  salary,
  department,
  ROUND(100 * (salary - previous_salary) / previous_salary :: NUMERIC, 2) AS salary_percentage_change,
  DATE_PART('year', now()) - DATE_PART('year', hire_date) as company_tenure_years,
  DATE_PART('year', now()) - DATE_PART('year', title_from_date) as title_tenure_years,
  DATE_PART('year', now()) - DATE_PART('year', department_from_date) as department_tenure_years,
  employee_name,
  manager
FROM cte_joined_data;

-- Generating benchmark views 

DROP VIEW IF EXISTS mv_employees.tenure_benchmark;
CREATE VIEW mv_employees.tenure_benchmark AS 
SELECT 
  company_tenure_years,
  AVG(salary) AS tenure_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY 1;


DROP VIEW IF EXISTS mv_employees.gender_benchmark;
CREATE VIEW mv_employees.gender_benchmark AS 
SELECT 
  gender,
  AVG(salary) AS gender_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY 1;


DROP VIEW IF EXISTS mv_employees.department_benchmark;
CREATE VIEW mv_employees.department_benchmark AS 
SELECT 
  department,
  AVG(salary) AS department_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY 1;


DROP VIEW IF EXISTS mv_employees.title_benchmark;
CREATE VIEW mv_employees.title_benchmark AS 
SELECT 
  title,
  AVG(salary) AS title_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY 1;

-- Historic employee record view based on all the previously generated views 

DROP VIEW IF EXISTS mv_employees.historic_employees_record CASCADE;
CREATE VIEW mv_employees.historic_employee_records AS 
WITH cte_previous_salary AS (
  SELECT
    *
  FROM 
    (SELECT 
      employee_id,
      to_date,
      LAG(amount) OVER (PARTITION BY employee_id ORDER BY from_date) AS amount,
      ROW_NUMBER() OVER (PARTITION BY employee_id ORDER BY to_date DESC) AS record_rank
    FROM mv_employees.salary) all_salaries
  WHERE record_rank = 1
),

cte_joined_data AS (
  SELECT 
    e.id AS employee_id,
    e.birth_date,
    DATE_PART('year', now()) - DATE_PART('year', e.birth_date) AS employee_age,
    CONCAT_WS(' ',e.first_name, e.last_name) AS employee_name,
    e.gender,
    e.hire_date,
    t.title,
    s.amount AS salary,
    c.amount AS previous_latest_salary,
    d.dept_name AS department,
    CONCAT_WS(' ',manager.first_name, manager.last_name) AS manager,
    DATE_PART('year', now()) - DATE_PART('year', e.hire_date) AS company_tenure_years,
    DATE_PART('year', now()) - DATE_PART('year', t.from_date) AS title_tenure_years,
    DATE_PART('year', now()) - DATE_PART('year', de.from_date) AS department_tenure_years,
    DATE_PART('months', AGE(now(),t.from_date)) AS title_tenure_months,
    GREATEST(
      t.from_date,
      s.from_date,
      de.from_date
      ) AS effective_date,
    LEAST(
      t.to_date,
      s.to_date,
      de.to_date
      ) AS expiry_date
  FROM mv_employees.employee e 
  INNER JOIN mv_employees.title t 
  ON e.id = t.employee_id 
  INNER JOIN mv_employees.salary s 
  ON e.id = s.employee_id 
  INNER JOIN cte_previous_salary c 
  ON e.id = c.employee_id 
  INNER JOIN mv_employees.department_employee de 
  ON e.id = de.employee_id 
  INNER JOIN mv_employees.department d 
  ON de.department_id = d.id 
  INNER JOIN mv_employees.department_manager dm 
  ON d.id = dm.department_id 
  INNER JOIN mv_employees.employee AS manager 
  ON dm.employee_id = manager.id  
),

cte_ordered_transactions AS (
  SELECT 
    employee_id,
    birth_date,
    employee_age,
    employee_name,
    gender,
    hire_date,
    title,
    LAG(title) OVER w AS previous_title,
    salary,
    previous_latest_salary,
    LAG(salary) OVER w AS previous_salary,
    department,
    LAG(department) OVER w AS previous_department,
    manager,
    LAG(manager) OVER w AS previous_manager,
    company_tenure_years,
    title_tenure_years,
    title_tenure_months,
    department_tenure_years,
    effective_date,
    expiry_date,
    ROW_NUMBER() OVER (PARTITION BY employee_id ORDER BY effective_date DESC) AS event_order 
  FROM cte_joined_data 
  WHERE effective_date <= expiry_date
  WINDOW w AS (PARTITION BY employee_id ORDER BY effective_date)
),

final_output AS 
(
  SELECT 
    base.employee_id,
    base.gender,
    base.birth_date,
    base.employee_age,
    base.hire_date,
    base.title,
    base.employee_name,
    base.previous_title,
    base.salary,
    previous_latest_salary,
    -- previous salary is based off the LAG records
    base.previous_salary,
    base.department,
    base.previous_department,
    base.manager,
    base.previous_manager,
    -- tenure metrics
    base.company_tenure_years,
    base.title_tenure_years,
    base.title_tenure_months,
    base.department_tenure_years,
    base.event_order,
    -- only include the latest salary change for the first event_order row
    CASE
      WHEN event_order = 1
        THEN ROUND(
          100 * (base.salary - base.previous_latest_salary) /
            base.previous_latest_salary::NUMERIC,
          2
        )
      ELSE NULL
    END AS latest_salary_percentage_change,
    -- event type logic by comparing all of the previous lag records
    CASE
      WHEN base.previous_salary < base.salary
        THEN 'Salary Increase'
      WHEN base.previous_salary > base.salary
        THEN 'Salary Decrease'
      WHEN base.previous_department <> base.department
        THEN 'Dept Transfer'
      WHEN base.previous_manager <> base.manager
        THEN 'Reporting Line Change'
      WHEN base.previous_title <> base.title
        THEN 'Title Change'
      ELSE NULL
    END AS event_name,
    -- salary change
    ROUND(base.salary - base.previous_salary) AS salary_amount_change,
    ROUND(
      100 * (base.salary - base.previous_salary) / base.previous_salary::NUMERIC,
      2
    ) AS salary_percentage_change,
    -- benchmark comparisons - we've omit the aliases for succinctness!
    -- tenure
    ROUND(tenure_benchmark_salary) AS tenure_benchmark_salary,
    ROUND(
      100 * (base.salary - tenure_benchmark_salary)
        / tenure_benchmark_salary::NUMERIC
    ) AS tenure_comparison,
    -- title
    ROUND(title_benchmark_salary) AS title_benchmark_salary,
    ROUND(
      100 * (base.salary - title_benchmark_salary)
        / title_benchmark_salary::NUMERIC
    ) AS title_comparison,
    -- department
    ROUND(department_benchmark_salary) AS department_benchmark_salary,
    ROUND(
      100 * (salary - department_benchmark_salary)
        / department_benchmark_salary::NUMERIC
    ) AS department_comparison,
    -- gender
    ROUND(gender_benchmark_salary) AS gender_benchmark_salary,
    ROUND(
      100 * (base.salary - gender_benchmark_salary)
        / gender_benchmark_salary::NUMERIC
    ) AS gender_comparison,
    -- usually best practice to leave the effective/expiry dates at the end
    base.effective_date,
    base.expiry_date
  FROM cte_ordered_transactions AS base
  INNER JOIN mv_employees.tenure_benchmark tb 
  ON base.company_tenure_years = tb.company_tenure_years
  INNER JOIN mv_employees.title_benchmark tib 
  ON base.title = tib.title 
  INNER JOIN mv_employees.department_benchmark db 
  ON base.department = db.department
  INNER JOIN mv_employees.gender_benchmark gb 
  ON base.gender = gb.gender
)

SELECT * FROM final_output;

DROP VIEW IF EXISTS mv_employees.employee_deep_dive;
CREATE VIEW mv_employees.employee_deep_dive AS 
SELECT 
  *
FROM mv_employees.historic_employee_records
WHERE event_order <= 5;

```
