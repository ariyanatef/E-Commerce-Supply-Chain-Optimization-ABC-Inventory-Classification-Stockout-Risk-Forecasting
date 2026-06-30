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


