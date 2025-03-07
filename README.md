# Supply-Chain-Dashboard-3

Problem Statement: 
Use Kaggle set to explore and create a comprehensive ain dashboard. 

https://www.kaggle.com/datasets/talhabu/us-regional-sales-data

Step 1) 
Clean and index data in Excel:

1) Trimmed order number to a whole number with no string/text values

2) Transpose sales_channel values (in-store, online, distributor, and wholesale) from column to row to attach the warehouse information or store information with the correct sales_channel. So, in-store sales have a store_id, store_branch, and store_location, but a null or 0 value for warehouse-related information. 

3) Raw data dates did not follow logic where Procured Date, Order Date, Ship Date, and Delivery Date are in order from oldest to newest dates. I created new dates using data validation and the randomizer function with DAX. 


  ```
ProcuredDate:
In cell D2: =RANDBETWEEN(DATE(2020,1,1), DATE(2022,12,31)).

OrderDate:
In cell E2: =D2 + RANDBETWEEN(1, 30)
Between 1-30 days after the procured date 

ShipDate:
In cell F2: =E2 + RANDBETWEEN(1, 10)
Between 1-10 days after the order date 

DeliveryDate:
In cell G2: =F2 + RANDBETWEEN(1, 15)
Between 1-15 days after the ship date  

Data validation 
=IF(AND(A2 < B2, B2 < C2, C2 < D2), "Valid", "Invalid")

(Conditional formating rule works as well
=OR(A2 >= B2, B2 >= C2, C2 >= D2))

  ```

4) Separated year and month from each date to analyze by month and year separately using both
=year() 
=month() and
=TEXT(D2, "mm") or =TEXT(D2, "yyyy") 

5) Remove currency column (USD)

6) Re-indexed order numbers 

7) Change sales_team to a randomized list of names and attach an index as sales_team_id as a number from 1-15


  ```
=INDEX({"Team Alpha","Team Beta","Team Omega","Team Charlie","Team Dynamo","Team Phoenix","Team Pinnacle","Team Titan","Team Velocity","Team Echo","Team Mayhem","Team Synergy","Team Invictus","Team Delta","Team Bravo"}, RANDBETWEEN(1, 15))
  ```

8) Add in columns for sale_price (unit price – unit price* discount_applied) & Add column for profit (sale_price – unit_cost (cogs)) 

9) Added a randomized number for holding cost with wholesale, distributor, online between 5-30, and store as 1-10

10) Created warehouse_branch_code to a number that matches with a randomized location from a list (ie, 7338 is for Charlotte, NC)


  ```
=INDEX({"Austin, TX", "Charlotte, NC", "Chicago, IL", "Columbus, OH", "Dallas, TX", "Denver, CO", "Fort Worth, TX", "Houston, TX", "Indianapolis, IN", "Jacksonville, FL", "Los Angeles, CA", "New York, NY", "Philadelphia, PA", "Phoenix, AZ", "San Antonio, TX", "San Diego, CA", "San Francisco, CA", "San Jose, CA", "Seattle, WA", "Washington, D.C."}, RANDBETWEEN(1, 20))


  ```

11) Indexed product_id, payment_type, and customer_gender. Along with warehouse_code, store_id, product_id, and salesteam_id. 

  ```
=INDEX({"Electronic accessories", "Fashion accessories", "Food and beverages", "Health and beauty", "Home and lifestyle", "Sports and travel"}, RANDBETWEEN(1, 6))

=INDEX({"Cash","Credit Card","Ewallet"}, RANDBETWEEN(1, 3))

=INDEX({"Male","Female"}, RANDBETWEEN(1, 2))

  ```

![Screen Shot 2025-03-06 at 12 08 38 PM](https://github.com/UserDna95/Supply-Chain-Dashboard-3/blob/main/2025-03-06%20(1).png)

Step 2)
Upload CSV file into MySQL and use data import wizard to insert values from a csv file into table structures. The following shows the initial structure for the Orders table. 

  ```
CREATE TABLE Orders (
    order_number INT PRIMARY KEY,
    customer_id INT,
    product_id INT,
    store_id INT,
    warehouse_code INT, 
    salesteam_id INT,
    sales_channel VARCHAR(50),
    procured_date DATE,
    procured_date_year INT,
	procured_date_month INT, 
    order_date DATE,
    order_date_year INT,
    order_date_month INT,
    ship_date DATE,
    ship_date_year INT,
    ship_date_month INT,
    delivery_date DATE,
    delivery_date_year INT,
	delivery_date_month INT,
    order_quantity INT,
    FOREIGN KEY (customer_id) REFERENCES Customer(customer_id),
    FOREIGN KEY (product_id) REFERENCES Product(product_id),
    FOREIGN KEY (store_id) REFERENCES Store(store_id),
    FOREIGN KEY (salesteam_id) REFERENCES Salesteam(salesteam_id),
	FOREIGN KEY (warehouse_code) REFERENCES Warehouse(warehouse_code)
);
  ```

