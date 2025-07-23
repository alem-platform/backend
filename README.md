
# hot-coffee-sql

## Learning Objectives

- SQL
- PostgreSQL
- CRUD
- ERD

## Abstract

In this project, you will rewrite the existing [hot-coffee](https://github.com/alem-platform/foundation/tree/main/hot-coffee) project to work with a PostgreSQL database. The main task is to modify the current handlers in hot-coffee so that they execute SQL queries instead of working with JSON.

In addition to refactoring the existing handlers, you will add several new ones related to aggregation and reporting. Throughout this project, you will learn to perform basic SQL operations and explore aggregation tools provided by PostgreSQL.

## Context

Systems become more complex and modernized over time, and the hot-coffee project is no exception. While using a JSON-based database is convenient, it is not scalable and makes maintenance difficult for other developers. Moving to a PostgreSQL database is the better solution.

Fortunately, since the project was initially built using a layered architecture, the transition will be easier. You will need to add a new Data Access Layer (Repositories) and integrate it into the project.

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

Your task is to rewrite the existing endpoints in the **hot-coffee** project to work with a PostgreSQL database. To do this, you may use a third-party PostgreSQL driver as part of your solution.

You need to create the following tables: `user`, `order`, `menu`, `menu_item`, `inventory`, and `ingredient`, and define the relationships between them. You are not limited to only these tables; you may create additional ones if necessary for implementing the required functionality. The design and relationships of the tables are at your discretion.

Pay special attention to designing the relationships between tables (e.g., one-to-many, one-to-one). For instance, the `menu` and `menu_item` tables will have a "one-to-many" relationship, meaning the `menu_item` table should include a column with the following constraint:
```sql
menu_id INT NOT NULL CONSTRAINT fk_menu FOREIGN KEY(menu_id) REFERENCES menu(id)
```
Plan your data structure carefully to ensure the project's scalability and maintainability.

Since adding a database dependency to your project makes it more challenging for auditor to run and test it, you need to containerize your service and database into separate containers and write a `docker-compose.yml` file. This will allow the project to be started with a single command:
```bash
docker compose up
```

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
`GET /numberOfOrderedItems?startDate={startDate}&endDate={endDate}`: Returns a list of ordered items and their quantities for a specified time period. If the `startDate` and `endDate` parameters are not provided, the endpoint should return data for the entire time span.
##### **Parameters**:
- `startDate` _(optional)_: The start date of the period in `YYYY-MM-DD` format.
- `endDate` _(optional)_: The end date of the period in `YYYY-MM-DD` format.

response:
```shell
GET /numberOfOrderedItems?startDate=10.11.2024&endDate=11.11.2024
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

#### 2. Cash flow
`GET /cashFlow?startDate={startDate}&endDate={endDate}`: Returns the total sales amount for a specified period. If the `startDate` and `endDate` parameters are not provided, the endpoint should return data for the entire time span.
##### **Parameters**:
- `startDate` _(optional)_: The start date of the period in `YYYY-MM-DD` format.
- `endDate` _(optional)_: The end date of the period in `YYYY-MM-DD` format.

response:
```json
GET /cashFlow?startDate=10.11.2024&endDate=11.11.2024`
HTTP/1.1 200 OK
Content-Type: application/json

{
  "cashFlow": 159940.56,
}
```

#### 3. Ordered items by period
`GET /orderedItemsByPeriod?period={day|month}&month={month}`: Returns the number of orders for the specified period, grouped by day within a month or by month within a year. The `period` parameter can take the value `day` or `month`. The `month` parameter is optional and used only when `period=day`.
##### **Parameters**:
- `period` _(required)_:
    - `day`: Groups data by day within the specified month.
    - `month`: Groups data by month within the specified year.
- `month` _(optional)_: Specifies the month (e.g., `october`). Used only if `period=day`.
- `year` _(optional)_: Specifies the year. Used only if `period=month`.

response:
```json
GET /orderedItemsByPeriod?period=day&month=october
HTTP/1.1 200 OK
Content-Type: application/json

{
    "period": "day",
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

response:
```json
GET /orderedItemsByPeriod?period=month&year=2023
HTTP/1.1 200 OK
Content-Type: application/json

{
    "period": "month",
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
`GET /getLeftOvers?sortBy={value}&page={page}&pageSize={pageSize}`: Returns the inventory leftovers in the coffee shop, including sorting and pagination options.
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

response example:
```json
GET /getLeftOvers?sortBy=quantity?page=1&pageSize=4
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


Since the auditor will not have access to your local database and its data, you need to create a migration file named `init.sql`. This file should include:

1. Scripts to Create All Required Tables
    Define the structure of the tables (`user`, `order`, `menu`, `menu_item`, `inventory`, `ingredient`, etc.) in SQL, including column definitions, data types, and constraints.

2. Definition of Relationships Between Tables
    Ensure that relationships such as `FOREIGN KEY` constraints are properly defined.
3. Mock Data
	Include sample data for testing purposes:
	- Test users
	- Sample orders
	- Menu items
	- Inventory records
	- Ingredients

## Guidelines from Author

In the initial stages of working with tables and relationships, it can be challenging to understand and organize everything. Therefore, it is recommended to use **pgAdmin**

## Author

This project has been created by:

Askaruly Nurislam, alumni of Alem school

Contacts:

- Email: [askaruly@hotmail.com](mailto:askaruly@hotmail.com)
- [GitHub](https://github.com/darwin939/)