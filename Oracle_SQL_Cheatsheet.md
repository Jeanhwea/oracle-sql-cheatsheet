# Oracle SQL Cheatsheet

## Table of contents

1. ETL statements
2. Union operations
3. Window functions
4. DB management utilities
5. Special query clauses
6. Data types
7. Global constants in Oracle SQL

## 1. ETL statements

Chapter contents:

1. `CREATE TABLE` vs `CREATE VIEW`
2. `INSERT INTO`
3. `UPDATE`
4. `DELETE`

### 1.1. `CREATE TABLE` vs `CREATE VIEW`
#### `CREATE TABLE` Statement
The `CREATE TABLE` statement is used to create a new table in a database.

##### Copying the structure from another table

```sql
-- In SAS
CREATE TABLE table_name LIKE source_table;

-- In Oracle SQL
CREATE TABLE table_name AS
SELECT * FROM source_table
WHERE 1=0; -- Never true, therefore yields no rows
```
> NOTE: When executing the Oracle SQL version of this query, no indexes or constraints will be selected.

##### Oracle SQL - Complex `CREATE TABLE` Example
When you issue the following statement, you create a table named `admin_emp` in the `hr` schema and store it in the `admin_tbs` tablespace:

```sql
CREATE TABLE hr.admin_emp (
         empno      NUMBER(5) PRIMARY KEY,
         ename      VARCHAR2(15) NOT NULL,
         ssn        NUMBER(9) ENCRYPT,
         job        VARCHAR2(10),
         mgr        NUMBER(5),
         hiredate   DATE DEFAULT (sysdate),
         photo      BLOB,
         sal        NUMBER(7,2),
         hrly_rate  NUMBER(7,2) GENERATED ALWAYS AS (sal/2080),
         comm       NUMBER(7,2),
         deptno     NUMBER(3) NOT NULL
                     CONSTRAINT admin_dept_fkey REFERENCES hr.departments
                     (department_id))
   TABLESPACE admin_tbs
   STORAGE (INITIAL 50K);

COMMENT ON TABLE hr.admin_emp IS 'Enhanced employee table';
```
**Note the following about this example:**
- Integrity constraints are defined on several columns of the table.

- The `STORAGE` clause specifies the size of the first extent.

- Encryption is defined on one column (`ssn`), through the transparent data encryption feature of Oracle Database. The Oracle Wallet must therefore be open for this `CREATE TABLE` statement to succeed.

- The `photo` column is of data type `BLOB`, which is a member of the set of data types called large objects (LOBs). LOBs are used to store semi-structured data (such as an XML tree) and unstructured data (such as the stream of bits in a color image).

- One column is defined as a virtual column (`hrly_rate`). This column computes the employee's hourly rate as the yearly salary divided by 2,080.

- A `COMMENT` statement is used to store a comment for the table. You query the `*_TAB_COMMENTS` data dictionary views to retrieve such comments.

#### CREATE VIEW Statement
In SQL, a view is a virtual table based on the result-set of an SQL statement.

A view contains rows and columns, just like a real table. The fields in a view are fields from one or more real tables in the database.

You can add SQL functions, `WHERE`, and `JOIN` statements to a view and present the data as if the data were coming from one single table.

###### CREATE VIEW Syntax

```sql
CREATE [OR REPLACE] VIEW view_name AS -- Use OR REPLACE to update the view
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```
> **NOTE:** A view always shows up-to-date data! The database engine recreates the data, using the view's SQL statement, every time a user queries a view.

### 1.2. `INSERT INTO`
The `INSERT INTO` statement is used to insert new records in a table.

#### `INSERT INTO` Syntax
It is possible to write the `INSERT INTO` statement in **3 main ways**:

1. The first way **specifies** both the **column names** and the **values** to be inserted:

    ```sql
    INSERT INTO table_name (column1, column2, column3, ...)
    VALUES (value1, value2, value3, ...);

    -- SAS alternative version:
    INSERT INTO table_name
    SET
        column1 = value1,
        column2 = value2,
        column3 = value3,
        ...
    ;
    ```

