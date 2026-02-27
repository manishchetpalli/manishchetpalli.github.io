# SQL

## **What is a database?**

Database is structured, organised set of data.Think of it as a filecabinet whre you store data in different sections called tables.

---

## **What is DBMS?**

A software which allows users to interact with data. Stores data in structured format. A schema defines the structure of data.

---

## **Acid Properties**

- Atomicity:
Each transaction is either properly carried out or the database reverts back to the state before the transaction has started.Atomicity enforces the "all or nothing" principle, ensuring that an entire transaction either completes successfully or is rolled back, preventing partial, inconsistent states. In a financial transfer, if a crash occurs after deducting funds from one account but before adding them to another, an atomic system would detect the failure before the final COMMIT and roll back the entire transaction.

- Consistency:
Database must be in a consistent state before and after the transaction.Consistency guarantees that a transaction moves the database from one valid state to another, enforcing rules (like checking for sufficient funds) to prevent inconsistent results.

- Isolation:
Multiple transactions occur independently without interference.Isolation is the "no interference policy," meaning multiple concurrent transactions operate independently without seeing each other's uncommitted changes; Delta achieves this using. Optimistic Concurrency Control (OCC).

- Durability:
Successful transactions are persisted even in case of failure.Durability ensures that once a transaction is committed, it permanently remains recorded in the system, even in the event of crashes or power outages. This is achieved by logging the start of the operation and the final COMMIT status in a transaction log (ledger); if the system restarts and the log shows no COMMIT, the changes are rolled back

---

## The WHERE Clause
SQL filters data row by row. For every row in a table, the SQL engine evaluates the condition in the `WHERE` clause; if the result is true, the row is kept in the output; if false, it is removed.

> --- **Comparison Operators**

These are used to compare two expressions (columns, static values, functions, or subqueries).

`=` (Equal): Checks if two values are exactly the same.

`!=` or `<>` (Not Equal): Filters out rows that match a specific value.

`>` / `<` (Greater/Less than): Standard numerical or alphabetical comparisons.

`>=` / `<=` (Greater/Less than or equal to): Includes the boundary value in the results.

> --- **Logical Operators**

Used to combine multiple conditions into a single filter.

`AND`: Highly restrictive. Every single condition must be true for the row to be included.

`OR`: Relaxed. A row is included if at least one of the conditions is true.

`NOT`: A reverse operator that switches "true" to "false" and vice versa, excluding matching values.

> --- **Membership & Range Operators**

`BETWEEN`: Checks if a value falls within a specific range.

`IN`: Checks if a value exists within a defined list.

!!! Tip
    Use `IN` instead of multiple `OR` statements. It is cleaner, easier to extend, and offers better performance.

> --- **Search Operator (LIKE)**

Used to search for patterns within text using wildcards.

`%` (Percentage): Represents any number of characters (zero, one, or many).

       `'M%'`: Starts with M.
       `'%n'`: Ends with n.
       `'%r%'`: Contains r anywhere.

`_` (Underscore): Represents exactly one character.
       `'__R%'`: Matches a string where 'R' is specifically in the third position.

---

## **JOINS**

> --- **The Four Basic Join Types**

| Join Type | Result | Order of Tables Matters? |
| :--- | :--- | :--- |
| INNER JOIN | Returns only matching rows from both tables. | No; A join B is the same as B join A. |
| LEFT JOIN | Returns all rows from the left table and matching rows from the right. | Yes; The first table mentioned is the "primary" source. |
| RIGHT JOIN | Returns all rows from the right table and matching rows from the left. | Yes; The second table is the "primary" source. |
| FULL JOIN | Returns all rows from both tables, matching or not. | No; Both tables are equally important. |

!!! Note
    In `LEFT`, `RIGHT`, and `FULL` joins, if there is no matching data, SQL will display NULL for the missing values.

> --- **Anti-Joins: The "Non-Existence" Check**