![Screen Shot 2025-03-06 at 12 09 38 PM](https://github.com/UserDna95/Supply-Chain-Dashboard-3/blob/main/supplychaindashboard2.png)

Step 3)
Create SQL queries to create and understand the KPI's of the overall supply chain data. 

A simple SQL query aggregates data, such as finding out which sales channel is most profitable. 

  ```
SELECT 
    sales_channel, 
    SUM(profit) AS total_profit
FROM 
    Orders AS o
JOIN 
    Product AS p 
ON 
    o.product_id = p.product_id
GROUP BY 
    sales_channel
ORDER BY 
    total_profit DESC;

  ```

More complex queries looked at sales team performance, product demand, and price elasticity, to name a few.
  
Which sales team is offering the most discount and providing a positive net profit (this can help create strategies for other teams and to help assess which products are being discounted the most and why)

  ```
SELECT 
    st.salesteam_name,
    SUM(p.discount_applied) AS total_discount,
    SUM(p.profit) AS total_profit,
    (SUM(p.profit) - SUM(p.discount_applied)) AS net_profit
FROM 
    Orders AS o
JOIN 
    Product AS p 
ON 
    o.product_id = p.product_id
JOIN 
    SalesTeam AS st
ON 
    o.salesteam_id = st.salesteam_id
GROUP BY 
    st.salesteam_name
ORDER BY 
    net_profit DESC;

  ```

Economic order quantity (EOQ) determines the optimal order quantity that minimizes total inventory costs, including holding and ordering costs.
  ```
SELECT 
    p.product_type, 
    ROUND(SQRT((2 * annual_demand * ordering_cost) / holding_cost), 2) AS eoq
FROM 
    (SELECT 
        p.product_type, 
        SUM(o.order_quantity) AS annual_demand,
        35 AS ordering_cost,  
        AVG(p.holding_cost) AS holding_cost
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id
    GROUP BY 
        p.product_type) AS product_data;

SELECT 
    product_data.product_type, 
    ROUND(SQRT((2 * product_data.annual_demand * product_data.ordering_cost) / product_data.holding_cost), 2) AS eoq
FROM 
    (SELECT 
        p.product_type, 
        SUM(o.order_quantity) AS annual_demand,
        35 AS ordering_cost,  
        AVG(p.holding_cost) AS holding_cost
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id
    GROUP BY 
        p.product_type) AS product_data;

  ```

Safety stock is the extra inventory held to mitigate the risk of stock outs due to demand and lead time variability.

  ```
WITH DemandLeadTimeStats AS (
    SELECT 
        p.product_type,
        AVG(DATEDIFF(o.delivery_date, o.order_date)) AS avg_lead_time_days,
        STDDEV(DATEDIFF(o.delivery_date, o.order_date)) AS stddev_lead_time,
        SUM(o.order_quantity) / COUNT(DISTINCT o.order_date) AS avg_daily_demand,
        STDDEV(o.order_quantity / DATEDIFF(o.order_date, o.procured_date)) AS stddev_daily_demand
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id
    GROUP BY 
        p.product_type
)
SELECT 
    product_type,
    1.65 * SQRT((stddev_lead_time ^ 2 * avg_daily_demand ^ 2) + (stddev_daily_demand ^ 2 * avg_lead_time_days ^ 2)) AS safety_stock
FROM 
    DemandLeadTimeStats;

  ```

