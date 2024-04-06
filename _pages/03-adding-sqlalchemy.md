---
title: Adding SQLAlchemy
author: Aya Elsayed, Rhythm Patel
category: SQLAlchemy Workshop
date: 2024-01-03
layout: post
---

In the last step, we had seen that `db_accessor.py` used `psycopg2`, the PostgreSQL database adapter, to interact with the database.
In this section, we will introduce SQLAlchemy, whose core will act as an abstraction layer to connect with the PostgresSQL database.

To start, checkout the branch `step-2-sqlalchemy`:

```sh
git checkout step-2-sqlalchemy
```

## Engine

Let's have a look at `db/base.py`.

Here we create the `Engine` object, which is the entry point of any SQLAlchemy application.  The `Engine` serves as the main source of DBAPI connections to a given database. We can create the `Engine` with the `create_engine()` method. Here, we have passed in a `URL` object to `create_engine()`, which includes all the necessary information required to connect to a database:
- Type of database, represented by `postgresql`. 
    - This instructs SQLAlchemy to use the PostgreSQL **dialect**, which is a system used to communicate with various kinds of databases and their respective drivers.
- DBAPI, represented by `psycopg2`.
    - **DBAPI** (Python Database API Specification) is a driver that SQLAlchemy uses to connect to a database. It's a "low level" API that lets Python talk to the database.
- Database connection configuration like host, port, database, username, and password.

We've also enabled the `echo` flag, which will log the generated SQL queries by SQLAlchemy. We'll display these logs to explain what happens behind the scenes.

> **Lazy Connecting**
> 
> When `create_engine()` first returns an `Engine` object, it will not reach out to the database yet. It will wait for a task to be performed against the database, such as a SELECT query, and then connect to the database, following the [lazy initialization](https://en.wikipedia.org/wiki/Lazy_initialization) design pattern.


## Connection

Once our `Engine` object is ready, it will be used to connect to the database by providing a unit of connectivity called the `Connection`. It provides services to execute SQL statements and transaction control, and this is how we'll interact with the database. The `Connection` is acquired by `Engine.connect()`.

We don't want to keep a `Connection` running indefinitely, and thus, the recommended way to use `Connection` is with context managers, which will frame the operations inside into a transaction.

### Executing Queries

Here, we see how we can execute queries with SQLAlchemy. For now, we'll be using raw SQL. Let's look at `db_accessor.py`.

```py
def execute_query(query, params=None):
    with engine.connect() as conn:
        result = conn.execute(text(query), params)

        return [row._asdict() for row in result]

rows = execute_query("SELECT * FROM customer")
```

SQLAlchemy logs:
```sql
BEGIN (implicit)
SELECT * FROM customer
[generated in 0.00021s] {}
ROLLBACK
```

The statement is executed with the `Connection.execute()` function, which returns a `Result` that represents an iterable object of resulting rows, depicted by `Row`.

As you can see from the logs, a **ROLLBACK** was emitted at the end. This marked the of the transaction. An automatic rollback occurs when a connection is closed after use, to ensure that the connection is 'clean' for its next use.

Try building and running the other queries yourself, and see the logs to understand what's happening behind the scenes.


### Committing Data

You might have noticed the absense of the **COMMIT** statement from the SQLAlchemy logs. If we want to commit some data, we need to explicitly call `Connection.commit()` inside the block.

```py
def execute_insert_query(query, params=None):
    with engine.connect() as conn:
        result = conn.execute(text(query), params)
        conn.commit()

        return [row._asdict() for row in result]

new_order_id = execute_insert_query("INSERT INTO orders...") # line 104
```

SQLAlchemy logs:
```sql
BEGIN (implicit)
INSERT INTO orders (customer_id, order_time) VALUES (%(customer_id)s, NOW()) RETURNING id
[generated in 0.00041s] {'customer_id': 1}
COMMIT
```

As you can see, `conn.commit()` committed the transaction, and the statement **COMMIT** is logged, as compared to **ROLLBACK** in the previous example. We can then call `conn.commit()` for committing additional statements. This style is called **commit as you go**. Additionally, notice how the parameter `customer_id` is passed into the SQL statement. We'll talk about this later.

> ##### Test Your Understanding
>
> Can you predict the resulting logs when `execute_insert_queries()` is called after this in the `add_new_order_for_customer()` function?
{: .block-tip }

Another way to commit data is to use the context manager `Engine.begin()` instead of `Engine.connect()`. It will declare the whole block to be one transcation block, and will enclose everything inside the transaction with one **COMMIT** at the end. This method is called **begin once**.

```py
with engine.begin() as conn:
    result = conn.execute(text("ISERT INTO orders..."), params)
```

SQLAlchemy logs:
```sql
BEGIN (implicit)
INSERT INTO orders ...
[generated in 0.00041s] {...}
COMMIT
```

Notice there is a **COMMIT** without explicitly writing `conn.commit()`.


If an exception occurs duing the transaction, the changes will be rolled back and a **ROLLBACK** will be displayed instead.


### Parameters

We might want to select specific rows, or insert some data to the table. The `Connection.execute()` function can accept parameters called **bound parameters**. We indicate the presense of parameters in the `text()` construct by using colons, such as `:customer_id`. We can then send the actual value of these parameters as a dictionary in the second argument of `Connection.execute()`, like `{"customer_id": 1}`. Have a look at the code in `add_new_order_for_customer()` function to see how we've used these bound paramters.

> ##### WARNING
> 
> Never put variables directly in the query string. Doing so leaves your code vulnerable to SQL injection attacks. **Always** use parameter binding. Using parameters allows the dialect and DBAPI to correctly handle the input, and enables the driver to have the best performance.
{: .block-danger }

If we want to send multiple sets of parameters, such as insert multiple records in the table, we can pass a list of dictionaries to `Connection.execute()` and send multiple parameter sets. The SQL statement will be executed once for each parameter set. Such example is shown in the `add_new_order_for_customer()` function in `db_accessor.py`.


Now that we've added SQLAlchemy, let's eliminate raw SQL text and introduce ORMs!