Anti-joins are used to find rows in one table that do not have a corresponding match in another. These are essential for identifying "strange cases" or missing data.

Left Anti-Join: Returns rows from the left table only if they have no match on the right.
       Logic: `LEFT JOIN` + `WHERE [right_table_key] IS NULL`.

Right Anti-Join: Returns rows from the right table that have no match on the left.
       Logic: `RIGHT JOIN` + `WHERE [left_table_key] IS NULL`.

Full Anti-Join: Returns all unmatching rows from both sides (the opposite of an Inner Join).
       Logic: `FULL JOIN` + `WHERE [left_key] IS NULL OR [right_key] IS NULL`.

> --- **The Cross Join (Cartesian Product)**

A Cross Join combines every row from the first table with every row from the second table.

Key Distinction: Unlike all other joins, it does not require an `ON` condition because it doesn't care about matching data.

The Math: The total number of rows is the product of both tables (e.g., 5 customers × 4 orders = 20 rows).

Use Cases: Generating test data, simulations, or finding all possible combinations (like every product matched with every available color).

> --- **The Join Decision Tree**

When deciding how to combine tables, follow this logic based on your goal:

To see ONLY matching data: Use an Inner Join.

To see EVERYTHING (Don’t want to miss any data):
       If one table is more important (a "Master" table): Use a Left Join.
       If all tables are equally important: Use a Full Join.

To see ONLY unmatching data (Checkups):
       From only one specific side: Use a Left Anti-Join.
       From both sides: Use a Full Anti-Join.

## **Set Operators**

SQL Joins: Combine tables side-by-side by merging columns, resulting in a wider table.

Set Operators: Combine tables by appending rows underneath each other, resulting in a longer table.

Key Requirement: While joins require a key column (like an ID), set operators require the queries to have the same column structure.

> --- **The Four Set Operators**

`UNION`: Returns all distinct rows from both queries. It automatically removes duplicates.

`UNION ALL`: Returns all rows from both queries, including duplicates.

!!! Tip
    `UNION ALL` is much faster than `UNION` because it doesn't perform the extra step of checking for and removing duplicates.

`EXCEPT` (or `MINUS`): Returns distinct rows from the first query that are not found in the second query.

`INTERSECT`: Returns only the rows that are common to both queries.

> --- **Expert Tips & Interview Strategies**

Choose `UNION ALL` by Default: If you know your data has no duplicates or if you are performing a quality check to find duplicates, use `UNION ALL` for better performance.

The "Order" Trap: In an interview, remember that `EXCEPT` is the only operator where the order of tables matters. Switching the order will yield completely different results.

Avoid `SELECT `: For production scripts, explicitly list all column names. Using `` is "quick and dirty" and can break your query if the table schema changes in the future.

Identify the Source: When combining data from multiple tables (like `Orders` and `Orders_Archive`), create a static column (e.g., `'Archive' AS SourceTable`) so users know where each row originated.

> --- **Real-World Use Cases (To impress interviewers)**

Data Migration Quality: Use `EXCEPT` in both directions between two databases to ensure they are identical. If both results are empty, the migration was 100% complete.

Finding the "Delta": Data engineers use `EXCEPT` to compare today's data with yesterday's to identify only the new records that need to be loaded.

Centralizing "Person" Data: Instead of writing four separate reports for Employees, Customers, Suppliers, and Students, use a `UNION` to combine them into one master "Persons" table first. This ensures consistent logic across your analysis.

---

## **SQL String**

> --- **Data Manipulation Functions**

These functions are used to change the format or appearance of string values.

`CONCAT`: Combines multiple string values or columns into a single value.
!!! Tip
    Always remember to add a manual separator (like a space `' '` or underscore `'_'`) between columns, or the data will be squashed together.

`UPPER` & `LOWER`: Converts text to all capital letters or all lowercase letters.
       
`TRIM`: Removes "evil" leading and trailing white spaces from a string.
!!! Tip
    Spaces are often invisible. If your data isn't matching as expected, use `TRIM` to clean up the "mess".

