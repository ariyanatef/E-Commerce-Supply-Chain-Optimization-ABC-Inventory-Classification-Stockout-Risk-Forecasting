# Ask
Problem: 

There are a lot of large e-commerce platforms on the Internet. However, they need to manage all their inventory to determine which items
are good for business and which items aren't. One example of a large e-commerce platform is Olist, a Brazilian E-commerce company that focuses on
providing resources to small businesses. Due to the large amount of supply volume, supply chain teams can be overwhelmed and make mistakes in prioritizing
products. Without clear priotization, manager waste a large amount of time and money restocking low-margin, slow-moving products while products producing
large amounts of revenue sit empty which can cause massive, unmitigated revenue losses.

Stakeholders:

VP of Supply Chain Operations: They want to minimize the amount of time products remain out of stock and stabilize the marketplace fulfillment metric.
Chief Financial Officer (CFO): They want to maximize revenue protection and ensure capital is being use properly.

Guiding Questions:

* Which products generate the top 80% of Olist's total marketplace revenue?
* Which of these high-value products are currently out of stock or at critical risk of going out of stock within 5 days?
* How can we urgently and visually show off the logisitics teams which items are important for the business to keep?


# Prepare

1. Go to Kaggle.com and search up "Brazilian E-Commerce Public Dataset by Olist".
2. Download the zipfile and extract it (I extracted it to my desktop)
3. We will be taking a look at three essential files:
![Which files we are looking at](Visuals/Which%20files%20we%20are%20looking%20at.png)   

4. I will be using BigQuery Sandbox to perform these SQL queries but you can use whatever works best for you. Just keep in mind that the queries might vary slightly depending on what you use.
5. After going to BigQuery Sandbox, I am going to create a project and name it “ABC Inventory Analysis”
6. After creating the project, we have to upload the datasets.
   We have to first create a dataset and to do so, we click on add data.
   Then we click on the local file and then the local file again.
   Click on browse and find where your files are downloaded (mine are on the desktop).

7. We want to upload the three csv files that have been marked in step 3.
   I uploaded olist_order_items_dataset.csv first.
   Now we have to create a dataset which I will name “olist_inventory” but name it how you see fit.

8. Now name the table.
   I will name it the name of the file without the csv (ex: olist_order_items_dataset).
   Click on autodetect for schema.
   Repeat steps 6-8 for the other files, ensuring you select the name of the dataset you created in step 7.

9. The first thing I want to is to find any duplicates before we join the datasets.
    So in order to do that, I used this query below:
```sql
   SELECT
      order_id,
      order_item_id,
      COUNT (*) AS duplicate_count
   FROM
      `olist_inventory.olist_order_items_dataset`
GROUP BY
   order_id,
   order_item_id
HAVING duplicate_count > 1;
```
We see that there is no data to display. However we need to edit and run this for the other two datasets as well.

10. Due to the olist_orders_dataset & the olist_products_dataset having different primary keys, we have to slightly edit the code so it runs properly.
    ```sql
    SELECT
       product_id,
       COUNT (*) AS duplicate_count
    FROM
       `olist_inventory.olist_products_dataset`
    GROUP BY
       product_id
    HAVING
       COUNT (*) > 1;
    ```

   ```sql
   SELECT
      order_id,
      COUNT (*) AS duplicate_count
   FROM
      `olist_inventory.olist_orders_dataset`
   GROUP BY
      order_id
   HAVING
      COUNT (*) ? 1;
   ```
   10a) After performing Steps 9 & 10, the results for all of them show that there is no data to display indicating there are no duplicate values.

 11. Now we have to check for null values. In order to do that, I used this query below:
      ```sql
     SELECT
        COUNTIF(oi.product_id IS NULL) AS missing_product_ids,
        COUNTIF(oi.price IS NULL) AS missing_prices,
        COUNTIF(p.product_category_name IS NULL) AS missing_categories,
        COUNTIF(o.order_purchase_timestamp IS NULL) AS missing_timestamps
      FROM
        `olist_inventory.olist_order_items_dataset` oi
      JOIN
        `olist_inventory.olist_orders_dataset` o ON oi.order_id = o.order_id
      LEFT JOIN
        `olist_inventory.olist_products_dataset` p ON oi.product_id = p.product_id;
      ```
We see that there are 1,603 missing categories in the results. We will filter these out once we start our analysis.


*keep in mind all of this will go on one query window, I am breaking it down for learning purposes
12. Now that we have our preliminary checks out of the way, we will conduct our actual analysis using SQL. 
First we want to ensure we have a new table that combines all of our data tables alongside having completely unique values. To do this we will use this query:
```sql
CREATE OR REPLACE TABLE `olist_inventory.inventory_analysis_results` AS 
WITH deduplicated_items AS (
  SELECT
    order_id,
    order_item_id,
    product_id,
    price
  FROM
    `olist_inventory.olist_order_items_dataset`
    WHERE
      product_id IS NOT NULL
      AND price IS NOT NULL
    QUALIFY
      ROW_NUMBER() OVER(PARTITION BY order_id, order_item_id ORDER BY price DESC) = 1
),
```

