# Oracle SQL Cheatsheet

## Table of contents

1. ETL statements
2. Union functions
3. Window functions
4. DB management utilities
5. Special query clauses
6. Data types and global constants in Oracle SQL

## 1. ETL statements
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

## 2. Union functions

## 3. Window functions

## 4. DB Management utilities

## 5. Special query clauses

## 6. Data types and global constants in Oracle SQL

### Character data types

| Data Type Syntax | Explanation |
| :--------------- | :---------- |
| char(size)   | Where size is the number of characters to store. Fixed-length strings. Space padded.   |
| varchar2(size)  | Where size is the number of characters to store. Variable-length string.  |

### Numeric data types
| Data Type Syntax     | Explanation     |
| :------------- | :------------- |
| number(p,s)      | Where p is the precision and s is the scale. For example, number(7,2) is a number that has 5 digits before the decimal and 2 digits after the decimal.       |
|   |   |