Reorder Point (ROP) indicates the inventory level at which a new order should be placed to avoid stock outs.

  ```
WITH DemandLeadTimeStats AS (
    SELECT 
        p.product_type,
        AVG(DATEDIFF(o.delivery_date, o.order_date)) AS avg_lead_time_days,
        STDDEV(DATEDIFF(o.delivery_date, o.order_date)) AS stddev_lead_time,
        SUM(o.order_quantity) / COUNT(DISTINCT o.order_date) AS avg_daily_demand,
        STDDEV(o.order_quantity / DATEDIFF(o.order_date, o.procured_date)) AS stddev_daily_demand
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id
    GROUP BY 
        p.product_type
)
SELECT 
    product_type,
    (avg_daily_demand * avg_lead_time_days) AS lead_time_demand,
    1.65 * SQRT((stddev_lead_time ^ 2 * avg_daily_demand ^ 2) + (stddev_daily_demand ^ 2 * avg_lead_time_days ^ 2)) AS safety_stock,
    (avg_daily_demand * avg_lead_time_days) + 
    (1.65 * SQRT((stddev_lead_time ^ 2 * avg_daily_demand ^ 2) + (stddev_daily_demand ^ 2 * avg_lead_time_days ^ 2))) AS reorder_point
FROM 
    DemandLeadTimeStats;

  ```

Inventory turnover ratio measures how often inventory is sold and replaced over a period.

  ```
WITH cogs_data AS (
    SELECT 
        p.product_type, 
        SUM((p.unit_cost + p.holding_cost) * o.order_quantity) AS cogs,
        SUM(o.order_quantity) AS total_order_quantity
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id
    GROUP BY 
        p.product_type
)

SELECT 
    product_type, 
    cogs / NULLIF(total_order_quantity, 0) AS inventory_turnover_ratio
FROM 
    cogs_data;

  ```

Cost of goods (COGS)

  ```
SELECT 
    p.product_type, 
    SUM((p.unit_cost + p.holding_cost) * o.order_quantity) AS cost_of_goods
FROM 
    Orders AS o
JOIN 
    Product AS p 
ON 
    o.product_id = p.product_id
GROUP BY 
    p.product_type;

  ```

Gross income/gross margin percentage

  ```
WITH cogs_data AS (
    SELECT 
        p.product_type, 
        SUM((p.unit_cost + p.holding_cost) * o.order_quantity) AS cogs
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id
    GROUP BY 
        p.product_type
),

revenue_data AS (
    SELECT 
        p.product_type, 
        SUM(p.profit * o.order_quantity) AS revenue
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id
    GROUP BY 
        p.product_type
)

SELECT 
    r.product_type, 
    r.revenue,
    c.cogs,
    (r.revenue - c.cogs) AS gross_income,
    ((r.revenue - c.cogs) / NULLIF(r.revenue, 0)) * 100 AS gross_margin_percentage
FROM 
    revenue_data AS r
JOIN 
    cogs_data AS c
ON 
    r.product_type = c.product_type;

  ```
Demand by product category.

  ```
SELECT 
    p.product_type, 
    SUM(o.order_quantity) AS total_demand
FROM 
    Orders AS o
JOIN 
    Product AS p 
ON 
    o.product_id = p.product_id
GROUP BY 
    p.product_type;

  ```

Moving average demand is a method used to smooth out fluctuations in data, making it easier to identify trends. The following SQL query calculates the moving average demand for products over the past three months.

  ```
SELECT 
    p.product_type,
    o.order_date_year,
    o.order_date_month,
    SUM(o.order_quantity) AS monthly_demand,
    AVG(SUM(o.order_quantity)) OVER (
        PARTITION BY p.product_type 
        ORDER BY o.order_date_year, o.order_date_month 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_average_demand
FROM 
    Orders AS o
JOIN 
    Product AS p 
ON 
    o.product_id = p.product_id
WHERE 
    o.order_date_year BETWEEN 2022 AND 2023
GROUP BY 
    p.product_type, o.order_date_year, o.order_date_month
ORDER BY 
    p.product_type, o.order_date_year, o.order_date_month;

  ```

Forecasted demand uses historical data to forecast the demand of a product in the near future. 

  ```
WITH monthly_demand AS (
    SELECT 
        p.product_type,
        o.order_date_year,
        o.order_date_month,
        SUM(o.order_quantity) AS monthly_demand
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id
    WHERE 
        o.order_date_year BETWEEN 2019 AND 2023
    GROUP BY 
        p.product_type, o.order_date_year, o.order_date_month
),

moving_average AS (
    SELECT 
        product_type,
        order_date_year,
        order_date_month,
        monthly_demand,
        AVG(monthly_demand) OVER (
            PARTITION BY product_type 
            ORDER BY order_date_year, order_date_month 
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS moving_average_demand
    FROM 
        monthly_demand
)

SELECT 
    product_type,
    MAX(order_date_year) + 1 AS forecast_year,
    MAX(order_date_month) + 1 AS forecast_month,
    AVG(moving_average_demand) AS forecasted_demand
FROM 
    moving_average
GROUP BY 
    product_type
ORDER BY 
    product_type;


  ```