2. If you are adding values for **all the columns** of the table, you do not need to specify the column names in the SQL query. However, make sure the **order of the values** is in the **same order as the columns** in the table. The `INSERT INTO` syntax would be as follows:

    ```sql
    INSERT INTO table_name
    VALUES (value1, value2, value3, ...);
    ```

3. You may also insert **values from other tables**:

    ```sql
    INSERT INTO table_name
    SELECT
        expression1, expression2, expression3, ...
    FROM source_tables
    [WHERE conditions]
    ```

### 1.3. `UPDATE`
The SQL `UPDATE` statement is used to update existing records in the tables.

#### `UPDATE` Syntax
It is possible to write the `UPDATE` statement in **2 main ways**:

1. Setting hardcoded values, analogous to the 1st and 2nd ways to insert rows (`INSERT INTO` statement):

    ```sql
    UPDATE table
    SET
        column1 = expression1,
        column2 = expression2,
        ...
    [WHERE conditions];
    ```

2. Updating a table with data from another table (_correlated update_):

    ```sql
    UPDATE table1 t1
    SET (name, description) = (
        SELECT t2.name, t2.description
        FROM table2 t2 -- this is t2's scope!!
        WHERE t1.id = t2.id
    )
    -- t2 can't be accessed here, but t1 can
    /* Failing to include WHERE EXISTS clause will update non-matching values with nulls */

    /* This version is the easiest to understand */
    WHERE EXISTS (
        SELECT 1
        FROM table2 t2
        WHERE t1.id = t2.id -- AND t1.col2 = t2.col2 AND t1.col3 = t2.col3 ...
    )

    /* This one also works, but is a bit harder to grasp: */

    WHERE t1.id IN (
        SELECT DISTINCT t2.id
        FROM table2 t2
        -- WHERE t1.col2 = t2.col2 and t1.col3 = t2.col3
    )
    ;
    ```

### 1.4. `DELETE`
The Oracle DELETE statement is used to delete a single record or multiple records from a table in Oracle.

#### Example - Using `EXISTS` Clause

You can also perform more complicated deletes.

You may wish to delete records in one table based on values in another table. Since **you can't list more than one table** in the Oracle `FROM` clause **when you are performing a `DELETE`**, you can use the Oracle `EXISTS` clause.

For example:

```sql
DELETE FROM suppliers t1

-- Just like with UPDATE statement:
-- Option 1:
WHERE EXISTS (
    SELECT 1
    FROM customers t2
    WHERE t1.supplier_id = t2.customer_id
    AND t2.customer_id > 25
)
-- Option 2:
WHERE t1.supplier_id IN (
    SELECT DISTINCT t2.customer_id
    FROM customers t2
    WHERE t2.customer_id > 25
)
;
```

## 2. Union operations

Chapter contents:

1. `UNION`
2. `INTERSECT`
3. `MINUS`

### Common basic requirements for union functions

- Same number of columns
- Similar data types
- Same order of columns

### 2.1. `UNION [ALL]`

`UNION` operator returns only distinct rows that appear in either result. `ALL` operator does not eliminate duplicate rows.

### 2.2. `INTERSECT`

Yields only those rows returned by both queries.

### 2.3. `MINUS`

Yields only unique rows returned by the first query but not the second.

### Query example for union operations

```sql
CREATE TABLE final_table AS
SELECT * FROM table_1
{UNION [ALL]|INTERSECT|MINUS}
SELECT * from table_2;
```

## 3. Window functions

A window function performs a calculation across a set of table rows that are somehow related to the current row. This is comparable to the type of calculation that can be done with an aggregate function. But, unlike regular aggregate functions, use of a window function does not cause the rows to become grouped into a single output row; **the rows retain their separate identities**. Behind the scenes, the window function is able to access more than just the current row of the query result.

### General example and overview

```sql
SELECT
    AGGREGATION(var1)
    OVER -- Designates a window function
        (
            /* PARTITION BY narrows the window from the entire dataset
            to individual groups within the dataset */
            [PARTITION BY partition_var]
            /* ORDER BY creates a running total. Without it, each value
            will be an aggregation of var1 within its respective partition_var
            group (akin to a remerged GROUP BY) */
            [ORDER BY order_var]
        ) -- Window specification
        /* The ORDER and PARTITION define what is referred to as the "window" -
        the ordered subset of data over which calculations are made */
FROM ...
;
```

