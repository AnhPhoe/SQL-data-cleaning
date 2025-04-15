# SQL-data-cleaning

This is an educational project on data cleaning and preparation using SQL. The original database in CSV format is located in the file club_member_info.csv. Here, we will explore the steps that need to be applied to obtain a cleansed version of the dataset.

## Data Introduction
```SQL
SELECT *
FROM club_member_info_cleaned cmic
LIMIT 10;
```
The result:
|full_name|age|martial_status|email|phone|full_address|job_title|membership_date|
|---------|---|--------------|-----|-----|------------|---------|---------------|
|addie lush|40|married|alush0@shutterfly.com|254-389-8708|3226 Eastlawn Pass,Temple,Texas|Assistant Professor|7/31/2013|
|      ROCK CRADICK|46|married|rcradick1@newsvine.com|910-566-2007|4 Harbort Avenue,Fayetteville,North Carolina|Programmer III|5/27/2018|
|Sydel Sharvell|46|divorced|ssharvell2@amazon.co.jp|702-187-8715|4 School Place,Las Vegas,Nevada|Budget/Accounting Analyst I|10/6/2017|
|Constantin de la cruz|35||co3@bloglines.com|402-688-7162|6 Monument Crossing,Omaha,Nebraska|Desktop Support Technician|10/20/2015|
|  Gaylor Redhole|38|married|gredhole4@japanpost.jp|917-394-6001|88 Cherokee Pass,New York City,New York|Legal Assistant|5/29/2019|
|Wanda del mar       |44|single|wkunzel5@slideshare.net|937-467-6942|10864 Buhler Plaza,Hamilton,Ohio|Human Resources Assistant IV|3/24/2015|
|Joann Kenealy|41|married|jkenealy6@bloomberg.com|513-726-9885|733 Hagan Parkway,Cincinnati,Ohio|Accountant IV|4/17/2013|
|   Joete Cudiff|51|divorced|jcudiff7@ycombinator.com|616-617-0965|975 Dwight Plaza,Grand Rapids,Michigan|Research Nurse|11/16/2014|
|mendie alexandrescu|46|single|malexandrescu8@state.gov|504-918-4753|34 Delladonna Terrace,New Orleans,Louisiana|Systems Administrator III|3/12/1921|
| fey kloss|52|married|fkloss9@godaddy.com|808-177-0318|8976 Jackson Park,Honolulu,Hawaii|Chemical Engineer|11/5/2014|
## Copy table
### Create a new table for cleaning
```SQL
-- club_member_info definition
CREATE TABLE club_member_info_cleaned (
	full_name VARCHAR(50),
	age INTEGER,
	martial_status VARCHAR(50),
	email VARCHAR(50),
	phone VARCHAR(50),
	full_address VARCHAR(50),
	job_title VARCHAR(50),
	membership_date VARCHAR(50)
);
```
### Copy all value from the original table
```SQL
INSERT INTO club_member_info_cleaned
SELECT * FROM club_member_info;
```
### Check duplicate value
```SQL
SELECT full_name, COUNT(*)
FROM club_member_info_cleaned
GROUP BY full_name
HAVING COUNT(*) > 1;
```
|full_name|COUNT(*)|
|---------|--------|
|ERWIN HUXTER|3|
|GARRICK REGLAR|2|
|GEORGES PREWETT|2|
|HASKELL BRADEN|2|
|MADDIE MORRALLEE|2|
|NICKI FILLISKIRK|2|
|OBED MACCAUGHEN|2|
|SEYMOUR LAMBLE|2|
|TAMQRAH DUNKERSLEY|2|

