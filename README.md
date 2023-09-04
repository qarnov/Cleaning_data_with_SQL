# Data Cleaning With SQL

## Abstarct

An organinzation's attendance tracking software produces duplicates and needs to be cleaned and analyzed for monthly individual and organizational insights 

- Access Dataset [HERE](https://docs.google.com/spreadsheets/d/1oyj43AGT2smaDe5n_e-IrJa0Z1PcC51Ic98w6yyuorM/edit?usp=sharing)

## ERD

![Employee_Attendance (1)](https://user-images.githubusercontent.com/112409778/221864935-8c7df479-dc82-4227-aa49-93812fa5ecc8.png)

## Variables

There are several variables within the original dataset that had an effect on the outcome of the calculation of an employee's attendance percentage

- **Employee_Name**: Employee name
- **Employee ID**: Employee's identification number
- **Date**: Date of the Absence
- **Week Number**: Week number of the year 
- **Department Name**: The department that employee works in
- **Absence Type**: Reason why the employee was absent on a given date
- **Hours**: The number of hours requested off by employee
- **Employee_Status**: Active or Inactive

## SQL Syntax

### **Step 1: Creation of TEMP TABLE with 'clean' data**

~~~ SQL 
/*Temp Table Creation to perfrom mulitple queries*/
CREATE TEMP TABLE t1 AS -- Creation of TEMP TABLE 
SELECT 
  employee_name, Employee_ID, 
  EXTRACT(MONTH FROM date) AS Month, -- Creating Month column
  Absence_Type, department_name,
    
    CASE -- CASE for Campus
    WHEN department_name = 'Primary School' THEN 'PS'
    WHEN department_name = 'Middle School' THEN 'MS'
    WHEN department_name = 'High School' THEN 'HS'
    WHEN department_name = 'Intermediate School' THEN 'IS'
    ELSE 'HO' END AS Campus,

    CASE -- CASE for School Days
    WHEN EXTRACT(MONTH FROM date) = 8 THEN 8
    WHEN EXTRACT(MONTH FROM date) = 9 THEN 20 
    WHEN EXTRACT(MONTH FROM date) = 10 THEN 20
    WHEN EXTRACT(MONTH FROM date) = 11 THEN 17 
    WHEN EXTRACT(MONTH FROM date) = 12 THEN 17 
    WHEN EXTRACT(MONTH FROM date) = 1 THEN 19 
    END AS School_Days,

    COUNT(CASE WHEN absence_type IN ('Sick','Personal','Absent','Personal_(Longevity)','Unpaid_Time') THEN 'Unexcused' END) AS Unexcused, -- Counting Unexcused Absences
    COUNT(CASE WHEN absence_type IN ('Vacation','Holiday', 'Covid_Sick', 'Religious_Observation', 'Bereavement','Professional_Development', 'Administrative_Leave' ,'Worker Compensation') THEN 'Excused' END ) AS Excused, -- Counting Excused Absences 
  Hours, 

    CASE -- CASE Statement for half or whole day
    WHEN Hours = 4 THEN .5
    ELSE 1 END Day
 FROM `my-data-project-36654.FA_Attendance_Data.Attendance_Data`
 WHERE Hours <> 8.75 AND Employee_Name <> 'ADMIN FA' AND  Employee_Name <> 'Academic Support' -- Filtering out the duplicate absence and irrelevant names 
 GROUP BY employee_name, Employee_ID,Absence_Type, department_name, Month, School_Days, Hours ;
 ~~~

### **Step 2: Calculating Individual Attendance from TEMP TABLE *t1***

~~~ SQL 
/*Individual Attendance*/

SELECT
  sub.employee_name,sub.campus,sub.month,
  (sub.school_days - sub.excused_abs) - sub.unexcused_Abs AS Days_Present,-- Difference between possible days and unexcused absences  
  sub.school_days - sub.excused_abs AS Possible_Days, -- Difference between school days and excused absences
  ROUND(((sub.school_days - sub.excused_abs) - sub.unexcused_Abs)/ (sub.school_days - sub.excused_abs),2) AS Percentage, -- quotient of Days_Present and Possible_Days
  sub.school_days

FROM 
 (SELECT 
  DISTINCT t1.employee_name, 
  t1.Month, 
  t1.Campus,
  SUM(t1.Excused * t1.Day) OVER (PARTITION BY t1.employee_name, t1.Month) AS Excused_Abs, -- Calculating Total Excused Absences with Day Type
  SUM(t1.Unexcused * t1.Day) OVER (PARTITION BY t1.employee_name, t1.Month) AS Unexcused_Abs, -- Calculating Total Unexcused Absences with Day Type
  t1.School_Days
 FROM t1) AS sub -- Subquery
 WHERE ROUND(((sub.school_days - sub.excused_abs) - sub.unexcused_Abs)/ (sub.school_days - sub.excused_abs),2) > 0
 ORDER BY sub.Employee_name, sub.Month;
 ~~~
 
 - Access Query Result [HERE](https://docs.google.com/spreadsheets/d/1oyj43AGT2smaDe5n_e-IrJa0Z1PcC51Ic98w6yyuorM/edit?usp=sharing) - Indivdual Attendance

### **Step 3: Calculating Campus Attendance from TEMP TABLE *t1***
~~~ SQL 
/*Campus Attendance*/

SELECT 
 DISTINCT  sub1.Campus, sub1.Month, 
  ROUND(AVG(sub1.Percentage) OVER (PARTITION BY sub1.Campus, sub1.Month),2) AS Campus_Percentage -- AVG Attendance of Campus
FROM 
(SELECT
  sub.employee_name,sub.campus,sub.month,
  (sub.school_days - sub.excused_abs) - sub.unexcused_Abs AS Days_Present,-- Difference between possible days and unexcused absences  
  sub.school_days - sub.excused_abs AS Possible_Days, -- Difference between school days and excused absences
  ROUND(((sub.school_days - sub.excused_abs) - sub.unexcused_Abs)/ (sub.school_days - sub.excused_abs),2) AS Percentage, -- quotient of Days_Present and Possible_Days
  sub.school_days

FROM 
 (SELECT 
  DISTINCT t1.employee_name, 
  t1.Month, 
  t1.Campus,
  SUM(t1.Excused * t1.Day) OVER (PARTITION BY t1.employee_name, t1.Month) AS Excused_Abs, -- Calculating Total Excused Absences with Day Type
  SUM(t1.Unexcused * t1.Day) OVER (PARTITION BY t1.employee_name, t1.Month) AS Unexcused_Abs, -- Calculating Total Unexcused Absences with Day Type
  t1.School_Days
 FROM t1) AS sub -- Subquery
 WHERE ROUND(((sub.school_days - sub.excused_abs) - sub.unexcused_Abs)/ (sub.school_days - sub.excused_abs),2) > 0
 ORDER BY sub.Employee_name, sub.Month) AS sub1
 ORDER BY sub1.Campus, sub1.Month;
 
 ~~~
 
- Access Query Result [HERE](https://docs.google.com/spreadsheets/d/1oyj43AGT2smaDe5n_e-IrJa0Z1PcC51Ic98w6yyuorM/edit?usp=sharing)  - Campus Attendance

### **Step 4: Calculating Weighted Average of Individual Attendance from TEMP TABLE *t1*** 

~~~ SQL

 /*Individual Attendance Summative*/ -- Weighted Average of Attendance
 
 SELECT 
  sub1.employee_name, sub1.Campus,
  (sub1.Cum_School_Days-sub1.Cum_Excused_Abs) - sub1.Cum_Unexcused_Abs AS Days_present,
  sub1.Cum_School_Days-sub1.Cum_Excused_Abs AS Possible_Days,
  sub1.Cum_School_Days,
  ROUND(((sub1.Cum_School_Days-sub1.Cum_Excused_Abs) - sub1.Cum_Unexcused_Abs)/ (sub1.Cum_School_Days-sub1.Cum_Excused_Abs),2) AS Weighted_Avg_Percentage -- Days_Present/Possible_Days
 FROM 
 (SELECT 
  DISTINCT sub.employee_name,sub.Campus,
  SUM(sub.Excused_Abs) OVER (PARTITION BY sub.employee_name,sub.Campus) AS Cum_Excused_Abs, --Cumlative Excused Absences
  SUM(sub.Unexcused_Abs) OVER (PARTITION BY sub.employee_name,sub.Campus) AS Cum_Unexcused_Abs, --Cumlative Unexcused Absences
  SUM(sub.School_days) OVER (PARTITION BY sub.employee_name,sub.Campus) AS Cum_School_Days -- Culative School Days
 FROM
 (SELECT 
  DISTINCT t1.employee_name, 
  t1.Month, 
  t1.Campus,
  SUM(t1.Excused * t1.Day) OVER (PARTITION BY t1.employee_name, t1.Month) AS Excused_Abs, -- Calculating Total Excused Absences with Day Type
  SUM(t1.Unexcused * t1.Day) OVER (PARTITION BY t1.employee_name, t1.Month) AS Unexcused_Abs, -- Calculating Total Unexcused Absences with Day Type
  t1.School_Days
 FROM t1) AS sub) AS sub1
 ORDER BY sub1.Employee_name, sub1.Campus
 
 ~~~
 - Access Query Results [HERE](https://docs.google.com/spreadsheets/d/1oyj43AGT2smaDe5n_e-IrJa0Z1PcC51Ic98w6yyuorM/edit?usp=sharing) - Weighted Average
 
 