### Window function possibilities

| Aggregation Function | Description    |
| :------------- | :------------- |
| `SUM()`, `COUNT()`, `AVG() `      | Pretty straightforward |
| `ROW_NUMBER()`   | Displays the number of a given row. It starts at 1 and numbers the rows according to the `ORDER BY` clause of the window specification  |
| `RANK()`  | Like `ROW_NUMBER()`, but repeating the resulting value for identical rows. Resumes next row's value where `ROW_NUMBER()` would, if it is not identical.  |
| `DENSE_RANK()`  | Like `RANK()`, but instead of resuming where `ROW_NUMBER()` would, it does at last returned value + 1.  |
| `NTILE(n_groups)`  | Whatever percentile or other subdivision (as set by `n_groups`). For 2 records, it would just create 1st and 2nd percentile, instead of 1 and 100. Keep in mind to use small bands when working with `NTILE()` over small windows.  |
| `LAG ( expression [, offset [, default] ] )`  | **Returns values from a previous row in the table**. Creates null values at the beginning of the dataset. `expression`: An expression that can contain other built-in functions, but can not contain any analytic functions. `offset`: It is the physical offset from the current row in the table. If this parameter is omitted, the default is 1. `default`: It is the value that is returned if the offset goes out of the bounds of the table. If this parameter is omitted, the default is null. |
| `LEAD ( expression [, offset [, default] ] )`  | Same as `LAG()`, but with next rows in the table. Creates null values at the end of the dataset. |

## 4. DB Management utilities

Chapter content:

1. `ALTER TABLE`
2. Constraints
3. Indexes

### 4.1. `ALTER TABLE`

The ALTER TABLE statement, as its name already indicates, allows adding, deleting or modifying columns/constraints in a table after it has been created, among other utilities.

```sql
-- Adding multiple columns
ALTER TABLE customers
ADD (
    customer_name varchar2(45),
    city varchar2(40) DEFAULT 'Seattle'
);

-- Dropping a column
ALTER TABLE table_name
DROP COLUMN column_name;

-- Modifying multiple columns
ALTER TABLE customers
MODIFY (
    customer_name varchar2(100) NOT NULL,
    city varchar2(75) DEFAULT 'Seattle' NOT NULL
);

-- Renaming a column
ALTER TABLE customers
  RENAME COLUMN customer_name TO cname;

-- Renaming a table
ALTER TABLE table_name
  RENAME TO new_table_name;
```

### 4.2. Constraints
SQL constraints are used to specify **rules** for data in a table. If there is any violation between the constraint and the data action, the action is aborted. Constraints can be specified when the table is created with the `CREATE TABLE` statement, or after the table is created with the `ALTER TABLE` statement.

The following constraints are commonly used in SQL:
- `NOT NULL`: Ensures that a column **cannot** have a `NULL` **value**
- `UNIQUE`: Ensures that all **values** in a column are **different**
- `PRIMARY KEY`: A **combination** of a `NOT NULL` and `UNIQUE`. **Uniquely** identifies **each row** in a table
- `FOREIGN KEY`: **Uniquely** identifies a row/record in **another table**
- `CHECK`: Ensures that all values in a column satisfies a **specific condition**
- `DEFAULT`: Sets a **default value** for a column when no value is specified

Constraints may exist in either of these 4 states:



> **Modifying constraints:** There is no way of modifying a constraint. In order to modify it, you will need to drop it and re-create it.

**What is the difference between a `UNIQUE` constraint and a `PRIMARY KEY`?**

| Primary Key	 | Unique Constraint     |
| :------------- | :------------- |
| None of the fields that are part of the primary key can contain a null value.       | 	Some of the fields that are part of the unique constraint can contain null values as long as the combination of values is unique.       |

#### `NOT NULL`

##### Create `NOT NULL` constraint

By default, a column can hold `NULL`. If you do not want to allow `NULL` value in a column, you will want to place the `NOT NULL` constraint on this column. The `NOT NULL` constraint specifies that `NULL` is not an allowable value.