After that, duplicate value will be removed.
```SQL
DELETE FROM club_member_info_cleaned
WHERE ROWID NOT IN (
    SELECT MIN(ROWID)
    FROM club_member_info_cleaned
    GROUP BY full_name
);
```
The result: The table is from 2010 rows to 2000 rows that removed 10 rows duplicate 
## Cleaning data
#### full_name column
The column will contain all upper case names with no extra whitespace at the beginning or end.
```SQL
UPDATE club_member_info_cleaned SET full_name = TRIM(full_name);
UPDATE club_member_info_cleaned SET full_name = UPPER(full_name);
```
#### age column
- Truncation: First, the age column will be truncated to the first two characters.
- Imputation: Then, any age values that are 0 will be replaced with the average age of members with valid ages.
- Ceiling: Finally, all age values will be rounded up to the nearest whole number.
```SQL
UPDATE club_member_info_cleaned SET age = SUBSTRING(age, 1, 2);
UPDATE club_member_info_cleaned SET age = (SELECT AVG(age) FROM club_member_info_cleaned WHERE age > 0)
WHERE age = 0;
UPDATE club_member_info_cleaned SET age = CEILING(age);
```
#### martial_status
- The first statement adds a new column called martial_id to the table, which will store numerical values.
- The second statement if the corresponding martial_status is 'married', 'single', 'divorced', or 'divored' which input 1 and all other martial status values input 0.
```SQL
ALTER TABLE club_member_info_cleaned
ADD martial_id FLOAT (2010);
UPDATE club_member_info_cleaned 
SET martial_id = CASE WHEN martial_status IN ('married', 'single', 'divorced', 'divored') THEN 1
ELSE 0 
END;
```
- The purpose of the martial_id column is to calculate the most frequent martial_status.
- If any value has the highest result, it will be taken as the result to replace the blank row.
```SQL
UPDATE club_member_info_cleaned SET martial_status = (SELECT martial_status FROM (SELECT martial_status, COUNT(martial_id) AS count_status
FROM club_member_info_cleaned
GROUP BY martial_status
ORDER BY count_status DESC LIMIT 1) AS most_frequent_status)
WHERE martial_status = "";
```
#### phone column
- Replaces all empty phone number entries in the club_member_info_cleaned table with the placeholder phone number '999-999-9999'.
```SQL
UPDATE club_member_info_cleaned SET phone = '999-999-9999'
WHERE phone ="";
```
#### job_title
- The first statement adds a new column called job_id to the table, which will store numerical values.
- The second statement if the corresponding text which inputs 1 and blank values input 0.
```SQL
ALTER TABLE club_member_info_cleaned
ADD job_id FLOAT (50);
UPDATE club_member_info_cleaned SET job_id  = CASE WHEN job_title = "" THEN 0
ELSE 1 
END;
```
- The purpose of the job_id column is to calculate the most frequent job_title.
- If any value has the highest result, it will be taken as the result to replace the blank row.
```SQL
UPDATE club_member_info_cleaned SET job_title = (SELECT job_title FROM (SELECT job_title, COUNT(job_id) AS count_job
FROM club_member_info_cleaned
WHERE job_title <> ""
GROUP BY job_title 
ORDER BY count_job DESC LIMIT 1) AS most_frequent_job)
WHERE job_title = "";
```
#### membership_date
- If the year portion of the membership_date appears to be in the 1900s. If it is, it assumes that the year was incorrectly recorded and corrects it to the 2000s by replacing the "19" with "20".
```SQL
UPDATE club_member_info_cleaned SET membership_date = CASE WHEN SUBSTR(membership_date, -6+2) LIKE '19%' THEN CONCAT (SUBSTR(membership_date, 1, LENGTH(membership_date) -4),'20',
SUBSTR (membership_date, -2))
ELSE membership_date
END;
```
### Consequence
Data is cleaned - club_member_info_cleaned
```SQL
SELECT * FROM club_member_info_cleaned cmic LIMIT 10;
```
|full_name|age|martial_status|email|phone|full_address|job_title|membership_date|martial_id|job_id|
|---------|---|--------------|-----|-----|------------|---------|---------------|----------|------|
|ADDIE LUSH|40|married|alush0@shutterfly.com|254-389-8708|3226 Eastlawn Pass,Temple,Texas|Assistant Professor|7/31/2013|1.0|1.0|
|ROCK CRADICK|46|married|rcradick1@newsvine.com|910-566-2007|4 Harbort Avenue,Fayetteville,North Carolina|Programmer III|5/27/2018|1.0|1.0|
|SYDEL SHARVELL|46|divorced|ssharvell2@amazon.co.jp|702-187-8715|4 School Place,Las Vegas,Nevada|Budget/Accounting Analyst I|10/6/2017|1.0|1.0|
|CONSTANTIN DE LA CRUZ|35|married|co3@bloglines.com|402-688-7162|6 Monument Crossing,Omaha,Nebraska|Desktop Support Technician|10/20/2015|0.0|1.0|
|GAYLOR REDHOLE|38|married|gredhole4@japanpost.jp|917-394-6001|88 Cherokee Pass,New York City,New York|Legal Assistant|5/29/2019|1.0|1.0|
|WANDA DEL MAR|44|single|wkunzel5@slideshare.net|937-467-6942|10864 Buhler Plaza,Hamilton,Ohio|Human Resources Assistant IV|3/24/2015|1.0|1.0|
|JOANN KENEALY|41|married|jkenealy6@bloomberg.com|513-726-9885|733 Hagan Parkway,Cincinnati,Ohio|Accountant IV|4/17/2013|1.0|1.0|
|JOETE CUDIFF|51|divorced|jcudiff7@ycombinator.com|616-617-0965|975 Dwight Plaza,Grand Rapids,Michigan|Research Nurse|11/16/2014|1.0|1.0|
|MENDIE ALEXANDRESCU|46|single|malexandrescu8@state.gov|504-918-4753|34 Delladonna Terrace,New Orleans,Louisiana|Systems Administrator III|3/12/2021|1.0|1.0|
|FEY KLOSS|52|married|fkloss9@godaddy.com|808-177-0318|8976 Jackson Park,Honolulu,Hawaii|Chemical Engineer|11/5/2014|1.0|1.0|




