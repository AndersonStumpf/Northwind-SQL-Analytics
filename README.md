# Advanced SQL Reports on Northwind

## Objective

This repository aims to showcase advanced reports built using SQL. The analyses provided here can be applied to businesses of all sizes that aspire to become more data-driven. These reports will help organizations extract valuable insights from their data, aiding in strategic decision-making.

## Reports to Create

1. **Revenue Reports**
    
    * What was the total revenue in 1997?

    ```sql
    -- I'm using Inner Join with subquery because it's more performative than use only a Left Join, because SQL executes FROM and Joins first, so it consumes less data
    -- If I use Left Join first and after that use Where to get only data from 1997, It'll demand more time and data to do.  
    CREATE VIEW total_revenues_1997_view AS
    SELECT SUM((order_details.unit_price) * order_details.quantity * (1.0 - order_details.discount)) AS total_revenues_1997
    FROM order_details
    INNER JOIN (
        SELECT order_id 
        FROM orders 
        WHERE EXTRACT(YEAR FROM order_date) = '1997'
    ) AS ord 
    ON ord.order_id = order_details.order_id;
    ```

    * Perform a monthly growth analysis and calculate YTD (Year-To-Date).

    ```sql
    CREATE VIEW view_cumulative_revenues AS
    WITH MonthlyRevenues AS (
        SELECT
            EXTRACT(YEAR FROM orders.order_date) AS Year,
            EXTRACT(MONTH FROM orders.order_date) AS Month,
            SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) AS Monthly_Revenue
        FROM
            orders
        INNER JOIN
            order_details ON orders.order_id = order_details.order_id
        GROUP BY
            EXTRACT(YEAR FROM orders.order_date),
            EXTRACT(MONTH FROM orders.order_date)
    ),
    CumulativeRevenues AS (
        SELECT
            Year,
            Month,
            Monthly_Revenue,
            SUM(Monthly_Revenue) OVER (PARTITION BY Year ORDER BY Month) AS YTD_Revenue
        FROM
            MonthlyRevenues
    )
    SELECT
        Year,
        Month,
        Monthly_Revenue,
        Monthly_Revenue - LAG(Monthly_Revenue) OVER (PARTITION BY Year ORDER BY Month) AS Monthly_Change,
        YTD_Revenue,
        (Monthly_Revenue - LAG(Monthly_Revenue) OVER (PARTITION BY Year ORDER BY Month)) / LAG(Monthly_Revenue) OVER (PARTITION BY Year ORDER BY Month) * 100 AS Monthly_Percentage_Change
    FROM
        CumulativeRevenues
    ORDER BY
        Year, Month;

    ```

2. **Customer Segmentation**
    
    * What is the total amount paid by each customer?

    ```sql
    CREATE VIEW view_total_revenues_per_customer AS
    SELECT 
        customers.company_name, 
        SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) AS total
    FROM 
        customers
    INNER JOIN 
        orders ON customers.customer_id = orders.customer_id
    INNER JOIN 
        order_details ON order_details.order_id = orders.order_id
    GROUP BY 
        customers.company_name
    ORDER BY 
        total DESC;

    ```

    * Group customers into 5 segments based on total payments.

    ```sql
    CREATE VIEW view_customer_segments AS
    SELECT 
        customers.company_name, 
        SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) AS total,
        NTILE(5) OVER (ORDER BY SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) DESC) AS group_number
    FROM 
        customers
    INNER JOIN 
        orders ON customers.customer_id = orders.customer_id
    INNER JOIN 
        order_details ON order_details.order_id = orders.order_id
    GROUP BY 
        customers.company_name
    ORDER BY 
        total DESC;

    ```


    * Identify customers in groups 3, 4, and 5 for targeted marketing

    ```sql
    CREATE VIEW clients_for_marketing AS
    WITH marketing_clients AS (
        SELECT 
            customers.company_name, 
            SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) AS total,
            NTILE(5) OVER (ORDER BY SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) DESC) AS group_number
        FROM 
            customers
        INNER JOIN 
            orders ON customers.customer_id = orders.customer_id
        INNER JOIN 
            order_details ON order_details.order_id = orders.order_id
        GROUP BY 
            customers.company_name
        ORDER BY 
            total DESC
    )
    SELECT *
    FROM marketing_clients
    WHERE group_number >= 3;
    ```

3. **Top 10 Best-Selling Products**
    
    * Identify the 10 best-selling products.

    ```sql
    CREATE VIEW top_10_products AS
    SELECT products.product_name, SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) AS sales
    FROM products
    INNER JOIN order_details ON order_details.product_id = products.product_id
    GROUP BY products.product_name
    ORDER BY sales DESC;
    ```

4. **UK Customers Paying Over $1,000**
    
    * Which UK customers have paid over $1,000?

    ```sql
    CREATE VIEW uk_clients_over_1000 AS
    SELECT customers.contact_name, SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) AS payments
    FROM customers
    INNER JOIN orders ON orders.customer_id = customers.customer_id
    INNER JOIN order_details ON order_details.order_id = orders.order_id
    WHERE LOWER(customers.country) = 'uk'
    GROUP BY customers.contact_name
    HAVING SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) > 1000;
    ```

## Context

The `Northwind` database contains sales data for a company called `Northwind Traders`, which imports and exports specialty foods worldwide.

The Northwind database is an ERP system with data on customers, orders, inventory, purchases, suppliers, shipping, employees, and accounting.

The Northwind dataset includes sample data for:

* **Suppliers:** Vendors of Northwind
* **Customers:** Buyers of Northwind's products
* **Employees:** Details about Northwind Traders' staff
* **Products:** Product information
* **Shippers:** Details of transporters delivering goods
* **Orders and Order Details:** Sales transactions between customers and the company

The database includes 14 tables, with relationships shown in the following entity-relationship diagram:

![northwind](https://github.com/lvgalvao/Northwind-SQL-Analytics/blob/main/pics/northwind-er-diagram.png?raw=true)


## Initial Setup

### Manually

Use the provided `nortwhind.sql` file to populate your database.

### With Docker and Docker Compose

**Prerequisites**: Install Docker and Docker Compose

* [Get Started with Docker](https://www.docker.com/get-started)
* [Install Docker Compose](https://docs.docker.com/compose/install/)

### Steps for Docker Setup:

1. **Start Docker Compose** 
    Run the following command to start the services:
    
    ```
    docker-compose up
    ```
    
    Wait for the config menssage, like:
    
    ```csharp
    Creating network "northwind_psql_db" with driver "bridge"
    Creating volume "northwind_psql_db" with default driver
    Creating volume "northwind_psql_pgadmin" with default driver
    Creating pgadmin ... done
    Creating db      ... done
    ```
       
2. **Connect to PgAdmi** Access PgAdmin at: [http://localhost:5050](http://localhost:5050), with the password `postgres`. 

Configure a new server in PgAdmin:
    
    * **General Tab**:
        * Nome: db
    * **Connection Tab**:
        * Nome do host: db
        * Nome de usuário: postgres
        * Senha: postgres 
        * Database: "northwind"

3. **Stop Docker Compose** Stop the services using Ctrl-C and remove containers with: `docker-compose up` 
    
    ```
    docker-compose down
    ```
    
4. **Data Persistence** Database modifications persist in the Docker volume `postgresql_data` To delete data, run:
    
    ```
    docker-compose down -v
    ```
# Northwind-SQL-Analytics