```sql
-- In CREATE TABLE statement:
CREATE TABLE Persons (
    ID int NOT NULL,
    LastName varchar(255) NOT NULL,
    FirstName varchar(255) NOT NULL,
    Age int NOT NULL
);
-- In ALTER TABLE statement:
ALTER TABLE customers
MODIFY (
    ID int NOT NULL, -- Column type still has to be declared
    LastName varchar(255) NOT NULL,
    FirstName varchar(255) NOT NULL,
    Age int NOT NULL
);
```

##### Drop (alter) `NOT NULL` constraint

The `NOT NULL` constraint may not be dropped, since a variable will always be either `NULL` or `NOT NULL`. Therefore, this characteristic may only be switched through an `ALTER TABLE` statement:

```sql
ALTER TABLE customers
MODIFY (
    -- Either NULL or NOT NULL must be specified in order to alter the constraint
    -- Otherwise, the previous constraint will prevail
    ID int NULL,
    LastName varchar(255) NULL,
    FirstName varchar(255) NULL,
    Age int NULL
);
```

#### `UNIQUE`
A unique constraint is a single field or combination of fields that uniquely defines a record. Some of the fields can contain null values as long as the combination of values is unique. **Multiple UNIQUE constraints can be defined on a table**.

##### Create `UNIQUE` constraint

```sql
-- In CREATE TABLE statement (single column(s), no constraint name):
CREATE TABLE table_name
(
    column1 datatype [UNIQUE],
    column2 datatype [UNIQUE],
    ...
    /* If we set the UNIQUE constraint separately on both columns, it will
    evaluate value duplicity ON A COLUMN BY COLUMN BASIS, i.e. if we insert
    values (1, 'A') and (1, 'B'), it will raise an error because there are two
    1's in column1, even though the combination of values is unique. */
);

-- In CREATE TABLE statement (single/multiple columns, with constraint name):
CREATE TABLE table_name
(
    column1 datatype,
    column2 datatype,
    ...
    CONSTRAINT constraint_name UNIQUE (uc_col1 [, uc_col2, ... uc_col_n])
);

-- In ALTER TABLE statement (single column(s), no constraint name):
ALTER TABLE table_name
MODIFY (column1 datatype [UNIQUE], column2 datatype [UNIQUE], ...);
/* Same reasoning as CREATE TABLE applies */

-- In ALTER TABLE statement (single/multiple columns, with constraint name):
ALTER TABLE table_name
ADD CONSTRAINT constraint_name UNIQUE (uc_col1 [, uc_col2, ... uc_col_n]);
```

##### Drop `UNIQUE` constraint

```sql
-- In ALTER TABLE statement (single column(s), no constraint name):
ALTER TABLE table_name
DROP UNIQUE(column1)
DROP UNIQUE(column2)
...;

-- In ALTER TABLE statement (single/multiple columns, with constraint name):
ALTER TABLE table_name
DROP CONSTRAINT constraint_name;
```

#### `PRIMARY KEY`
In Oracle, a primary key is a single field or combination of fields that **uniquely defines a record**. None of the fields that are part of the primary key can contain a null value. A table can have **only one primary key**.

##### Create `PRIMARY KEY` constraint

```sql
-- In CREATE TABLE statement (single column, no constraint name):
CREATE TABLE table_name
(
    id datatype PRIMARY KEY,
    column1 datatype [ NULL | NOT NULL ],
    ...
);

-- In CREATE TABLE statement (single/multiple columns, with constraint name):
CREATE TABLE table_name
(
    column1 datatype,
    column2 datatype,
    ...
    CONSTRAINT constraint_name PRIMARY KEY (column1 [, column2, ... column_n])
);

-- In ALTER TABLE statement (single column, no constraint name):
ALTER TABLE table_name
MODIFY (id datatype PRIMARY KEY);

-- In ALTER TABLE statement (single/multiple column/s, with constraint name):
ALTER TABLE table_name
ADD CONSTRAINT constraint_name PRIMARY KEY (column1 [, column2, ... column_n]);
```

##### Drop `PRIMARY KEY` constraint