`REPLACE`: Swaps a specific character or substring with something new. To remove a character instead of replacing it, specify an empty string (`''`) as the new value.
       

`LENGTH` (or `LEN`): Counts the total number of characters in a value, including spaces, special characters, and digits.


`LEFT`: Extracts a specific number of characters starting from the beginning (left side) of the string.

`RIGHT`: Extracts a specific number of characters starting from the end (right side) of the string.

`SUBSTRING`: Extracts a part of a string from any specified position. Needs three arguments: the value, the starting position, and the number of characters (length) to extract.

> --- **Important Interview Tips & Expert Logic**

Detecting Hidden Spaces: A classic interview task is finding records with leading or trailing spaces. The speaker suggests two methods:
    1.  Logical Comparison: Use `WHERE column != TRIM(column)`. If the trimmed version isn't equal to the original, spaces were present.
    2.  Length Difference: Use `WHERE LENGTH(column) != LENGTH(TRIM(column))`. A difference in length confirms hidden characters.

Nested Functions: Demonstrate expertise by combining functions. For example, if you need the first two characters of a name but the data has leading spaces, use `LEFT(TRIM(first_name), 2)`.

Dynamic Substrings: When you want to extract "everything after a certain point" but the total length of the strings varies (like removing the first character of every name), use `LENGTH` as the third argument in your `SUBSTRING`.
    The Trick: Providing a length that is longer than the actual string (e.g., `SUBSTRING(name, 2, LENGTH(name))`) ensures you capture all remaining characters without getting an error.

Data Cleansing Awareness: Emphasize that functions like `TRIM` are used for data cleansing to ensure accuracy in analysis and reporting.

## **Date & Time**

> --- **Core Concepts: Date vs. Time vs. DateTime**

In SQL, date and time information is structured into a hierarchy (Year > Month > Day > Hour > Minute > Second).

Date: Contains only year, month, and day (e.g., 2025-08-20).

Time: Contains only hours, minutes, and seconds.

DateTime / Timestamp: Combines both date and time into one structure.

Sources of Dates: You can query dates from database columns, hardcoded strings (static values), or the `GETDATE()` function, which returns the current system date and time.

> --- **Part Extraction Functions (The "Big Three")**

These are the most common functions for retrieving specific components of a date.

`DAY()`, `MONTH()`, `YEAR()`: Quick functions that accept one parameter (the date) and return the corresponding part as an integer.

`DATEPART(part, date)`: A more powerful function that can extract "hidden" information like weeks, quarters, or hours as integers.

`DATENAME(part, date)`: Similar to `DATEPART`, but returns the name of the part (e.g., "August" or "Wednesday") as a string.

> --- **Advanced Manipulation Functions**

`DATETRUNC(part, date)`: This function "resets" or truncates a date to a specific level of granularity. For example, truncating to the `MONTH` level resets the day to 01 and all time components to 00:00:00. It always returns a DateTime value.

`EOMONTH(date)`: Returns the last day of the month for a given date. It returns a Date data type.
   The "First Day of Month" Trick: There is no "Start of Month" function, but you can simulate it by using `DATETRUNC(month, date)`, which resets the day to the 1st.


> --- **Important Interview Tips & Best Practices**

Performance Optimization: When filtering data in a `WHERE` clause, always use numeric functions (`MONTH`, `YEAR`, `DATEPART`) instead of string functions (`DATENAME`). Searching for integers is significantly faster for the database than searching for text strings.
   Decision-Making Tree:
       Need Day, Month, or Year as a number? Use `DAY()`, `MONTH()`, or `YEAR()`.
       Need a full name (like "January") for a report? Use `DATENAME()`.
       Need a specific part like Week or Quarter? Use `DATEPART()`.

Reporting vs. Analysis: Use `DATENAME` to make reports more "human-readable" (e.g., "Sales for January" looks better than "Sales for Month 1").

