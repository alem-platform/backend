# frappuccino

## Learning Objectives

- SQL
- PostgreSQL
- CRUD
- ERD

## Abstract

In this project, you will rewrite the existing [hot-coffee](https://github.com/alem-platform/foundation/tree/main/hot-coffee) project to work with a PostgreSQL database. The main task is to modify the current handlers in hot-coffee so that they execute SQL queries instead of working with JSON.

In addition to refactoring the existing handlers, you will add several new ones related to aggregation and reporting. Throughout this project, you will learn to perform basic and advanced SQL operations and explore aggregation tools provided by PostgreSQL.

## Context

Systems become more complex and modernized over time, and the hot-coffee project is no exception. While using a JSON-based database is convenient, it is not scalable and makes maintenance difficult for other developers. Moving to a PostgreSQL database is a better solution.

Fortunately, since the project was initially built using a layered architecture, the transition will be easier. You will need to just change Data Access Layer (Repositories).

As part of this project, you must design tables correctly and define relationships between them appropriately.

## General Criteria

- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will be graded `0` automatically.
- Your program MUST be able to compile successfully.
- Your program MUST not exit unexpectedly (any panics: `nil-pointer dereference`, `index out of range` etc.). If so, you will get `0` during the defence.
- Only built-in packages are allowed, except for the PostgreSQL driver. Using any other external packages will result in a grade of 0.
- The project MUST be run by the following command in the project's root directory:

```shell
$ docker compose up
```

## Mandatory Part

### ERD (Entity-Relationship Diagram) Requirements

Before implementing PostgreSQL database, you must design your [ERD](https://www.lucidchart.com/pages/er-diagrams). The diagram should be based on the original hot-coffee data models but enhanced with proper database relationships.

Your database schema **must** utilize the following specific data types. Here are some hints about where they might be useful, but the final implementation is up to you:

#### JSONB
Areas of implementation:
- Menu item customization options
- Order special instructions
- Customer preferences
- Additional item metadata

#### Arrays
Areas of implementation:
- Item categories/tags
- Allergen information
- Multiple order statuses
- Ingredient substitutes

#### ENUM
Areas of implementation:
- Order status values
- Payment methods
- Item sizes
- Staff roles

#### Timestamp with time zone
Areas of implementation:
- Order dates
- Inventory updates
- Price change history
- Staff schedules

### Core Tables

1. **orders**
    - Main order information table
    - Needs to track status changes
    - Must include proper timestamps
    - Links to customer details
    - Should be able to track total amount

2. **order_items**
    - Individual items in each order
    - Connects orders with menu items
    - Keeps track of quantity and price at time of order
    - Could store customization information

3. **menu_items**
    - Available products for sale
    - Should include basic product information
    - Needs category or type classification
    - Must track pricing
    - Connects to required ingredients

4. **menu_item_ingredients**
    - Junction table between menu items and ingredients
    - Stores recipe quantities
    - Important for inventory management
    - Helps calculate if item can be made

5. **inventory**
    - Tracks all available ingredients
    - Monitors stock levels
    - Helps with reordering
    - Records unit types and amounts

6. **order_status_history**
    - Keeps track of order state changes
    - Important for order lifecycle
    - Helps with analytics

7. **price_history**
    - Tracks menu item price changes
    - Useful for revenue analysis
    - Important for historical reporting

8. **inventory_transactions**
    - Records all inventory changes
    - Helps track ingredient usage
    - Important for stock management

> While based on the original JSON structure, your database design should take advantage of PostgreSQL's relational nature rather than just mimicking the JSON structure directly.



Your task is to rewrite the existing endpoints in the **hot-coffee** project to work with a PostgreSQL database. To do this, you may use a third-party PostgreSQL driver as part of your solution.

### Containerization Guide

Since adding a database dependency to your project makes it more challenging for auditor to run and test it, you need to containerize your service and database into separate containers.
Don't worry about it, everything made up for you. Just use ready `Dockerfile` and `docker-compose.yml` files provided for you [here](https://github.com/alem-platform/backend/tree/main/frappuccino). Place everything in the root folder of the project and then follow the instructions:

1. Docker compose file requires you to create `init.sql` that should be placed in the root folder of the project, near previously downloaded files. It should contain all `SQL` code for creating necessary tables and relations
2. Adjust the `Dockerfile` provided to run your application - edit `CMD` command accordingly 
3. This will allow the project to be started with a single command:
```bash
docker-compose up
```

## Important Notes

- Your API will be available at *localhost:8080*
- Use these database connection settings in your code:
    - **Host**: db
    - **Port**: 5432
    - **User**: latte
    - **Password**: latte
    - **Database**: frappuccino
- The init.sql file will automatically create your tables when the container starts

### New Endpoints

As part of the task, the following endpoints must be rewritten to work using SQL queries:

- **Orders:**

    - `POST /orders`: Create a new order.
    - `GET /orders`: Retrieve all orders.
    - `GET /orders/{id}`: Retrieve a specific order by ID.
    - `PUT /orders/{id}`: Update an existing order.
    - `DELETE /orders/{id}`: Delete an order.
    - `POST /orders/{id}/close`: Close an order.
- **Menu Items:**

    - `POST /menu`: Add a new menu item.
    - `GET /menu`: Retrieve all menu items.
    - `GET /menu/{id}`: Retrieve a specific menu item.
    - `PUT /menu/{id}`: Update a menu item.
    - `DELETE /menu/{id}`: Delete a menu item.
- **Inventory:**

    - `POST /inventory`: Add a new inventory item.
    - `GET /inventory`: Retrieve all inventory items.
    - `GET /inventory/{id}`: Retrieve a specific inventory item.
    - `PUT /inventory/{id}`: Update an inventory item.
    - `DELETE /inventory/{id}`: Delete an inventory item.
- **Aggregations:**

    - `GET /reports/total-sales`: Get the total sales amount.
    - `GET /reports/popular-items`: Get a list of popular menu items.

In addition to rewriting the existing endpoints, you must also develop the following new ones:

#### 1. Number of ordered items
`GET /orders/numberOfOrderedItems?startDate={startDate}&endDate={endDate}`: Returns a list of ordered items and their quantities for a specified time period. If the `startDate` and `endDate` parameters are not provided, the endpoint should return data for the entire time span.
##### **Parameters**:
- `startDate` _(optional)_: The start date of the period in `YYYY-MM-DD` format.
- `endDate` _(optional)_: The end date of the period in `YYYY-MM-DD` format.

Response example:
```json
GET /orders/numberOfOrderedItems?startDate=10.11.2024&endDate=11.11.2024
HTTP/1.1 200 OK
Content-Type: application/json

{
  "latte": 109,
  "muffin": 56,
  "espresso": 120,
  "raff": 0,
  ...
}
```

#### 2. Full Text Search Report
`GET /reports/search`: Search through orders, menu items, and customers with partial matching and ranking.

##### Parameters:
- `q` _(required)_: Search query string
- `filter` _(optional)_: What to search through, can be multiple values comma-separated:
   * orders (search in customer names and order details)
   * menu (search in item names and descriptions)
   * all (default, search everywhere)
- `minPrice` _(optional)_: Minimum order/item price to include
- `maxPrice` _(optional)_: Maximum order/item price to include

Response example:
```json
GET /reports/search?q=chocolate cake&filter=menu,orders&minPrice=10
HTTP/1.1 200 OK
Content-Type: application/json

{
   "menu_items": [
       {
           "id": "12",
           "name": "Double Chocolate Cake",
           "description": "Rich chocolate layer cake",
           "price": 15.99,
           "relevance": 0.89
       },
       {
           "id": "15",
           "name": "Chocolate Cheesecake",
           "description": "Creamy cheesecake with chocolate",
           "price": 12.99,
           "relevance": 0.75
       }
   ],
   "orders": [
       {
           "id": "1234",
           "customer_name": "Alice Brown",
           "items": ["Chocolate Cake", "Coffee"],
           "total": 18.99,
           "relevance": 0.68
       }
   ],
   "total_matches": 3
}
```

#### 3. Ordered items by period
`GET /reports/orderedItemsByPeriod?period={daily|monthly}&month={month}`: Returns the number of orders for the specified period, grouped by day within a month or by month within a year. The `period` parameter can take the value `daily` or `monthly`. The `month` parameter is optional and used only when `period=daily`.
##### **Parameters**:
- `period` _(required)_:
    - `daily`: Groups data by day within the specified month.
    - `monthly`: Groups data by month within the specified year.
- `month` _(optional)_: Specifies the month (e.g., `october`). Used only if `period=daily`.
- `year` _(optional)_: Specifies the year. Used only if `period=monthly`.

Response example:
```json
GET /reports/orderedItemsByPeriod?period=daily&month=october
HTTP/1.1 200 OK
Content-Type: application/json

{
    "period": "daily",
    "month": "october",
    "orderedItems": [
        { "1": 109 },
        { "2": 234 },
        { "3": 198 },
        { "4": 157 },
        { "5": 223 },
        { "6": 143 },
        { "7": 256 },
        { "8": 199 },
        { "9": 275 },
        { "10": 187 },
        { "11": 234 },
        { "12": 150 },
        { "13": 178 },
        { "14": 210 },
        { "15": 202 },
        { "16": 190 },
        { "17": 260 },
        { "18": 215 },
        { "19": 240 },
        { "20": 180 },
        { "21": 300 },
        { "22": 250 },
        { "23": 199 },
        { "24": 210 },
        { "25": 220 },
        { "26": 190 },
        { "27": 170 },
        { "28": 260 },
        { "29": 230 },
        { "30": 210 },
        { "31": 180 }
    ]
}

```

Response example:
```json
GET /reports/orderedItemsByPeriod?period=monthly&year=2023
HTTP/1.1 200 OK
Content-Type: application/json

{
    "period": "monthly",
    "year": "2023",
    "orderedItems": [
        { "january": 6528 },
        { "february": 7324 },
        { "march": 8452 },
        { "april": 7890 },
        { "may": 9103 },
        { "june": 8675 },
        { "july": 9234 },
        { "august": 8820 },
        { "september": 9345 },
        { "october": 8901 },
        { "november": 8123 },
        { "december": 9576 }
    ]
}
```

#### 4. Get leftovers
`GET /inventory/getLeftOvers?sortBy={value}&page={page}&pageSize={pageSize}`: Returns the inventory leftovers in the coffee shop, including sorting and pagination options.
##### **Parameters**:
- `sortBy` _(optional)_: Determines the sorting method. Can be either:
    - `price`: Sort by item price.
    - `quantity`: Sort by item quantity.
- `page` _(optional)_: Current page number, starting from 1.
- `pageSize` _(optional)_: Number of items per page. Default value: `10`.
##### **Response**:
- Includes:
    - A list of leftovers sorted and paginated.
    - `currentPage`: The current page number.
    - `hasNextPage`: Boolean indicating whether there is a next page.
    - `totalPages`: Total number of pages.

Response example:
```json
GET /inventory/getLeftOvers?sortBy=quantity?page=1&pageSize=4
HTTP/1.1 200 OK
Content-Type: application/json

{
    "currentPage": 1,
    "hasNextPage": true,
    "pageSize": 4,
    "totalPages": 10,
    "data": [
        {
            "name": "croissant",
            "quantity": 109,
            "price": 950
        },
        {
            "name": "sugar",
            "quantity": 93,
            "price": 50
        },
        {
            "name": "muffin",
            "quantity": 63,
            "price": 350
        },
        {
            "name": "milk",
            "quantity": 1,
            "price": 200
        }
    ]
}
```

#### 5. Bulk Order Processing
`POST /orders/batch-process`: Process multiple orders simultaneously while ensuring inventory consistency. This endpoint must handle concurrent orders and maintain data integrity using transactions.

Request Body:
```json
{
   "orders": [
       {
           "customer_name": "Alice",
           "items": [
               {
                   "menu_item_id": 1,
                   "quantity": 2
               },
               {
                   "menu_item_id": 3,
                   "quantity": 1
               }
           ]
       },
       {
           "customer_name": "Bob",
           "items": [
               {
                   "menu_item_id": 2,
                   "quantity": 1
               }
           ]
       }
   ]
}
```

Response example:

```json
{
    "processed_orders": [
        {
            "order_id": 123,
            "customer_name": "Alice",
            "status": "accepted",
            "total": 15.50
        },
        {
            "order_id": 124,
            "customer_name": "Bob",
            "status": "rejected",
            "reason": "insufficient_inventory"
        }
    ],
    "summary": {
        "total_orders": 2,
        "accepted": 1,
        "rejected": 1,
        "total_revenue": 15.50,
        "inventory_updates": [
            {
                "ingredient_id": 1,
                "name": "Coffee Beans",
                "quantity_used": 100,
                "remaining": 2400
            }
        ]
    }
}
```

## Initial Setup

You need to create an `init.sql` file that will set up the complete database structure. This file must include:

1. **Create Type Statements**
    - Define ENUMs for order status, units of measurement, etc.
    - Create any composite types needed

2. **Create Table Statements**
    - Core tables (orders, menu_items, inventory)
    - Junction tables (menu_item_ingredients)
    - History tables (order_status_history, price_history)
    - Include all constraints and relationships

3. **Create Index Statements**
    - At least 2 indexes over all tables
    - Indexes for frequently queried columns
    - Full text search indexes
    - Composite indexes where needed

4. **Mock Data Inserts**

    Must include sufficient test data:

    - At least 10 menu items with different prices and categories
    - At least 20 inventory items with various quantities
    - At least 30 orders in different statuses
    - Price history spanning several months
    - Order status history showing different state transitions
    - Inventory transactions showing stock movements

5. **Testing Coverage**

    Your mock data must allow testing of:
    - Full text search functionality
    - Date range queries
    - Status transitions
    - Inventory calculations
    - Price history tracking

Important:
- Tables must be created in correct order (referenced tables first)
- All foreign key relationships must be properly defined
- Mock data must be consistent (e.g., orders reference existing menu items)
- Initial data should be realistic for a coffee shop context

## Guidelines from Author

In the initial stages of working with tables and relationships, it can be challenging to understand and organize everything. 

Therefore, you may use visualising tools like **pgAdmin** for actually *seeing* what you're doing.

Also, start with ERD and make it as detailed as you can, it would really help you to connect all the dots.

## Author

This project has been created by:

Askaruly Nurislam, alumni of Alem School

Contacts:

- Email: [askaruly@hotmail.com](mailto:askaruly@hotmail.com)
- [GitHub](https://github.com/darwin939/)