```sql
-- In ALTER TABLE statement (single/multiple column/s, no constraint name):
ALTER TABLE table_name
DELETE PRIMARY KEY;

-- In ALTER TABLE statement (single/multiple column/s, with constraint name):
ALTER TABLE table_name
DROP CONSTRAINT constraint_name;
```

#### `FOREIGN KEY`
A foreign key is a column (or columns) that references a column (most often the primary key) of another table. The purpose of the foreign key is to ensure **referential integrity** of the data. In other words, only **values that are supposed to appear** in the database are permitted.

For example, say we have two tables, a `CUSTOMER` table that includes all customer data, and an `ORDERS` table that includes all customer orders. Business logic requires that all orders must be associated with a customer that is already in the `CUSTOMER` table. To enforce this logic, we place a foreign key on the `ORDERS` table and have it reference the primary key of the `CUSTOMER` table. This way, we can ensure that all orders in the `ORDERS` table are related to a customer in the `CUSTOMER` table. In other words, the `ORDERS` table cannot contain information on a customer that is not in the `CUSTOMER` table.

##### Create `FOREIGN KEY` constraint

```sql
-- In CREATE TABLE statement (single column, no constraint name):
CREATE TABLE ORDERS (
    Order_ID int PRIMARY KEY,
    Order_Date date,
    Customer_SID int REFERENCES CUSTOMER(SID), -- FOREIGN KEY constraint
    -- Single-column FOREIGN KEY == Single-column reference
    /* In this case, since the constraint declaration is in-line, no custom constraint name has been generated. Therefore, Oracle generates a default constraint name. This is considered bad practice, since in case you need to modify the constraint, you will need to find out its name from among a list of other constraints in the USER_CONSTRAINTS table */
    Amount double
);

-- In CREATE TABLE statement (single/multiple column/s, with constraint name):
CREATE TABLE table_name
(
  column1 datatype,
  column2 datatype,
  ...

  CONSTRAINT constraint_name
    FOREIGN KEY (column1 [, column2, ... column_n])
    REFERENCES parent_table(p_column1 [, p_column2, ... p_column_n])
);

-- In ALTER TABLE statement (single column, no constraint name):
ALTER TABLE ORDERS
MODIFY (Customer_SID int REFERENCES CUSTOMER(SID));

-- In ALTER TABLE statement (single/multiple column/s, with constraint name):
ALTER TABLE table_name
ADD CONSTRAINT constraint_name
    FOREIGN KEY (column1 [, column2, ... column_n])
    REFERENCES parent_table(p_column1 [, p_column2, ... p_column_n]);
```

##### Drop `FOREIGN KEY` constraint

In order to drop a `FOREIGN KEY`, the generic `ALTER TABLE ... DROP CONSTRAINT` script must be run:

```sql
ALTER TABLE new_table_name
DROP CONSTRAINT constraint_name;
```

#### `CHECK`
##### Create `CHECK` constraint
##### Drop `CHECK` constraint

#### `DEFAULT`
##### Create `DEFAULT` constraint
##### Drop `DEFAULT` constraint

TODO: Enable / Disable / Validate / Novalidate constraints. Syntax & effects.

## 5. Special query clauses

Chapter contents:

1. `WITH` clause
2. `LIKE` clause
3. Hints

### 5.1. `WITH` clause

The `WITH` clause may be processed as an **inline view** or resolved as a **temporary table**. The advantage of the latter is that repeated references to the subquery may be more efficient as the data is easily retrieved from the temporary table, rather than being requeried by each reference. You should assess the performance implications of the WITH clause on a case-by-case basis.

#### `WITH` clause example
So that we don't need to redefine the same subquery multiple times, we just use the query name defined in the `WITH` clause, making the query much easier to read.