Aggregation Strategy: Use `DATETRUNC` to quickly "zoom in or zoom out" of data during analysis. It allows you to group thousands of precise timestamps into manageable buckets like "Sales by Month" or "Sales by Year".

Validation: Use the `ISDATE()` function to test if a string is a valid date that SQL can understand before performing calculations.

> --- **Casting vs. Formatting**

Formatting: Changing how a value looks (e.g., changing 2025-01-01 to "Jan 25"). It always returns the data as a string.

Casting: Changing the actual data type of a value (e.g., turning the string "123" into an integer 123, or a `DateTime` into a `Date`).

| Function | Best Use Case | Can Format? | Can Cast? |
| :--- | :--- | :--- | :--- |
| `CAST` | Simple, readable data type changes. | No (Always returns standard format). | Yes (Any type to any type). |
| `CONVERT` | Casting while applying specific "Style" codes. | Yes (Using predefined style numbers). | Yes (Any type to any type). |
| `FORMAT` | Custom, complex, or culture-specific looks. | Yes (Full flexibility with specifiers). | Limited (Only to a String). |

When using the `FORMAT` function, these specifiers are case-sensitive.

Years: `yyyy` (4 digits), `yy` (2 digits).

Months: `MM` (2 digits), `MMM` (Abbreviation like "Jan"), `MMMM` (Full name like "January").

Days: `dd` (2 digits), `ddd` (Short name like "Mon"), `dddd` (Full name like "Monday").

Time: `HH` (24-hour), `hh` (12-hour), `mm` (minutes), `ss` (seconds), `tt` (AM/PM).
   

> --- **Formatting Numbers**

The `FORMAT` function is unique because it can also style numeric values.

`'N'` (Numeric): Adds commas for thousands separation.

`'C'` (Currency): Adds a currency symbol (like `$`) based on the culture.

`'P'` (Percentage): Multiplies by 100 and adds the `%` sign.

> --- **DATEADD: Manipulating Dates**

The `DATEADD` function allows you to add or subtract specific time intervals (years, months, days) to or from a date.

Syntax: `DATEADD(part, interval, date)`.
    Part: What you want to change (e.g., `year`, `month`, `day`).
    Interval: A number representing how much to add. Use positive numbers to move forward in time and negative numbers to subtract.
       Date: The value you are manipulating.


> --- **DATEDIFF: Calculating Differences**

`DATEDIFF` (Date Difference) finds the amount of time passed between two specific dates and returns the result as an integer.

Syntax: `DATEDIFF(part, start_date, end_date)`.

Logic: It subtracts the start date from the end date based on the specified part (years, months, or days).

Common Use Cases for Interviews:
    Calculating Age: Use `DATEDIFF(year, birth_date, GETDATE())` to find the number of years between a birthday and the current system date.
    Shipping Duration: Calculate how many days passed between an order being placed and being shipped using `DATEDIFF(day, order_date, ship_date)`.
    Time Gap Analysis: Finding the number of days between the current order and the previous order by combining `DATEDIFF` with the `LAG` window function.

> --- **ISDATE: Data Validation & Quality**

`ISDATE` is a validation function that checks if a string or numeric value is a valid date.

Return Values: Returns 1 (True) if the value is a valid date and 0 (False) if it is not.

Standard Formats: SQL generally only understands "standard" database formats; non-standard strings (like `day/month/year` in some contexts) may return 0 even if they look like dates to a human.


> --- **Important Interview Tips & Professional Strategies**

The "Casting Failure" Trap: A common interview scenario involves a query failing because you tried to `CAST` a string column to a `DATE` type, but the column contained "corrupt" or bad data.

Force-Casting with Logic: To avoid query errors when dealing with bad data quality, the speaker recommends using `CASE WHEN` combined with `ISDATE`:
    Strategy: `CASE WHEN ISDATE(col) = 1 THEN CAST(col AS DATE) ELSE NULL END`. This forces SQL to only attempt the conversion on valid dates, while turning "bad" data into `NULL` instead of crashing the query.

