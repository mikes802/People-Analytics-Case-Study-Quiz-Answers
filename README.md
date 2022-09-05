# People Analytics Case Study Quiz Answers
The following are my solutions to the People Analytics Case Study quiz questions in 
[Danny Ma's Serious SQL course](https://www.datawithdanny.com/ "Data With Danny"). There are three quizzes in total; I have completed Quiz #1 so far, which has eleven questions.
<br/>
<br/>
The tables used to solve the questions come from a dataset (`employees`) that was copied as a new schema with materialized views (`mv_employees`) and containing the same indexes as the original dataset. This was because there was an error in the date fields of the original dataset that needed to be corrected prior to data analysis.
<br/>
<br/>
For Quiz #1, among other SQL techniques, I employed the following in order to find the required data:
<br/>
- `WHERE` and `ORDER BY` to filter and sort data
- aggregate functions with `GROUP BY` to summarize data
- window functions and a 1/0 flag with `CASE WHEN` to capture specific data about a particular group within a dataset
- `MAX` with `CASE WHEN` to transpose data to a wide format in order to `CONCAT` for a more readable output
- `JOIN` to join a table back onto itself after creating the join criteria in a CTE using `LIMIT`

## Quiz #1
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
