# People Analytics Case Study Quiz Answers
The following are my solutions to the People Analytics Case Study quiz questions in 
[Danny Ma's Serious SQL course](https://www.datawithdanny.com/ "Data With Danny"). There are three quizzes with a total of 29 questions. I have completed all three quizzes.
<br/>
<br/>
The tables used to solve the questions come from a dataset (`employees`) that was copied as a new schema with materialized views (`mv_employees`) and containing the same indexes as the original dataset. This was because there was an error in the date fields of the original dataset that needed to be corrected prior to data analysis.
<br/>
<br/>
Among other SQL techniques, I employed the following in order to find the required data in the quizzes:
<br/>
- `WHERE` and `ORDER BY` to filter and sort data
- aggregate functions with `GROUP BY` to summarize data
- window functions and a 1/0 flag with `CASE WHEN` to capture specific data about a particular group within a dataset
- `MAX` with `CASE WHEN` to transpose data to a wide format in order to `CONCAT` for a more readable output
- `JOIN` to join a table back onto itself after creating the join criteria in a CTE using `LIMIT`
- window functions in a `SELECT` expression to find percentages
- `DATE_PART` to extract the year from a date type to aggregate data by year
- `PERCENTILE_CONT` to find the median of an ordered list of values
- `ROW_NUMBER` window function to rank an ordered list of values so as to filter by the first value
- `WHERE EXISTS` as an anti-join to filter out rows associated with primary keys that do not match the required criteria
- `LAG` window function to extract data that occurs just prior to a target row

## Table of Contents
1. [Quiz #1](#quiz-1)
2. [Quiz #2](#quiz-2)
3. [Quiz #3](#quiz-3)

## [Quiz #1](#table-of-contents)
> 1. What is the full name of the employee with the highest salary?

```sql
-- Use mv_employees.current_employee_snapshot
SELECT
  employee_name,
  salary
FROM mv_employees.current_employee_snapshot
ORDER BY salary DESC
LIMIT 1;
```
| employee_name  | salary |
|----------------|--------|
| Tokuyasu Pesch | 158220 |

> 2. How many current employees have the equal longest tenure years in their current title?

```sql
SELECT
  title_tenure_years,
  COUNT(DISTINCT employee_id) AS employee_count
FROM mv_employees.current_employee_snapshot
GROUP BY title_tenure_years
ORDER BY title_tenure_years DESC
LIMIT 1;
```
| title_tenure_years | employee_count |
|--------------------|----------------|
| 19                 | 3505           |

> 3. Which department has the least number of current employees? Choices: Production, Sales, Development, Customer Service


```sql
-- Added window SUM to make sure total is 240,124
SELECT *
FROM (
  SELECT
    department,
    COUNT(DISTINCT employee_id) AS employee_count,
    SUM(COUNT(DISTINCT employee_id)) OVER () AS total_employees
  FROM mv_employees.current_employee_snapshot
  GROUP BY department
) subquery
WHERE department IN ('Production', 'Sales', 'Development', 'Customer Service')
ORDER BY employee_count
LIMIT 1;
```
| department       | employee_count | total_employees |
|------------------|----------------|-----------------|
| Customer Service | 17569          | 240124          |


> 4. What is the largest difference between minimimum and maximum salary values for all current employees?

```sql
SELECT
  MIN(salary) AS min_salary,
  MAX(salary) AS max_salary,
  MAX(salary) - MIN(salary) AS biggest_diff
FROM mv_employees.current_employee_snapshot;
```
| min_salary | max_salary | biggest_diff |
|------------|------------|--------------|
| 38623      | 158220     | 119597       |

> 5. How many male employees are above the overall average salary value for the `Production` department? Hint: You might want to use a window function in a CTE first before using a SUM CASE WHEN

```sql
WITH cte_1 AS (
  SELECT *,
    AVG(salary) OVER () AS avg_salary
  FROM mv_employees.current_employee_snapshot
  WHERE department = 'Production'
)
SELECT
  SUM(
    CASE
      WHEN gender = 'M' AND  salary > avg_salary
      THEN 1
      ELSE 0
    END 
  ) AS _flag
FROM cte_1;
```
| _flag |
|-------|
| 14999 |

> 6. Which title has the highest average salary for male employees?

```sql
SELECT
  title,
  AVG(salary) AS avg_salary_title
FROM mv_employees.current_employee_snapshot
WHERE gender = 'M'
GROUP BY title
ORDER BY avg_salary_title DESC
LIMIT 1;
```
| title        | avg_salary_title   |
|--------------|--------------------|
| Senior Staff | 80735.479464575886 |

> 7. Which department has the highest average salary for female employees?

```sql
SELECT
  department,
  AVG(salary) AS avg_salary_dept
FROM mv_employees.current_employee_snapshot
WHERE gender = 'F'
GROUP BY department
ORDER BY avg_salary_dept DESC
LIMIT 1;
```
| department | avg_salary_dept    |
|------------|--------------------|
| Sales      | 88835.963930928729 |

> 8. Which department has the most female employees?

```sql
SELECT
  department,
  COUNT(DISTINCT employee_id) AS female_count
FROM mv_employees.current_employee_snapshot
WHERE gender = 'F'
GROUP BY department
ORDER BY female_count DESC
LIMIT 1;
```
| department  | female_count |
|-------------|--------------|
| Development | 24533        |

> 9. What is the gender ratio in the department which has the highest average male salary and what is the average male salary value rounded to the nearest integer?

```sql
WITH cte_1 AS (
  SELECT
    department,
    avg_salary AS avg_male_salary
  FROM mv_employees.department_level_dashboard
  WHERE gender = 'M'
  ORDER BY avg_male_salary DESC
  LIMIT 1
)
SELECT 
  a.gender,
  a.employee_count,
  a.avg_salary
FROM mv_employees.department_level_dashboard a 
INNER JOIN cte_1 b
  ON a.department = b.department;
```
| gender | employee_count | avg_salary |
|--------|----------------|------------|
| M      | 22702          | 88864      |
| F      | 14999          | 88836      |

I can also continue with the following to make the answer look exactly like the format for Danny's choices:

```sql
-- Save previous query as a table
DROP TABLE IF EXISTS sales_dept;
CREATE TABLE sales_dept AS 
WITH cte_1 AS (
  SELECT
    department,
    avg_salary AS avg_male_salary
  FROM mv_employees.department_level_dashboard
  WHERE gender = 'M'
  ORDER BY avg_male_salary DESC
  LIMIT 1
)
SELECT 
  a.gender,
  a.employee_count,
  a.avg_salary
FROM mv_employees.department_level_dashboard a 
INNER JOIN cte_1 b
  ON a.department = b.department;

-- Put data in wide format to `CONCAT`
WITH cte_1 AS (
  SELECT
    MAX(CASE WHEN gender = 'F'
      THEN ('Female ' || employee_count || ' : ') END) AS female_employee_count,
    MAX(CASE WHEN gender = 'M'
      THEN ('Male ' || employee_count || ' and ') END) AS male_employee_count,
    MAX(CASE WHEN gender = 'M'
      THEN TO_CHAR(avg_salary, '$FM999,999,999') END) AS avg_male_salary
  FROM sales_dept
)
SELECT
  female_employee_count || male_employee_count || avg_male_salary AS _output
FROM cte_1;
```
| _output                               |
|---------------------------------------|
| Female 14999 : Male 22702 and $88,864 |

> 10. HR Analytica want to change the average salary increase percentage value to 2 decimal places - what should the new value be for males for the company level dashboard? Hint: Look at the `mv_employees.company_level_dashboard` view.

```sql
SELECT
  gender,
  ROUND(AVG(salary_percentage_change), 2) AS avg_salary_percentage_change
FROM mv_employees.current_employee_snapshot
GROUP BY gender;
```
| gender | avg_salary_percentage_change |
|--------|------------------------------|
| M      | 3.02                         |
| F      | 3.03                         |

> 11. How many current employees have the equal longest overall time in their current positions (not in years)? Hint: You may want to recalculate the tenure value directly from the `mv_employees.department_employee` table!

I was a little confused by this one. I used the `mv_employees.title` table because that is where I can find current positions. However, none of the answer choices matched mine. I then determined that Danny probably meant to say "time in their current DEPARTMENT", not "positions", hence the clue to use the `department_employee` table. This resulted in a return that matched the correct answer in the quiz.

```sql
SELECT
  AGE(now(), department_employee.from_date) AS department_tenure,
  COUNT(DISTINCT employee_id) AS employee_count,
-- Add SUM window function to make sure the total number of employees is correct. Expect 240124.
  SUM(COUNT(DISTINCT employee_id)) OVER ()  AS total_employees
FROM mv_employees.department_employee
WHERE to_date = '9999-01-01'
GROUP BY department_tenure
ORDER BY department_tenure DESC
LIMIT 1;
```
| department_tenure                                                                                          | employee_count | total_employees |
|------------------------------------------------------------------------------------------------------------|----------------|-----------------|
| { "years": 19, "months": 8, "days": 3, "hours": 14, "minutes": 18, "seconds": 47, "milliseconds": 95.924 } | 9              | 240124          |

## [Quiz #2](#table-of-contents)
> 1. How many employees have left the company? Hint: You may want to use the `expiry_date` and `event_order` columns from the `mv_employees.historic_employee_records` view.
```sql
SELECT
  COUNT(DISTINCT employee_id) AS employee_churn_count
FROM mv_employees.historic_employee_records
WHERE event_order = 1
  AND expiry_date <> '9999-01-01';
```
| employee_churn_count |
|----------------------|
| 59910                |

> 2. What percentage of churn employees were male?
```sql
  SELECT
    gender,
    COUNT(*) AS gender_churn,
    SUM(COUNT(*)) OVER () AS total_churn,
    ROUND(100 * COUNT(*) / SUM(COUNT(*)) OVER ()::NUMERIC) AS churn_perc
  FROM mv_employees.historic_employee_records
  WHERE event_order = 1
    AND expiry_date <> '9999-01-01'
  GROUP BY gender
  ```
| gender | gender_churn | total_churn | churn_perc |
|--------|--------------|-------------|------------|
| M      | 35864        | 59910       | 60         |
| F      | 24046        | 59910       | 40         |

I could also put the above in a CTE and just capture the data for males:
```sql
WITH cte_1 AS (
  SELECT
    gender,
    COUNT(*) AS gender_churn,
    SUM(COUNT(*)) OVER () AS total_churn,
    ROUND(100 * COUNT(*) / SUM(COUNT(*)) OVER ()::NUMERIC) AS churn_perc
  FROM mv_employees.historic_employee_records
  WHERE event_order = 1
    AND expiry_date <> '9999-01-01'
  GROUP BY gender
)
SELECT *
FROM cte_1
WHERE gender = 'M';
```
| gender | gender_churn | total_churn | churn_perc |
|--------|--------------|-------------|------------|
| M      | 35864        | 59910       | 60         |

> 3. Which title had the most churn?
```sql
SELECT
  title,
  COUNT(DISTINCT employee_id) AS employee_churn_count
FROM mv_employees.historic_employee_records
WHERE event_order = 1
  AND expiry_date <> '9999-01-01'
GROUP BY title
ORDER BY employee_churn_count DESC
LIMIT 1;
```
| title    | employee_churn_count |
|----------|----------------------|
| Engineer | 16320                |

> 4. Which department had the most churn?
```sql
SELECT
  department,
  COUNT(DISTINCT employee_id) AS employee_churn_count
FROM mv_employees.historic_employee_records
WHERE event_order = 1
  AND expiry_date <> '9999-01-01'
GROUP BY department
ORDER BY employee_churn_count DESC
LIMIT 1;
```
| department  | employee_churn_count |
|-------------|----------------------|
| Development | 15578                |

> 5. Which year had the most churn?
```sql
SELECT
  DATE_PART('year', expiry_date) AS churn_year,
  COUNT(DISTINCT employee_id) AS employee_churn_count
FROM mv_employees.historic_employee_records
WHERE event_order = 1
  AND expiry_date <> '9999-01-01'
GROUP BY churn_year
ORDER BY employee_churn_count DESC
LIMIT 1;
```
| churn_year | employee_churn_count |
|------------|----------------------|
| 2018       | 7610                 |

> 6. What was the average salary for each employee who has left the company rounded to the nearest integer?
```sql
SELECT
  ROUND(AVG(salary)) AS avg_salary
FROM mv_employees.historic_employee_records
WHERE event_order = 1
  AND expiry_date <> '9999-01-01';
```
| avg_salary |
|------------|
| 61577      |

> 7. What was the median total company tenure for each churn employee just bfore they left?
```sql
SELECT
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY company_tenure_years)) AS median_comp_tenure
FROM mv_employees.historic_employee_records
WHERE event_order = 1
  AND expiry_date <> '9999-01-01';
```
| median_comp_tenure |
|--------------------|
| 15                 |

> 8. On average, how many different titles did each churn employee hold rounded to 1 decimal place?
```sql
WITH cte_1 AS (
  SELECT
    employee_id
  FROM mv_employees.historic_employee_records
  WHERE event_order = 1
    AND expiry_date <> '9999-01-01'
),
cte_2 AS (
  SELECT
    SUM 
      (CASE
        WHEN event_name = 'Title Change' THEN 1
        ELSE 0
      END) + 1 AS title_change_count
  FROM mv_employees.historic_employee_records a 
  INNER JOIN cte_1
    ON a.employee_id = cte_1.employee_id
  GROUP BY cte_1.employee_id
)
SELECT 
  ROUND(AVG(title_change_count), 1) AS avg_title_count_churn
FROM cte_2;
```
| avg_title_count_churn |
|-----------------------|
| 1.2                   |

> 9. What was the average last pay increase for churn employees?
```sql
WITH cte_1 AS (
  SELECT
    employee_id
  FROM mv_employees.historic_employee_records
  WHERE event_order = 1
    AND expiry_date <> '9999-01-01'
),
cte_2 AS (
  SELECT
    employee_id,
    salary_amount_change,
    ROW_NUMBER() OVER (
      PARTITION BY employee_id
      ORDER BY event_order
    ) AS _rank
  FROM mv_employees.historic_employee_records a 
  WHERE EXISTS (
    SELECT 1
    FROM cte_1
    WHERE a.employee_id = cte_1.employee_id
  )
  AND event_name = 'Salary Increase'
)
SELECT
  ROUND(AVG(salary_amount_change)) AS avg_last_pay_inc_churn
FROM cte_2
WHERE _rank = 1;
```
| avg_last_pay_inc_churn |
|------------------------|
| 2250                   |

> 10. What percentage of churn employees had a pay decrease event in their last 5 events?
```sql
WITH cte_1 AS (
  SELECT
    employee_id
  FROM mv_employees.historic_employee_records
  WHERE event_order = 1
    AND expiry_date <> '9999-01-01'
),
cte_2 AS (
  SELECT
    COUNT(DISTINCT employee_id) AS churn_w_dec_event
  FROM mv_employees.historic_employee_records a 
  WHERE EXISTS (
    SELECT 1
    FROM cte_1
    WHERE a.employee_id = cte_1.employee_id
  )
  AND event_order <= 5
  AND event_name = 'Salary Decrease'
) 
SELECT
  COUNT(DISTINCT a.employee_id) AS total_churn,
  b.churn_w_dec_event,
  ROUND(100 * b.churn_w_dec_event / COUNT(DISTINCT a.employee_id)::NUMERIC) AS _percent
FROM cte_1 a
CROSS JOIN cte_2 b
GROUP BY b.churn_w_dec_event;
```
| total_churn | churn_w_dec_event | _percent |
|-------------|-------------------|----------|
| 59910       | 14328             | 24       |

## [Quiz #3](#table-of-contents)
> 1. How many managers are there currently in the company?
```sql
SELECT
  COUNT(DISTINCT employee_id) AS manager_count
FROM mv_employees.current_employee_snapshot
WHERE title = 'Manager';
```
| manager_count |
|---------------|
| 9             |

> 2. How many employees have ever been a manager?
```sql
SELECT
  COUNT(title) AS historical_management_count
FROM mv_employees.title
WHERE title = 'Manager';
```
| historical_management_count |
|-----------------------------|
| 24                          |

> 3. On average - how long did it take for an employee to first become a manager from their the date they were originally hired in days?
```sql
WITH cte_1 AS (
  SELECT
    t1.id AS employee_id,
    t2.title,
    t1.hire_date,
    t2.from_date,
    t2.from_date - t1.hire_date AS _age 
  FROM mv_employees.employee AS t1 
  INNER JOIN mv_employees.title AS t2 
    ON t1.id = t2.employee_id
  WHERE t2.title = 'Manager'
)
SELECT
  ROUND(AVG(_age)) AS avg_days_to_manager
FROM cte_1;
```
| avg_days_to_manager |
|---------------------|
| 909                 |

> 4. What was the most common titles that managers had just before before they became a manager?
```sql
WITH cte_1 AS (
  SELECT *,
    LAG(title) OVER (
      PARTITION BY employee_id
      ORDER BY from_date) AS previous_title
  FROM mv_employees.title
)
SELECT
  previous_title,
  COUNT(*) AS previous_title_count
FROM cte_1
WHERE title = 'Manager'
  AND previous_title IS NOT NULL
GROUP BY previous_title
ORDER BY previous_title_count DESC
LIMIT 1;
```
| previous_title | previous_title_count |
|----------------|----------------------|
| Senior Staff   | 7                    |

> 5. How many managers were first hired by the company as a manager?
```sql
WITH cte_1 AS (
  SELECT *,
    LAG(title) OVER (
      PARTITION BY employee_id
      ORDER BY from_date) AS previous_title
  FROM mv_employees.title
)
SELECT
  COUNT(*) AS hired_as_manager
FROM cte_1
WHERE title = 'Manager'
  AND previous_title IS NULL
GROUP BY previous_title;
```
| hired_as_manager |
|------------------|
| 9                |

> 6. On average - how much more do current managers make on average compared to all other employees rounded to the nearest dollar?

I was stuck on this one attempting to use the `mv_employees.tenure_benchmark` table. I learned this lesson: DO NOT try to average the averages!
```sql
WITH salary_calc AS (
  SELECT
    AVG(salary) AS avg_salary_not_manager
  FROM mv_employees.current_employee_snapshot
  WHERE title <> 'Manager'
)
SELECT
  AVG(t1.salary) AS avg_salary_manager,
  t2.avg_salary_not_manager,
  ROUND(AVG(t1.salary) - t2.avg_salary_not_manager) AS _difference
FROM mv_employees.current_employee_snapshot AS t1 
CROSS JOIN salary_calc AS t2
WHERE t1.title = 'Manager'
GROUP BY t2.avg_salary_not_manager;
```
| avg_salary_manager | avg_salary_not_manager | _difference |
|--------------------|------------------------|-------------|
| 77723.666666666667 | 72012.021781229827     | 5712        |

> 7. Which current manager has the most employees in their department?
```sql
WITH employee_count_table AS (
  SELECT
    department,
    COUNT(*) AS employee_count,
-- Included window SUM to verify total employee count of 240124.
    SUM(COUNT(*)) OVER () AS total_employee_count
  FROM mv_employees.current_employee_snapshot
  GROUP BY department 
  ORDER BY employee_count DESC
  LIMIT 1
)
SELECT
  employee_count_table.department,
  employee_count_table.employee_count,
  mv_employees.employee.first_name || ' ' || mv_employees.employee.last_name AS manager_name
FROM employee_count_table
INNER JOIN mv_employees.department
  ON employee_count_table.department = mv_employees.department.dept_name
INNER JOIN mv_employees.department_manager
  ON mv_employees.department.id = mv_employees.department_manager.department_id
INNER JOIN mv_employees.employee
  ON mv_employees.department_manager.employee_id = mv_employees.employee.id
WHERE mv_employees.department_manager.to_date = '9999-01-01';
```
| department  | employee_count | manager_name  |
|-------------|----------------|---------------|
| Development | 61386          | Leon DasSarma |

I realized later that `manager` was already joined on to this table, resulting in a much simpler query. There were two versions of this table and I was looking at the wrong one to begin with. This teaches me to know my tables!
```sql
SELECT 
  manager,
  COUNT(*) AS employee_count
FROM mv_employees.current_employee_snapshot
GROUP BY manager
ORDER BY employee_count DESC
LIMIT 1;
```
| manager       | employee_count |
|---------------|----------------|
| Leon DasSarma | 61386          |

> 8. What is the difference in employee count between the 3rd and 4th ranking departments by size?
```sql
WITH rank_table AS (
  SELECT
    department,
    COUNT(*) AS employee_count,
    ROW_NUMBER() OVER (
      ORDER BY COUNT(*) DESC
    )
  FROM mv_employees.current_employee_snapshot
  GROUP BY department
)
SELECT
  MAX(CASE WHEN row_number = 3 THEN employee_count ELSE NULL END) -
  MAX(CASE WHEN row_number = 4 THEN employee_count ELSE NULL END) AS _diff
FROM rank_table;
```
| _diff |
|-------|
| 20132 |
