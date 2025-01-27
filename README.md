# Customers and Products Analysis Using SQL
The goal of this project is to analyze data from a sales records database for scale model cars and extract information for decision-making.
Below are the questions we want to answer for this project:

**Question 1**: Which products should we order more of or less of?

**Question 2**: How should we tailor marketing and communication strategies to customer behaviors?

**Question 3**: How much can we spend on acquiring new customers?

The scale model cars database schema is as follows.
![image](https://github.com/user-attachments/assets/f8793a8a-55e5-4120-81c6-1636c4333f04)

It contains eight tables:

Customers: customer data

Employees: all employee information

Offices: sales office information

Orders: customers' sales orders

OrderDetails: sales order line for each sales order

Payments: customers' payment records

Products: a list of scale model cars

ProductLines: a list of product line categories

## Project Outline

I first wrote a query to display the number of attributes and rows for each table in the databaase:
![image](https://github.com/user-attachments/assets/dceb5162-4287-4d48-a72a-7370293fbb30)


###  Question 1: Which Products Should We Order More of or Less of?
This question refers to inventory reports, including low stock(i.e. product in demand) and product performance. This will optimize the supply and the user experience by preventing the best-selling products from going out-of-stock.

The low stock represents the quantity of the sum of each product ordered divided by the quantity of product in stock. We can consider the ten highest rates. These will be the top ten products that are almost out-of-stock or completely out-of-stock.

The product performance represents the sum of sales per product.

Priority products for restocking are those with high product performance that are on the brink of being out of stock.

We'll need these calculations namely: 

low stock = SUM ( quantityOrdered )/quantityInStock

product performance = SUM ( quantityOrdered × priceEach)

```sql
--Write a query to compute the low stock for each product using a correlated subquery.
-- Query to compute low stock for each product:
WITH lowStock AS (
    SELECT 
        p.productCode,
        ROUND(
            (SELECT SUM(od.quantityOrdered) 
             FROM OrderDetails AS od 
             WHERE od.productCode = p.productCode) / p.quantityInStock, 
            2
        ) AS low_stock_rate
    FROM Products AS p
    GROUP BY p.productCode
    ORDER BY low_stock_rate DESC
    
),

--Write a query to compute the product performance for each product.
ProductPerformance AS (
    SELECT 
        od.productCode,
        ROUND(SUM(od.quantityOrdered * od.priceEach), 2) AS total_sales
    FROM OrderDetails od
    GROUP BY od.productCode
   ORDER BY total_sales DESC
    
)

-- Query to combine the results and display priority products for restocking:
SELECT DISTINCT 
    p.productCode,
    p.productName,
    ls.low_stock_rate,
    pp.total_sales
FROM Products p
JOIN lowStock AS ls ON p.productCode = ls.productCode
JOIN ProductPerformance AS pp ON p.productCode = pp.productCode
WHERE p.productCode IN (
    SELECT productCode FROM lowStock
    INTERSECT
    SELECT productCode FROM ProductPerformance
)
ORDER BY ls.low_stock_rate DESC, pp.total_sales DESC
LIMIT 10;

```
Output:
![image](https://github.com/user-attachments/assets/e1b7c970-0709-4086-b165-690c11bcb9f0)

**Top Priority for Restocking:**

Products like 1960 BSA Gold Star DBD34 (S24_2000) and 1968 Ford Mustang (S12_1099) have both high low_stock_ratio and strong sales performance, making them top priorities for restocking.
These items are high-demand and high-revenue generators, meaning restocking them quickly will ensure customer satisfaction and maintain profitability.

**High Revenue with Limited Stock:**

Products like 1928 Mercedes-Benz SSK (S18_2795) and 2002 Yamaha YZR M1 (S50_4713) have excellent sales performance despite being nearly out of stock. These items likely have strong customer interest and could lead to lost sales if not restocked soon.

**Moderate Stock Issues but Strong Performance:**

Products like 1997 BMW F650 ST (S32_1374) and Pont Yacht (S72_3212) are not as critically low in stock as others but still contribute significantly to revenue. Restocking these items will help sustain their momentum.

**Niche Items:**

Products like F/A 18 Hornet 1/72 (S700_3167) and The Mayflower (S700_1938) are generating notable revenue despite limited stock, suggesting they cater to a specific audience. Maintaining availability could enhance customer loyalty in niche markets.


###  Question 2: How Should We Match Marketing and Communication Strategies to Customer Behavior?
This involves categorizing customers: finding the VIP customers and those who are less engaged.

VIP customers bring in the most profit for the store. Less-engaged customers bring in less profit.

For example, we could organize some events to drive loyalty for the VIPs and launch a campaign for the less engaged.

Before we begin, let's compute how much profit each customer generates.
```sql
-- compute profit generated by each customer:
SELECT 
    c.customerNumber,
    c.customerName,
    ROUND(SUM(od.quantityOrdered * (od.priceEach - p.buyPrice)), 2) AS total_profit
FROM Customers AS c
JOIN Orders AS o ON c.customerNumber = o.customerNumber
JOIN OrderDetails AS od ON o.orderNumber = od.orderNumber
JOIN Products AS p ON od.productCode = p.productCode
GROUP BY c.customerNumber, c.customerName
ORDER BY total_profit DESC;
```


## Finding the VIP and Less Engaged Customers
Using the profit per customer from the previous query, finding VIP and less engaged customers is straightforward.
```sql
--Write a query to find the top five VIP customers.
WITH CustomerProfit AS(
    SELECT 
    c.customerNumber,
    c.customerName,
        ROUND(SUM(od.quantityOrdered * (od.priceEach - p.buyPrice)), 2) AS total_profit
    FROM Customers AS c
    JOIN Orders AS o ON c.customerNumber = o.customerNumber
    JOIN OrderDetails AS od ON o.orderNumber = od.orderNumber
    JOIN Products AS p ON od.productCode = p.productCode
    GROUP BY c.customerNumber, c.customerName
    ORDER BY total_profit DESC
)

-- Query to find the top five VIP customers:
SELECT 
    c.contactLastName,
    c.contactFirstName,
    c.city,
    c.country,
    ROUND(cp.total_profit, 2) AS profit
FROM CustomerProfit AS cp
JOIN customers AS c ON cp.customerNumber = c.customerNumber
ORDER BY cp.total_profit DESC
LIMIT 5
```

![image](https://github.com/user-attachments/assets/70f4d2de-caef-4233-850d-c4e9bcf43293)
```sql
-- Query to find the 5 least engaged customers:
SELECT 
    c.contactLastName,
    c.contactFirstName,
    c.city,
    c.country,
    ROUND(cp.total_profit, 2) AS profit
FROM CustomerProfit AS cp
JOIN customers AS c ON cp.customerNumber = c.customerNumber
ORDER BY cp.total_profit ASC
LIMIT 5
```

![image](https://github.com/user-attachments/assets/b63296b0-fa90-47da-9d29-3afe3b122e7b)

**Marketing and Communication Strategy**

**For VIP Customers:**
Organize exclusive events, personalized offers, or loyalty programs to retain and reward these high-value customers. Ensuring their satisfaction is key to maintaining profitability.

**For Less-Engaged Customers:**
Launch targeted campaigns, such as discounts, promotions, or direct communication, to re-engage these customers and potentially increase their spending.


### Question 3: How Much Can We Spend on Acquiring New Customers?
Let's find the number of new customers arriving each month. That way we can check if it's worth spending money on acquiring new customers. 
```sql
WITH 

payment_with_year_month_table AS (
SELECT *, 
       CAST(SUBSTR(paymentDate, 1,4) AS INTEGER)*100 + CAST(SUBSTR(paymentDate, 6,7) AS INTEGER) AS year_month
  FROM payments p
),

customers_by_month_table AS (
SELECT p1.year_month, COUNT(*) AS number_of_customers, SUM(p1.amount) AS total
  FROM payment_with_year_month_table p1
 GROUP BY p1.year_month
),

new_customers_by_month_table AS (
SELECT p1.year_month, 
       COUNT(DISTINCT customerNumber) AS number_of_new_customers,
       SUM(p1.amount) AS new_customer_total,
       (SELECT number_of_customers
          FROM customers_by_month_table c
        WHERE c.year_month = p1.year_month) AS number_of_customers,
       (SELECT total
          FROM customers_by_month_table c
         WHERE c.year_month = p1.year_month) AS total
  FROM payment_with_year_month_table p1
 WHERE p1.customerNumber NOT IN (SELECT customerNumber
                                   FROM payment_with_year_month_table p2
                                  WHERE p2.year_month < p1.year_month)
 GROUP BY p1.year_month
)

SELECT year_month, 
       ROUND(number_of_new_customers*100/number_of_customers,1) AS number_of_new_customers_props,
       ROUND(new_customer_total*100/total,1) AS new_customers_total_props
  FROM new_customers_by_month_table;
```
Output:
![image](https://github.com/user-attachments/assets/cafab8ed-6e1a-4b0a-9c04-fb4ea5f5ffad)

As you can see, the number of clients has been decreasing since 2003, and in 2004, we had the lowest values. The year 2005, which is present in the database as well, isn't present in the table above, this means that the store has not had any new customers since September of 2004. This means it makes sense to spend money acquiring new customers.

To determine how much money we can spend acquiring new customers, we can compute the Customer Lifetime Value (LTV), which represents the average amount of money a customer generates. We can then determine how much we can spend on marketing.
```sql
  
--a query to compute the average of customer profits using the CTE
-- Step 1: Use the previous CTE to calculate profit per customer
WITH CustomerProfit AS (
    SELECT 
        o.customerNumber, 
        SUM(quantityOrdered * (priceEach - buyPrice)) AS profit
    FROM products p
    JOIN orderdetails od ON p.productCode = od.productCode
    JOIN orders o ON o.orderNumber = od.orderNumber
    GROUP BY o.customerNumber
)

-- Step 2: Calculate the average customer profit (LTV)
SELECT 
    ROUND(AVG(profit), 2) AS average_customer_profit
FROM CustomerProfit;
```
Output:![image](https://github.com/user-attachments/assets/8df3deaa-b2d7-4f15-b458-cd975d2b76e7)

LTV tells us how much profit an average customer generates during their lifetime with our store. We can use it to predict our future profit. So, if we get ten new customers next month, we'll earn 390,395 dollars, and we can decide based on this prediction how much we can spend on acquiring new customers.

Additionally, understanding LTV helps balance investments between acquiring new customers and retaining existing ones. While attracting new customers drives growth, ensuring the satisfaction of high-LTV customers (VIPs) can sustain profitability in the long term.

**Reference/source code**
Dataquest. (n.d.). Guided Project: Customers and Products Analysis Using SQL. Retrieved from https://www.dataquest.io/mission/