Identifying Issues: You can quickly find data quality problems by filtering for all rows where `ISDATE(column) = 0`.

Using Dummy Values: Instead of `NULL`, you can use a "dummy" or very large date in your `ELSE` statement to make errors easy to spot in a report.

Analytical Power: Emphasize that these functions are not just for display; they are critical for data analytics, such as calculating average shipping durations per month or finding business-critical "time gaps" between customer actions.

## **NULL**

A NULL represents nothing, unknown, or missing data. NULL is not equal to zero, an empty string, or a blank space.


> --- **ISNULL vs. COALESCE (Replacing NULLs)**

`ISNULL(value, replacement)`: Replaces a NULL with a specified default value or another column. It is limited to only two arguments.

`COALESCE(val1, val2, ... valN)`: Returns the first non-null value from a list of multiple values. It checks values from left to right and stops as soon as it finds data.

!!! Tip
    Always prefer `COALESCE` over `ISNULL`. `COALESCE` is a standard SQL function that works across all databases (Oracle, MySQL, PostgreSQL), whereas `ISNULL` is specific to Microsoft SQL Server. Additionally, `COALESCE` is more powerful because it handles more than two values.

`NULLIF(val1, val2)` (Creating NULLs): Returns NULL if the two values are equal; otherwise, it returns the first value.

!!! Tip
    Use `NULLIF` to prevent the common "Divide by Zero" error by replacing a `0` divisor with `NULL` (e.g., `sales / NULLIF(quantity, 0)`).

`IS NULL` / `IS NOT NULL` (Checking for NULLs): These are comparison keywords that return a Boolean (True/False).



> --- **Critical Interview Tips & Use Cases**

1. Aggregations and NULLs
    
    The Trap: Standard aggregate functions (`AVG`, `SUM`, `MIN`, `MAX`) totally ignore NULL values. 
    
    Exception: `COUNT()` counts rows, so it includes NULLs, while `COUNT(column_name)` ignores them.
    
    Solution: If the business considers a NULL to be zero, you must handle the NULL (e.g., `COALESCE(score, 0)`) before running the aggregation to get an accurate average.

2. Mathematical & String Operations
   
    The Rule: Any number plus NULL equals NULL. Similarly, concatenating a string with a NULL results in a NULL.
    
    Interview Advice: Before merging first and last names or adding bonus points to a score, use `COALESCE` to replace NULLs with an empty string (`''`) or a `0` to avoid losing the entire result.

2. Handling NULLs in Joins
   
    The Problem: SQL cannot map or match NULL keys during a join. If a joining key contains a NULL, those records will be lost in the output, leading to inaccurate results.
    
    The Fix: Use `ISNULL` or `COALESCE` within the `ON` clause to replace NULL keys with a temporary default value (like an empty string) so SQL can successfully match the rows.
> --- **Detection Strategy: "Don't Trust Your Eyes"**

It is nearly impossible to distinguish between a NULL, an empty string, and a blank space just by looking at query results. 

!!! Tip
    Always use the `DATALENGTH()` (or `LENGTH()`) function to identify the true value.
       NULL: Returns a length of NULL.
       Empty String: Returns a length of 0.
       Blank Space: Returns a length of 1 or more, depending on the number of spaces.

| Feature | NULL | Empty String | Blank Space |
| :--- | :--- | :--- | :--- |
| Storage | Best (consumes the least) | Wastes space | Wastes space |
| Performance| Fastest for queries | Fast | Slowest (Worst option) |
| Searching | Must use `IS NULL` | Use `=` operator | Use `=` operator |

## **CASE WHEN**

The `CASE` statement is SQL’s way of applying conditional logic (similar to "if-then" in programming) to evaluate conditions and return specific values.