```sql
-- Without WITH clause:
SELECT
    e.ename AS employee_name,
    dc1.dept_count AS emp_dept_count,
    m.ename AS manager_name,
    dc2.dept_count AS mgr_dept_count
FROM emp e
JOIN (
    SELECT
        deptno,
        COUNT(*) AS dept_count
    FROM emp
    GROUP BY deptno
) dc1
    ON e.deptno = dc1.deptno
JOIN emp m
    ON e.mgr = m.empno
JOIN (
    SELECT
        deptno,
        COUNT(*) AS dept_count
        FROM emp
        GROUP BY deptno
) dc2
    ON m.deptno = dc2.deptno;

-- With WITH clause:
WITH dept_count AS (
    SELECT
        deptno,
        COUNT(*) AS dept_count
    FROM emp
    GROUP BY deptno
)
SELECT
    e.ename AS employee_name,
    dc1.dept_count AS emp_dept_count,
    m.ename AS manager_name,
    dc2.dept_count AS mgr_dept_count
FROM emp e
JOIN dept_count dc1
    ON e.deptno = dc1.deptno
JOIN emp m
    ON e.mgr = m.empno
JOIN dept_count dc2
    ON m.deptno = dc2.deptno;
```

If the contents of the `WITH` clause are sufficiently complex, Oracle may decide to resolve the result of the subquery into a **global temporary table**. This can make **multiple references** to the subquery more efficient. The `MATERIALIZE` and `INLINE` optimizer hints can be used to influence the decision.

### 5.2. `LIKE` clause
The `LIKE` conditions specify a test involving **pattern matching**. Whereas the **equality operator (=) exactly matches** one character value to another, the `LIKE` conditions match a **portion** of one character value to another by searching the first value for the pattern specified by the second.

#### Using the ESCAPE clause:
The ESCAPE clause identifies `&` as the escape character. In the pattern, the escape character precedes the underscore. This causes Oracle to interpret the underscore literally, rather than as a special pattern matching character.

The following example searches for employees with the pattern `A_B` in their name:

```sql
SELECT last_name
    FROM employees
    WHERE last_name LIKE '%A&_B%' ESCAPE '&';
```

### 5.3. Hints
An optimizer hint is a code snippet within an SQL statement controlling the decisions of the optimizer. The syntax is as follows:

`{DELETE|INSERT|SELECT|UPDATE} /*+ hint [text] [hint [text]] */`

#### Parallel processing
**PARALLEL** _(table n)_: This hint tells the optimizer to use n concurrent servers for a parallel operation:

```sql
/*+ PARALLEL my_table 16 */
```

#### `MATERIALIZE` & `INLINE`

The undocumented `MATERIALIZE` hint tells the optimizer to resolve the subquery as a **global temporary table**, while the `INLINE` hint tells it to process the query **inline**.

```sql
-- Materialize hint
WITH dept_count AS (
  SELECT /*+ MATERIALIZE */
    deptno,
    COUNT(*) AS dept_count
  FROM emp
  GROUP BY deptno
)
SELECT ...

-- Inline hint
WITH dept_count AS (
  SELECT /*+ INLINE */
    deptno,
    COUNT(*) AS dept_count
  FROM emp
  GROUP BY deptno
)
SELECT ...
```

## 6. Data types in Oracle SQL

Chapter contents:

1. Character data types
2. Numeric data types
3. Date / Time data types

### 6.1. Character data types

| Data Type Syntax | Explanation |
| :--------------- | :---------- |
| `char(size)`   | Where size is the number of characters to store. Fixed-length strings. Space padded.   |
| `varchar2(size)`  | Where size is the number of characters to store. Variable-length string.  |

### 6.2. Numeric data types

| Data Type Syntax     | Explanation     |
| :------------- | :------------- |
| `number(p,s)`      | Where `p` is the precision and `s` is the scale. For example, `number(7,2)` is a number that has 5 digits before the decimal and 2 digits after the decimal. This is the super-type for all the numeric data types available in PL/SQL. This stores positive, negative, zero and floating point numbers. Precision can range from 1 to 38. Scale can range from -84 to 127.      |

### 6.3. Date / Time data types

| Data Type Syntax     | Explanation     |
| :------------- | :------------- |
| `date`       | A date between Jan 1, 4712 BC and Dec 31, 9999 AD.       |
| `timestamp` (fractional seconds precision)  | Fractional seconds precision must be a number between 0 and 9 (default is 6). Includes year, month, day, hour, minute, and seconds. For example: `timestamp(6)`  |

## 7. Global constants in Oracle SQL
