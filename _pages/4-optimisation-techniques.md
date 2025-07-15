---
title: Optimisation Techniques
author: Aya Elsayed, Rhythm Patel
category: SQLAlchemy Workshop
date: 2024-01-05
layout: post
---

Let's talk about some techniques that will help to optimise your code when using ORMs with SQLAlchemy.
You can checkout the `step-3-orms` branch to follow along this interactive section.

## Relationship Loading Techniques

SQLAlchemy provides extensive configuration over how related objects mapped using `relationship()` get loaded when querying. We can configure this by using [`relationship.lazy`](https://docs.sqlalchemy.org/en/20/orm/relationship_api.html#sqlalchemy.orm.relationship.params.lazy) parameter of `relationship()`.

There are 3 categories of relationship loading techniques: **lazy**, **eager**, and **no** loading.

### Lazy Loading

[Lazy loading](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html#lazy-loading) occurs when objects are returned from a query without loading the related objects initially. When the related object is first accessed, additional **SELECT** statements are emitted to access that particular related object. This is the default loading technique when using SQLAlchemy ORMs.

Thus in lazy loading, for a Parent and Child relationship, if we perform `select(Parent)`, only 1 **SELECT** statement will be emmitted and data will be fetched for attributes of `Parent`. If we then try to access the child, say `print(parent.child)`, another **SELECT** statement will be emmitted to fetch the attributes of `Child`.

Lazy loading is helpful when you know that you will not use all the related or child objects, or even if you will not require them instantly. It will also be beneficial on the "collection side" of a many-to-one relationship. Otherwise, if you use eager loading, SQLAlchemy will try to **JOIN** the tables to fetch all data at once, which can be expensive. We'll discuss eager loading in detail in a bit.

We've used eager loading for the `Customer` and `Address` relationship. In order to configure lazy loading, use `lazy="select"` in the `relationship()` construct. Let's convert it to lazy loading to see what effect it can have when fetching all customers.

##### marketsvc/db/customer.py

```py
class Customer(Base):
    ...

    address: Mapped["Address"] = relationship(
        back_populates="customer", lazy="select"
    )  # one to one
```

##### marketsvc/db/address.py

```py
class Address(Base):
    ...

    customer: Mapped["Customer"] = relationship(
        back_populates="address",
        lazy="select",
    )  # one to one
```

Let's add a `print` statement in `get_customers()` to access the related object, `Address`, for each of our customers within the session.

##### marketsvc/db_accessor.py

```py
def get_customers():
    with Session(engine) as session:
        stmt = select(Customer)
        result = session.execute(stmt)
        customers = result.scalars().all()
        for customer in customers:
            print(customer.address)

        return customers
```

You can run this by using `./run.sh customers`.

##### SQLAlchemy logs

```sql
BEGIN (implicit)
SELECT customer.id, customer.name, customer.address_id  FROM customer
[generated in 0.00023s] ()
SELECT address.id AS address_id, address.flat_number AS address_flat_number, address.post_code AS address_post_code FROM address WHERE address.id = ?
[generated in 0.00016s] (1,)
SELECT address.id AS address_id, address.flat_number AS address_flat_number, address.post_code AS address_post_code  FROM address WHERE address.id = ?
[cached since 0.001042s ago] (2,)
SELECT address.id AS address_id, address.flat_number AS address_flat_number, address.post_code AS address_post_code  FROM address WHERE address.id = ?
[cached since 0.001606s ago] (3,)
ROLLBACK
```

Here, the first **SELECT** statement only fetches the attributes linked to `Customer` i.e. `id`, `name`, and `address_id`. Since we added `print` to print address details for each customer, we have 3 more **SELECT** statements to fetch address data for a particular `address_id`, as seen with the **WHERE** clause and the bound parameter. This occurs for **each** customer.

> ##### What if?
>
> What happens if there are 100 customers? What about 1000? Or, what if we have 100,000 customers?
> Some real world applications have millions of customers.
> How many **SELECT** statements would be emitted to fetch customer and address information?
{: .block-warning }

This is a well-known problem known as the **N+1 problem**.
Initially, one **SELECT** statement is emitted to load the result collection of parent objects.
Then, for each parent object, an additional **SELECT** is emitted to load attributes of each related child object.
Consequently, for N parent objects, we would have **N + 1 SELECT** statements.

> ##### Test Your Understanding
>
> For this scenario to fetch customer information, we require the `Address` information of all customers.
> Lazy loading is not a good strategy here.
> It becomes beneficial when we know we wouldn't be needing all the child objects.
> Is there a place in our service where lazy loading might be helpful?
{: .block-tip }

### Eager Loading

We can fix the N+1 problem by using **eager loading**, where the related objects are loaded with the parent object up front.
The child objects are automatically loaded along with its parent object.

The ORM accomplishes this by consolidating a **JOIN** to the **SELECT** statement to load the related objects simultaneously, or by emitting additional **SELECT** statements to load the related child objects.

Eager loading is advantageous when we want to fetch information about all or numerous child objects along with its parent object.
It's also a good practice to use this to reduce further queries and alleviate further load to the database.

As a result, eager loading is the right choice for `Customer` and `Address` relationship.

[Joined Eager Loading](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html#joined-eager-loading) is the most prominent type of eager loading.
It applies a **JOIN** to the given **SELECT** statement to that related objects are loaded in the same result.

Let's update `relationship()` in both `Customer` and `Address` to `lazy="joined"`, and run the request to get the customer information.

Once again, run this by using `./run.sh customers`.

##### SQLAlchemy logs

```sql
BEGIN (implicit)
SELECT customer.id, customer.name, customer.address_id, address_1.id AS id_1, address_1.flat_number, address_1.post_code
FROM customer LEFT OUTER JOIN address AS address_1 ON address_1.id = customer.address_id
[generated in 0.00075s] ()
ROLLBACK
```

Here, you can see how we only emit **1** query to the database.
The query has a **JOIN** to join the two tables and fetch all related information at once.
We've also successfully avoided the N+1 problem.

### No Loading

No loading means to disable loading on a relationship.
The child object is empty and is never loaded, or an error is raised with its accessed. No loading is used to prevent unwanted lazy loads.
You can trigger this with the [`lazy="raise"`](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html#prevent-lazy-with-raiseload) parameter.

> ##### Try It Yourself
>
> There are more [relationship loading techniques](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html), but the ones described above are the most popular.
> Try experimenting with different queries yourself and see the generated SQL statements in the logs.
{: .block-tip }

## SQL Functions

You can also use SQL functions including aggregate functions while working with ORMs. We can create [`Function`](https://docs.sqlalchemy.org/en/20/core/functions.html#sqlalchemy.sql.functions.Function) objects by using the [`func`](https://docs.sqlalchemy.org/en/20/core/sqlelement.html#sqlalchemy.sql.expression.func) object, which acts as a factory.

Let's use the `SUM()` function to get the total cost of an order in the SQL query itself, rather than querying the individual item prices and summing them up in Python.

##### marketsvc/db_accessor.py

```py
from sqlalchemy.sql import func

...

def get_total_cost_of_an_order(order_id):
    with Session(engine) as session:
        result = session.execute(
            select(func.sum(Item.price * OrderItems.quantity))
            .join(Orders.order_items)
            .join(OrderItems.item)
            .where(Orders.id == order_id)
        )
        total_cost = result.scalar()

        return total_cost
```

Hit the `/api/order_total` endpoint by using the shell script with a argument that indicates the `order_id` of the order you wish to get the total of.

```sh
./run.sh ordertotal 1
```

##### SQLAlchemy logs

```sql
BEGIN (implicit)
SELECT sum(item.price * order_items.quantity) AS sum_1
FROM orders JOIN order_items ON orders.id = order_items.order_id JOIN item ON item.id = order_items.item_id
WHERE orders.id = ?
[generated in 0.00024s] (1,)
ROLLBACK
```

Here, we're directly using the `SUM()` aggregate function and returning a single value from our SQL query. Similarly, we can use any SQL functions while using ORMs depending on our use case.

## Hybrid Attributes

[Hybrid attributes](https://docs.sqlalchemy.org/en/20/orm/extensions/hybrid.html) have different behaviours at class and instance level, hence the name "hybrid". This is best explained with an example.

### Hybrid Properties

Let us consider the `order_items` table.
The `quantity` of an order is defined here and the `price` of an item is defined in the `items` table.
Whenever we want to get the total cost of an item, we have to multiply item price and quantity for each item in the order.

Alternatively, we could introduce a hybrid property that represents the total cost of an item.
We can do this with the [`@hybrid_property`](https://docs.sqlalchemy.org/en/20/orm/extensions/hybrid.html#sqlalchemy.ext.hybrid.hybrid_property) decorator.

##### marketsvc/db/order_items.py

```py
from sqlalchemy.ext.hybrid import hybrid_property

class OrderItems(Base):
    ...

    @hybrid_property
    def item_total(self):
        return self.item.price * self.quantity
```

Here, the `item_total` property returns the product of item's price and the quantity of that item bought. On instances of `OrderItems`, the multiplication occurs in Python.

Thus, in `as_dict()` method of `Orders`, where we return the information of an order, we can directly get the item's total by `order_item.item_total`. This makes it much easier to access the item total.

##### marketsvc/db/orders.py

```py
def as_dict(self):
    return {
        ...
        "items": [
            {
                ...
                "total": order_item.item_total,
            }
            for order_item in self.order_items
        ],
    }
```

### Hybrid Expressions

To use hybrid attributes in the SQL query, we need to use the [`hybrid_property.expression()`](https://docs.sqlalchemy.org/en/20/orm/extensions/hybrid.html#sqlalchemy.ext.hybrid.hybrid_property.expression) modifier.
This is because its SQL expression must be differentiated from the Python expression.

##### marketsvc/db/order_items.py

```py
from sqlalchemy.ext.hybrid import hybrid_property

class OrderItems(Base):
    ...

    @hybrid_property
    def item_total(self):
        return self.item.price * self.quantity

    @item_total.expression
    @classmethod
    def item_total(cls):
        return Item.price * cls.quantity
```

Above, we've also added an expression modifier to the `item_total` hybrid property.
Notice how this function is a class method and operates on the `OrderItems` and `Item` classes, similar to how we might do in the `select()` query.

Consequently, our query to get the total cost of an order becomes even simpler. Here, we have directly accessed the hybrid expression in the query.

##### marketsvc/db_accessor.py

```py
def get_total_cost_of_an_order(order_id):
    with Session(engine) as session:
        result = session.execute(
            select(func.sum(OrderItems.item_total))
            .join(Orders.order_items)
            .join(OrderItems.item)
            .where(Orders.id == order_id)
        )
        total_cost = result.scalar()

        return total_cost
```

`OrderItems.item_total` would be equivalent to `Item.price * OrderItems.quantity` used earlier.

You might notice hybrid properties are similar to Python's `@property` decorator. We can also define [setters](https://docs.sqlalchemy.org/en/20/orm/extensions/hybrid.html#sqlalchemy.ext.hybrid.hybrid_property.setter) on hybrid properties but it doesn't make sense to 'set' the item total in our case.

&nbsp;

We now have explored some ways how we can optimise our code using ORMs. There are many more techniques that would be hard to cover in a short span of time.
What we've discussed about SQLAlchemy is just the tip of the iceberg. Feel free to browse the SQLAlchemy [documentation](https://docs.sqlalchemy.org/en/20/index.html) to find ways to make your life easier!

Having said that, let's switch gears and look at asyncio, and how we can utilise it in our service.

> ##### Kudos!
>
> 🙌 You have now reached the `step-4-optimisations` part of the tutorial. If not, checkout that branch and continue from there:
>```sh
>git checkout step-4-optimisations
>```
{: .block-tip }

&nbsp;
