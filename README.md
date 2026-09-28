## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Data Modelling](#2-data-modelling)
   - [2.1 Data Dictionary](#21-data-dictionary)
   - [2.2 Data Preprocessing](#22-data-preprocessing)
   - [2.3 Data Model](#23-data-model)
3. [Insights & Recommendations](#3-insights--recommendations)
4. [Dashboard Screenshots](#4-dashboard-screenshots)

<br>

## 1. Problem Statement
<p align="justify">
The company has a large volume of historical sales data, but its highly denormalized structure makes it difficult to identify key drivers of sales performance. The requirements are to understand which pizza categories and products generate the most revenue and sales volume, how demand changes over time, which ingredients contribute most to product performance, and how ordering patterns vary across customers and order types. The objective of this project is to transform the raw sales data into a structured business intelligence solution that provides clear insights into product performance, revenue trends, customer ordering behavior, and ingredient demand. These insights can then be used to identify underperforming products, recognize sales trends, and support more informed decisions around product offerings and operational planning.
</p>

## 2. Data Modelling

### 2.1 Data Dictionary
<p align="justify">
The description of each field in the raw sales data file can be seen in the table below:
</p>

<div align="center">

| Field | Description |
|:---:|:---|
| order_id | Unique identifier for each order placed by a table |
| date | Date the order was placed (entered into the system prior to cooking & serving). Current date format is `dddd, MMMM d, yyyy` |
| time | Time the order was placed (entered into the system prior to cooking & serving). Current time format is `h:mm:ss AM/PM` |
| quantity | Quantity ordered for each pizza of the same type and size |
| pizza_id | Unique identifier for each pizza (constituted by its type and size) |
| size | Size of the pizza (`Small`, `Medium`, `Large`, `X Large`, or `XX Large`) |
| price | Price of the pizza in USD |
| name | Name of the pizza as shown in the menu |
| category | Category that the pizza fall under in the menu (`Classic`, `Chicken`, `Supreme`, or `Veggie`) |
| ingredients | Comma-delimited ingredients used in the pizza as shown in the menu (they all include Mozzarella Cheese, even if not specified; and they all include Tomato Sauce, unless another sauce is specified) |

<em>Table 1. Data Dictionary table</em>
</div>

### 2.2 Data Preprocessing
<p align="justify">
The raw sales table was transformed into a relational data model consisting of fact, dimension, and bridging tables. Five copies of the raw table were initially created to independently construct the required tables.
</p>

#### Data Normalization
<p align="justify">
The raw data was separated into dimension and fact tables. For the dimension tables, relevant columns were retained, text values were trimmed, appropriate data types were assigned, and numerical values were rounded where required. Auxiliary columns were also created where necessary, such as <code>size_number</code>, which provides a logical ordering for pizza sizes (<code>S</code>, <code>M</code>, <code>L</code>, <code>XL</code>, <code>XXL</code>). Duplicate rows were removed to ensure dimensional uniqueness, and primary keys were generated for each dimension. Later on, additional columns will also be created using DAX to support new analysis requirements.

For the fact tables, the dimension tables were joined using **left joins** to map the corresponding foreign keys. The resulting foreign keys were expanded into the fact tables, while redundant descriptive attributes were removed.
</p>

#### Ingredient Normalization
<p align="justify">
The original ingredients column contained multiple ingredients within a single field. The values were therefore split using the comma delimiter, followed by unpivoting, trimming, and duplicate removal. An index was then added to generate the <code>ingredient_id</code> column. Because <code>Tomato Sauce</code> was not explicitly present in the source ingredient data, a temporary one-row table containing the ingredient was created and combined with the resulting ingredient table to produce the complete ingredient dimension.
</p>

#### Pizza–Ingredient Bridge Table
<p align="justify">
A bridging table was created to establish the many-to-many relationship between pizzas and ingredients. The ingredient data was transformed into individual pizza–ingredient records.

Since <code>Mozzarella Cheese</code> is included in every pizza regardless of whether it is explicitly listed in the source data, a custom column was added to assign Mozzarella Cheese to every pizza. Similarly, the data dictionary specifies that every pizza contains <code>Tomato Sauce</code> **unless another sauce is specified**. A secondary table was therefore created containing only pizzas without an explicitly specified sauce. Tomato Sauce was assigned to these pizzas before the records were combined with the existing pizza–ingredient bridge table.
</p>

<br>
<br>

The resulting data model consists of the following tables:
| Table                | Columns                                                        |
| -------------------- | -------------------------------------------------------------- |
| **Order**            | `order_id`, `date`, `time`, `type`                             |
| **Pizza**            | `pizza_id`, `size`, `price`, `name`, `category`, `size_number` |
| **Ingredient**       | `ingredient_id`, `ingredient_name`                             |
| **Pizza–Ingredients** | `pizza_id`, `ingredient_id`                                   |
| **Orderline**        | `order_id`, `pizza_id`, `quantity`                             |


### 2.3 Data Model
<p align="justify">
Dedicated Date and Time dimension tables were created to support time-intelligence analysis and improve the efficiency of the data model by avoiding repeated date and time attributes within the fact table.
</p>

#### Date Table
<p align="justify">
The Date table was designed to provide a continuous calendar covering the entire date range present in the fact table. This ensures that dates with no recorded orders are still represented, allowing time-intelligence calculations to operate correctly.
</p>

#### Time Table
<p align="justify">
A dedicated Time table was created to support analysis of ordering patterns throughout the day. The original timestamps contain seconds (h:mm:ss AM/PM), which provide unnecessary granularity for this analysis. Therefore, seconds were removed and the time values were standardised to h:mm AM/PM. The table was generated by creating a sequence representing every minute within a 24-hour period (0–1439). Each minute was then converted into a time value using:

<code>Time = TIME(FLOOR('Time'[Minute] / 60, 1), MOD('Time'[Minute], 60), 0)</code>

Time-slot columns were subsequently generated using a parameterized formula, where X represents the desired slot duration in minutes:

<code>X-min timeslot = FLOOR('Time'[Minute] / X, 1) * X / 1440</code>

This approach allows different time-slot granularities to be generated by changing X. For example, setting X = 10 produces 10-minute intervals such as 6:30 PM, 6:40 PM, 6:50 PM, and so on.
</p>

<br>

The resulting Date and Time tables contains the following fields:

| Table                | Columns                                                        |
| -------------------- | -------------------------------------------------------------- |
| **Date**            | `date`, `year`, `month`, `month_name`, `month_abrev`, `quarter`, `day`, `day_abrev`, `day_number`, `week` |
| **Time**            | `time`, `30-min timeslot`, `60-min timeslot` |


#### Relationship Overview
<p align="justify">
The data model follows a <b>snowflake-schema structure</b>, with <code>OrderLine</code> serving as the primary fact table and the remaining tables providing descriptive and lookup information. The <code>Pizza-Ingredients</code> table acts as a bridge table to resolve the many-to-many relationship between pizzas and ingredients.
</p>

<p align="justify">
<b>Note:</b> Since the analysis requires identifying the top and bottom ingredients by order quantity, the cross-filter direction between the <code>Pizza</code> and <code>Pizza–Ingredient</code> tables is configured as <b>Both</b>. This allows filters applied to ingredients to propagate through the bridge table to the <code>Orderline</code> fact table, enabling order quantities to be correctly aggregated by ingredient.
</p>

<br>

| From Table | Relationship | To Table | Key |
|---|---|---|---|
| **Order** | One-to-Many (1:*) | **Orderline** | `Order[order_id]` → `Orderline[order_id]` |
| **Date** | One-to-Many (1:*) | **Order** | `Date[Date]` → `Order[date]` |
| **Time** | One-to-Many (1:*) | **Order** | `Time[Time]` → `Order[time]` |
| **Pizza** | One-to-Many (1:*) | **Orderline** | `Pizza[pizza_id]` → `Orderline[pizza_id]` |
| **Pizza** | One-to-Many (1:*) | **Pizza–Ingredient** | `Pizza[pizza_id]` → `Pizza–Ingredient[pizza_id]` |
| **Ingredient** | One-to-Many (1:*) | **Pizza–Ingredient** | `Ingredient[ingredient_id]` → `Pizza–Ingredient[ingredient_id]` |

<br>

<p align="center">
  <img width="817" height="509" alt="image" src="https://github.com/user-attachments/assets/f456fc14-d397-443f-b289-ed0ceaa3b5d3" />
  <br>
  <em>Figure 1. Data Model</em>
</p>

## 3. Insights & Recommendations

#### Multi-Item Orders
<p align="justify">
Approximately two-thirds of orders contain multiple pizza types, with these orders generating the majority of revenue. This suggests an opportunity to increase average order value by encouraging multi-item purchases through:

<ul>
  <li>Pizza bundles based on common co-purchase patterns.</li>
  <li>Targeted discounts on selected pizzas or categories to encourage additional purchases or support inventory management.</li>
  <li>Complementary products such as soft drinks, sides, and appetizers to provide additional purchasing opportunities.</li>
</ul>
</p>


#### Pizza Size
<p align="justify">
Large pizzas are frequently purchased, but the Classic category shows a stronger preference for Small pizzas. The company could investigate size-based pricing strategies that make upgrading to Medium or Large more attractive. For example, reducing the price gap between Small and Medium pizzas could encourage customers to upgrade. Any pricing changes should be evaluated against profitability before implementation.
</p>


#### Order Frequency
<p align="justify">
The average time between consecutive orders is 24.58 minutes, but this is distorted by gaps between operating days. For example, an 806-minute gap occurs between the final order on January 1 and the first order on January 2. Therefore, the average time between consecutive orders should be calculated within the same operating day only to provide a more meaningful measure of order frequency.
</p>


#### Sales Trends
<p align="justify">
Orders and revenue generally increase throughout the week, peak on Friday, and decline over the weekend, with Sunday recording the lowest activity.
A decline is also observed between September and December. However, only one year of data is available so further historical data is required to determine whether this represents a recurring seasonal trend.
</p>

## 4. Dashboard Screenshots
<p align="center">
  <img width="896" height="503" alt="image" src="https://github.com/user-attachments/assets/a562f2fa-3ef3-45cf-b6fe-a28a9221a688" />
  <br>
  <em>Figure 2. Order KPI page</em>
</p>

<br>

<p align="center">
  <img width="899" height="504" alt="image" src="https://github.com/user-attachments/assets/32dfca5f-642e-4ad0-8824-059933f9db24" />
  <br>
  <em>Figure 3. Sales Trend page</em>
</p>

<br>

<p align="center">
  <img width="897" height="505" alt="image" src="https://github.com/user-attachments/assets/17b90ff7-efb8-4f6d-8d47-d295900a9cf7" />
  <br>
  <em>Figure 4. Pizza page</em>
</p>

<br>

<p align="center">
  <img width="615" height="500" alt="image" src="https://github.com/user-attachments/assets/91eef40a-c030-4108-8422-2a20ae8539bb" />
  <br>
  <em>Figure 5. Ingredient page</em>
</p>