Demand Elasticity analyzes how changes in price affect the quantity demanded (price elasticity of demand).

  ```
SELECT 
    product_type, 
    (SUM(new_quantity) - SUM(old_quantity)) / SUM(old_quantity) AS percentage_change_in_quantity,
    (SUM(new_price) - SUM(old_price)) / SUM(old_price) AS percentage_change_in_price,
    (SUM(new_quantity) - SUM(old_quantity)) / SUM(old_quantity) / ((SUM(new_price) - SUM(old_price)) / SUM(old_price)) AS price_elasticity
FROM 
    (SELECT 
        p.product_type, 
        o.order_quantity AS old_quantity, 
        p.sale_price AS old_price,
        LEAD(o.order_quantity) OVER (PARTITION BY p.product_type ORDER BY o.order_date) AS new_quantity,
        LEAD(p.sale_price) OVER (PARTITION BY p.product_type ORDER BY o.order_date) AS new_price
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id) AS price_data
GROUP BY 
    product_type;

  ```

ABC/XYZ analysis categorizes products on predictability and variability. Ideally, we want high predictability in the sales of a product with low variability.  

  ```
WITH InventoryValue AS (
    SELECT
        p.product_id,
        p.product_type,
        p.unit_price,
        SUM(o.order_quantity) AS total_quantity,
        SUM(o.order_quantity * p.unit_price) AS annual_consumption_value
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id
    WHERE 
        o.order_quantity IS NOT NULL
    GROUP BY 
        p.product_id, p.product_type
),
DemandStats AS (
    SELECT
        p.product_id,
        p.product_type,
        AVG(o.order_quantity) AS avg_demand,
        STDDEV(o.order_quantity) AS stddev_demand,
        CASE 
            WHEN AVG(o.order_quantity) = 0 THEN 0
            ELSE STDDEV(o.order_quantity) / AVG(o.order_quantity)
        END AS cv_demand
    FROM 
        Orders AS o
    JOIN 
        Product AS p 
    ON 
        o.product_id = p.product_id
    GROUP BY 
        p.product_id, p.product_type
),
ABCClassified AS (
    SELECT
        *,
        @total_consumption := @total_consumption + annual_consumption_value AS cumulative_consumption,
        CASE
            WHEN @total_consumption <= @total_value * 0.8 THEN 'A'
            WHEN @total_consumption <= @total_value * 0.95 THEN 'B'
            ELSE 'C'
        END AS abc_classification
    FROM
        InventoryValue, (SELECT @total_consumption := 0, @total_value := (SELECT SUM(annual_consumption_value) FROM InventoryValue)) as init
    ORDER BY 
        annual_consumption_value DESC
),
CombinedStats AS (
    SELECT
        a.product_id,
        a.product_type,
        a.unit_price,
        a.total_quantity,
        a.annual_consumption_value,
        a.abc_classification,
        d.avg_demand,
        d.stddev_demand,
        d.cv_demand,
        CASE
            WHEN d.cv_demand < 0.5 THEN 'X'
            WHEN d.cv_demand BETWEEN 0.5 AND 1.0 THEN 'Y'
            ELSE 'Z'
        END AS xyz_classification
    FROM 
        ABCClassified a
    JOIN 
        DemandStats d 
    ON 
        a.product_id = d.product_id
)
SELECT
    product_id,
    product_type,
    unit_price,
    total_quantity,
    annual_consumption_value,
    abc_classification,
    avg_demand,
    stddev_demand,
    cv_demand,
    xyz_classification
FROM
    CombinedStats
ORDER BY
    abc_classification, xyz_classification, annual_consumption_value DESC;

  ```

Step 4) 
Dashboard in PowerBI

How to connect MYSQL to PowerBI
1) Get data > mysql

2) Click on database on the left and input the MYSQL login information (root username) and connect the hostname and port name on the database name

3) Load or transform data

![Screen Shot 2025-03-06 at 12 19 38 PM](https://github.com/UserDna95/Supply-Chain-Dashboard-3/blob/main/2025-03-06%20(3).png)



Key Insights:



![Screen Shot 2025-03-05 at 8 09 38 PM]()
![Screen Shot 2025-03-05 at 8 09 38 PM]()