The Syntax: 
    `CASE`: Indicates the start of the logic.
    `WHEN [condition] THEN [result]`: Defines the criteria and the value to show if true.
    `ELSE [default]`: Optional. Provides a value if all other conditions fail.
    `END`: Mandatory. Signals the completion of the logic.

The "NULL" Trap: If you skip the `ELSE` clause and none of your conditions are met, SQL will automatically return a `NULL`.

> --- **Top 4 Use Cases for Interviews**

The primary purpose of `CASE` is data transformation—creating new information from existing data without changing the source database.

1.  Categorization: Grouping data into buckets for better reporting (e.g., classifying sales as "High," "Medium," or "Low").

2.  Value Mapping: Translating technical codes or abbreviations into human-readable text (e.g., turning `0` into "Inactive" or `M` into "Male").

3.  Handling NULLs: Replacing `NULL` values with usable data (like `0`) to ensure calculations like `AVG` or `SUM` are accurate for the business.

4.  Conditional Aggregation: Flagging specific rows with a `1` or `0` based on a condition, then using `SUM` to count only those specific records (e.g., "Count only orders greater than $30").

## **WINDOW**

> --- **Window Functions vs. GROUP BY**

The most critical distinction for an interview is how these two handle data granularity:

`GROUP BY`: "Smashes" or "squeezes" multiple rows into a single summary row, causing you to lose the level of detail for individual records.

Window Functions: Perform calculations (like sums or averages) on a subset of data without losing the detail of individual rows.

Key Advantage: If you have 10 input rows, you get 10 output rows, allowing you to show both raw data (like Order ID) and aggregate data (like Total Sales) in the same query.


> --- **The Three Components of the `OVER` Clause**

The `OVER` clause tells SQL you are using a window function. It contains three optional parts:

`PARTITION BY`: Divides the data into groups or "windows" (e.g., by Product ID). If left empty, SQL treats the entire dataset as one single window.

`ORDER BY`: Sorts the data within each window. While optional for aggregate functions, it is mandatory for Ranking and Value functions.

`FRAME` Clause: Defines a specific scope of rows within a window to be involved in a calculation (e.g., "the current row and the two following rows").


| Group | Functions | Key Interview Detail |
| :--- | :--- | :--- |
| Aggregate | `SUM`, `COUNT`, `AVG`, `MIN`, `MAX` | `COUNT` accepts any data type; others require numeric data. |
| Ranking | `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE` | Used to assign a rank; they cannot take an expression (must be empty) except for `NTILE`. |
| Value/Analytics | `LEAD`, `LAG`, `FIRST_VALUE`, `LAST_VALUE` | Used to access specific values from other rows (e.g., the previous order). |

> --- **Mandatory Rules & Limitations**

Placement: You are only allowed to use window functions in the `SELECT` and `ORDER BY` clauses.

No Filtering: You cannot use a window function in the `WHERE` or `GROUP BY` clauses.

No Nesting: You cannot put one window function inside another (e.g., `SUM(RANK() OVER(...))`).

Execution Order: SQL executes the `WHERE` clause first, then calculates window functions on the filtered data.

Mixing with `GROUP BY`: You can use both in the same query only if the window function uses columns that are also present in the `GROUP BY` clause.

> --- **Important Tip**

The Default Frame Trap: Be careful when using `ORDER BY` with aggregate functions. SQL applies a hidden default frame (`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`), which creates a "running total" rather than a total for the whole window.

The Power of Partitioning: Use `PARTITION BY` to perform "row-level calculations." For example, you can calculate the total sales for a specific product and display it next to every individual order for that product.

Analytical Strategy: When faced with a complex task (like ranking customers by their total sales), Baraa recommends building the `GROUP BY` query first to get the totals, and then adding the window function (the rank) as the final step.

Flexibility: Window functions allow you to show different levels of aggregation in a single query—such as the individual sale, the total sales for that product, and the total sales for the entire company—all in one row.

Frame Boundaries: Remember that the lower boundary must always come before the higher boundary in the `ROWS BETWEEN` syntax.