13. For this second part, we want to aggregate all the clean sales and timeline information. 
We also want to handle missing categories and focus on completed operational cycles. To do so, we want to use this query below:
```sql
base_sales AS (
  SELECT
    oi.product_id,
    COALESCE (p.product_category_name, 'No Category Listed') AS category,
    COUNT(oi.order_item_id) AS total_units_sold,
    SUM(oi.price) AS total_revenue,
    AVG(oi.price) AS unit_price,
    DATE(MIN(o.order_purchase_timestamp)) AS first_sale_date,
    DATE(MAX(o.order_purchase_timestamp)) AS last_sale_date
  FROM
    deduplicated_items oi
  JOIN
    `olist_inventory.olist_orders_dataset` o ON oi.order_id = o.order_id
  LEFT JOIN
    `olist_inventory.olist_products_dataset` p ON oi.product_id = p.product_id
  WHERE
    o.order_status = 'delivered'
    AND o.order_purchase_timestamp IS NOT NULL
  GROUP BY
    oi.product_id, category
),
```

14. Now we want to calculate daily sales velocity (how many products are being sold) based on active selling days. To do so, we will use this query:
```sql
product_velocity AS (
  SELECT
    *,
    CASE
     WHEN DATE_DIFF(last_sale_date, first_sale_date, DAY) = 0 THEN 1
     ELSE DATE_DIFF(last_sale_date, first_sale_date, DAY)
    END AS active_days,
    ROUND (total_units_sold / CASE
      WHEN DATE_DIFF(last_sale_date, first_sale_date, DAY) = 0 THEN 1
      ELSE DATE_DIFF(last_sale_date, first_sale_date, DAY)
    END, 4) AS avg_daily_sales
  FROM
    base_sales
),
```
15. Now we want to sort our products by revenue, add up the running total, and see which items generate the majority of your sales (finding the ABC curve):
```sql
abc_shares AS (
  SELECT 
  *,
  SUM(total_revenue) OVER () AS total_portfolio_revenue,
  SUM(total_revenue) OVER (ORDER BY total_revenue DESC) AS cumulative_revenue
  FROM
    product_velocity  
),
```

16. Now we want to implement the Standard 80/15/5 pareto bands onto our inventory analysis. The standard 80/15/5 pareto bands are a three-tier system that will allow us to determine who products or demographics are the most valuable. It will help us focus time and money and put those resources to where it is most valuable.

   a) To break it down, we start with the 80% Band (The Vital Few) which makes up 80% of the total value. These items generate the most revenue caused by a few of your products.
   b) The 15% band (The Moderates) are your middle ground resources. These things aren’t anything extraordinary but they provide steady, reliable value to your business. They are mostly worth keeping but won’t contribute to huge results.

   c) Lastly we have the 5% band. These items do not provide any value to your business and are just wasting resources. These can be items, tasks, or even target audiences that don’t contribute much to the business.

17. In order to use the 80/15/5 pareto bands in our inventory analysis, we will use this query:
```sql
abc_classified AS (
SELECT
    *,
    CASE 
      WHEN (cumulative_revenue / total_portfolio_revenue) <= 0.80 THEN 'A'
      WHEN (cumulative_revenue / total_portfolio_revenue) <= 0.95 THEN 'B'
      ELSE 'C'
    END AS abc_class,
CAST(FLOOR(total_units_sold * 0.15) AS INT64) AS current_inventory
FROM
  abc_shares
)
```

18. Finally we will construct the operational runout & stockout risk levels. 
This essentially will allow us to determine which items are out of stock and how crucial those items are for the business. In order to do this, we will use this query:
```sql
SELECT
  product_id,
  category,
  ROUND(unit_price, 2) AS unit_price,
  total_units_sold,
  ROUND(total_revenue, 2) AS total_revenue,
  current_inventory,
  ROUND(avg_daily_sales, 2) AS avg_daily_sales,
  abc_class,
  CASE 
    WHEN avg_daily_sales = 0 THEN 999
    ELSE ROUND(current_inventory / avg_daily_sales, 1)
  END AS days_of_supply,
  CASE 
    WHEN current_inventory = 0 THEN 'Out of Stock'
    WHEN (current_inventory / avg_daily_sales) <= 5.0 THEN 'Critical Risk (<5 Days)'
    WHEN (current_inventory / avg_daily_sales) <= 14.0 THEN 'Medium Risk (5-14 Days)'
    ELSE 'Healthy (>14 Days)'
  END AS stockout_risk_level
FROM
  abc_classified;
```

19. After running the query, it created a new table called inventory_analysis_results. It has all of our clean data with all of the necessary labels ready for analysis. Now we need to export it and upload it onto Tableau. 
   a) To do this, we need to allow our project to have access to the BigQuery API.
   b) Then we need to open Google Sheets and click on Data, Data Connectors, and then connect to BigQuery.
   c) Select your project name (in my case I named it ABC Inventory Analysis but name it however you want), then click on your inventory_analysis_results table.
   d) Finally click on file, download, and then click on Comma Separated Values (.csv). This will allow you to store it onto your downloads folder and open it in Tableau.

# Analyze & Share Data

21. 




   
