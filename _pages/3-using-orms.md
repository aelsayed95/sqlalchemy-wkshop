---
title: Using ORMs and Querying Data
author: Aya Elsayed, Rhythm Patel
category: SQLAlchemy Workshop
date: 2024-01-04
layout: post
---

In the previous step, we've added SQLAlchemy to our codebase, but we still haven't fully leveraged its power.
In this step, we'll create ORM classes to represent database tables.

## ORM

ORM stands for Object Relational Mapping, which lets us map the tables of a database to user-defined Python classes.
The class attributes of the mapped class will be linked to columns of the table.
This structure is called [**Declarative Mapping**](https://docs.sqlalchemy.org/en/20/tutorial/metadata.html#using-orm-declarative-forms-to-define-table-metadata), which simultaneously defines a Python object model and metadata describing the actual tables that exist in a particular database.

Let's update `db/base.py` to add a `Base` class that subclasses `DeclarativeBase`, but remember to keep the engine!

##### marketsvc/db/base.py

```py
from sqlalchemy.orm import DeclarativeBase
...

class Base(DeclarativeBase):
    pass
```

When we create new classes that inherit `Base`, they will each be established as **ORM mapped classes** at the creation time.
Each mapped class will refer to a particular database table.

### Mapped Classes

The foundational unit of SQL when using the ORM is the Mapped Class.
As we learned, individual mapped classes are created by subclassing `Base`.

With this in mind, let's create a mapped class for `Customer` in `db/customer.py`.

##### marketsvc/db/customer.py

```py
from db.base import Base
from sqlalchemy.orm import Mapped, mapped_column


class Customer(Base):
    __tablename__ = "customer"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    name: Mapped[str]
    address_id: Mapped[int]

    def __repr__(self) -> str:
        return f"Customer(id={self.id!r}, name={self.name!r}, address_id={self.address_id!r})"
```

We indicate the name of the table by using the [`__tablename__`](https://docs.sqlalchemy.org/en/20/orm/mapping_api.html#sqlalchemy.orm.DeclarativeBase.__tablename__) class-level attribute.

We declare the columns by adding them as attributes along with a special type called [`Mapped`](https://docs.sqlalchemy.org/en/20/orm/internals.html#sqlalchemy.orm.Mapped), which is driven from type annotations defined [PEP 484](https://peps.python.org/pep-0484/).
The name of each attribute corresponds to the name of the column in the table.
The datatype is inferred from the `Mapped` type, such as `int` becomes `INTEGER`, `str` becomes `VARCHAR`, etc.

For example: `name: Mapped[str]` is the `name` column of the `customer` table of datatype `VARCHAR`. Consequently, we can indicate nullable contraints in the column by `Mapped[str | None]`.

To include additional specification for our attributes, we can use the [`mapped_column()`](https://docs.sqlalchemy.org/en/20/orm/mapping_api.html#sqlalchemy.orm.mapped_column) construct, which will generate `Column` objects. We can use `mapped_column()` without annotations too.
The following example is also valid.

```py
customer_name = mapped_column("name", String, nullable=False)
```

Notice how it's possible for the class attribute to have a different name than the column.
The datatype of the column is inferred from the [String](https://docs.sqlalchemy.org/en/20/core/type_basics.html#sqlalchemy.types.String) SQLAlchemy type object.

We also indicate the presense of the primary key with enabling the `primary_key` boolean parameter.
If there is a composite key constraint, we can represent that by enabling the `primary_key` boolean flag for all the attributes that are a part of the composite key.

In our code, we've set the `autoincrement` parameter to `True`.
This is because our IDs are integers starting from 1, and incremented automatically, i.e., we do not need to assign an integer to `id` while inserting rows.

Mapped classes include a default `__init__()` method that accepts all attribute names as keyword arguments.
This will help us to instantiate objects, which will allow us to insert records in the table.

Finally, we have included a `__repr__()` method which is not necessary, but helps us in debugging.

> ##### Test Your Understanding
>
> With what you've learnt, create the `Address` mapped class for the `address` table in `marketsvc/db/address.py`.
>
> _HINT:_ The `address` table has 3 columns: `id`, `flat_number`, and `post_code`.
> See `marketdb/init_db.sql` for more info.
{: .block-tip }

### Relationships

Amazing! You've now created the `Address` ORM mapped class as well.
As you may remember, the tables `address` and `customer` are related, where the `address_id` column in `customer` is the foreign key to the `id` column in `address`.
Let's represent this relationship in ORMs.

This is how `Address` will look like.

##### marketsvc/db/address.py

```py
from db.base import Base
from sqlalchemy.orm import Mapped, mapped_column, relationship


class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    flat_number: Mapped[int]
    post_code: Mapped[int]

    customer: Mapped["Customer"] = relationship(
        back_populates="address",
        lazy="joined",
    )  # one to one

    def __repr__(self) -> str:
        return f"Address(id={self.id!r}, flat_number={self.flat_number!r}, post_code={self.post_code!r})"
```

Let's also update the `Customer` class to indicate the relationship.

##### marketsvc/db/customer.py

```py
from db.address import Address
from db.base import Base
from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship


class Customer(Base):
    ...
    address_id: Mapped[int] = mapped_column(ForeignKey("address.id"))

    address: Mapped["Address"] = relationship(
        back_populates="customer", lazy="joined"
    )  # one to one
```

We can depict the foreign key constraint by using the [`ForeignKey`](https://docs.sqlalchemy.org/en/20/core/constraints.html#sqlalchemy.schema.ForeignKey) object in `mapped_class()`.
The syntax is `<table_name>.<column_name>`.
Thus, the `address_id` attribute in our `Customer` class is the foreign key to the `id` column in `address` table.

In addition to the column-based attributes, we have a [`relationship()`](https://docs.sqlalchemy.org/en/20/orm/relationship_api.html#sqlalchemy.orm.relationship) construct that represents a linkage between two mapped classes, and by extension, relationship between two tables.

In our case, the `relationship()` construct, along with the `Mapped["Address"]` construct, will be used to examine the table relationships.
As a result, `Customer.address` links `customer` to `address`, and `Address.customer` links `address` to `customer`.
The relationship linkage between the two tables is inferred with the foreign key constraint we had defined previously.

The `lazy="joined"` parameter signifies that we've used joined relationship loading technique, a type of eager loading.
We will talk about this in the next section in detail.

We have applied a singular type (rather than a collection) to the `Mapped` annotation on both sides of the relationship, i.e. `Mapped["Address"]` in `Customer`, and `Mapped["Customer"]` in `Address`.
With this, SQLAlchemy has determined that there is a **One to One** relationship.

> ##### Test Your Understanding
> We've learned how to represent a One to One relationship.
> How do you think we would represent a One to Many relationship?
{: .block-tip }


> ##### TIP
> Here, we have represented a bi-directional relationship with defining `relationship()` on both classes, where both classes know about each other.
> Oftentimes, we don't need the child class to know about the parent class.
> Thus, in such cases, we have a uni-directional relationship, where there is a `relationship()` construct only on the parent class.
{: .block-tip }

To represent our ORM class as a comprehensible response from our service, let's introduce an `as_dict()` method in `Customer`.
This converts the ORM class into a dictionary with a meaningful structure.

Finally, our `Customer` mapped class should look something like this.

##### marketsvc/db/customer.py

```py
from db.address import Address
from db.base import Base
from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship


class Customer(Base):
    __tablename__ = "customer"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    name: Mapped[str]
    address_id: Mapped[int] = mapped_column(ForeignKey("address.id"))

    address: Mapped["Address"] = relationship(
        back_populates="customer", lazy="joined"
    )  # one to one

    def __repr__(self) -> str:
        return f"Item(id={self.id!r}, name={self.name!r}, address_id={self.address_id!r}, address={self.address})"

    def as_dict(self):
        return {
            "name": self.name,
            "address": {
                "flat_number": self.address.flat_number,
                "post_code": self.address.post_code,
            },
        }
```

> ##### TIP
>
> There's a useful SQLAlchemy extension [`Automap`](https://docs.sqlalchemy.org/en/20/orm/extensions/automap.html#) that automatically generates mapped classes for you.
> You can also use a mix of programmatic and auto-generated classes.
{: .block-tip }

## Querying Data

We now have our `Customer` and `Address` ORMs set up.
We can now make _full use_ of the SQL Expression Language and eliminate the need of raw SQL statements, which is extremely intuitive and almost readable in English, thus allowing us to query data like magic!

Let's update the `get_customers()` function in `db_accessor.py`.

##### marketsvc/db_accessor.py

```py
from db.base import engine
from db.customer import Customer
from sqlalchemy import select
from sqlalchemy.orm import Session


def get_customers():
    with Session(engine) as session:
        stmt = select(Customer)
        result = session.execute(stmt)
        customers = result.scalars().all()

        return customers
```

When using ORMs, the `Engine` is managed by the [`Session`](https://docs.sqlalchemy.org/en/20/orm/session_api.html#sqlalchemy.orm.Session) object.
The `Session` is the fundamental transactional and database interactive object for ORMs. It is almost identical to the `Connection` discussed in the previous section.
In fact, `Session` refers to `Connection` internally to emit SQL.
Similarly, the session uses "commit as you go" behaviour.

The [`select()`](https://docs.sqlalchemy.org/en/20/core/selectable.html#sqlalchemy.sql.expression.select) construct generates a [`Select`](https://docs.sqlalchemy.org/en/20/core/selectable.html#sqlalchemy.sql.expression.Select) object that is used for **SELECT** queries.
The `select()` function accepts mapped classes as well as class-level attributes that represent mapped columns. In our case, we're passing in the `Customer` mapped class to `select()`.

The resulting `Select` object, `stmt` in our case, is passed to [`Session.execute()`](https://docs.sqlalchemy.org/en/20/orm/session_api.html#sqlalchemy.orm.Session.execute), which returns a [`Result`](https://docs.sqlalchemy.org/en/20/core/connections.html#sqlalchemy.engine.Result) object.
To get a list of `Customer` rather than a list of `Row`, we call [`scalars()`](https://docs.sqlalchemy.org/en/20/core/connections.html#sqlalchemy.engine.Result.scalars) on the `Result`.
Since we want to return all the customers, we further call `all()`.

Finally, let's update `customers()` in `server.py` to get the `Customer` objects as dictionaries for the response using the `as_dict()` function we defined earlier:

##### marketsvc/server.py

```py
@app.get("/api/customers")
def customers():
    customers = get_customers()
    return (customer.as_dict() for customer in customers)
```

You may now hit the endpoint with:

```sh
./run.sh customers
```

##### SQLAlchemy logs

```sql
BEGIN (implicit)
SELECT customer.id, customer.name, customer.address_id, address_1.id AS id_1, address_1.flat_number, address_1.post_code 
FROM customer LEFT OUTER JOIN address AS address_1 ON address_1.id = customer.address_id
[generated in 0.00019s] ()
ROLLBACK
```

The following will be the value of `customers`:

```py
[Customer(id=1, name='Alex', address_id=1, address=Address(id=1, flat_number=101, post_code=10001)), ...]
```

Notice how also have the `address` of the customer, even though our query was `select(Customer)`.
This is due the relationship we had defined between the two classes.

> ##### Test Your Understanding
>
> Try creating mapped classes for other tables as well in the `db/` folder, and updating the queries to use the ORMs in `db_accessor.py`.
> If you need a reference, feel free to look at the `step-3-orms` branch to see how we've built the mapped classes.
{: .block-tip }

## Inserting Data

Inserting data is quite simple too. Within the ongoing transaction, the `Session` object becomes responsible for generating [`Insert`](https://docs.sqlalchemy.org/en/20/core/dml.html#sqlalchemy.sql.expression.Insert) constructs and emit **INSERT** statements.
We can do this by adding the ORM objects to the session by the [`Session.add()`](https://docs.sqlalchemy.org/en/20/orm/session_api.html#sqlalchemy.orm.Session.add) method.

Instances of the mapped classes represent rows. Thanks to the automatically generated `__init__()` constructor of mapped classes, we can instantiate objects using names of the columns as keyword arguments.
Keep in mind that the ids are generated automatically due to the database's auto incrementing feature for the primary key.
For example:

```py
address = Address(flat_number=101, post_code=10001)
```

Having said that, let's update our `add_new_order_for_customer()` function in `db_accessor.py`:

##### marketsvc/db_accessor.py

```py
from datetime import datetime

from db.base import engine
from db.customer import Customer
from db.item import Item
from db.order_items import OrderItems
from db.orders import Orders
from sqlalchemy import select
from sqlalchemy.orm import Session

...

def add_new_order_for_customer(customer_id, items):
    try:
        with Session(engine) as session:
            result = session.execute(
                select(Customer).where(Customer.id == customer_id)
            )
            customer = result.scalar()

            new_order = Orders(
                customer_id=customer_id,
                order_time=datetime.now(),
                customer=customer,
            )

            new_order.order_items = [
                OrderItems(
                    item_id=item["id"],
                    quantity=item["quantity"],
                )
                for item in items
            ]

            session.add(new_order)
            session.commit()

        return True

    except Exception:
        logging.exception("Failed to add new order")
        return False
```

Here, we've first fetched the `Customer` object with the given `customer_id` by using [`Select.where()`](https://docs.sqlalchemy.org/en/20/core/selectable.html#sqlalchemy.sql.expression.Select.where).
We then create a new object of `Orders` class.

Adding `OrderItems` becomes quite effortless now. We can just assign a list of `OrderItems` objects to the `Order.order_items` class attribute.
Doing so is straightforward because of the relationship between the two classes, whose type annotation is `Mapped[list["OrderItems"]]`. We can call `Session.add(new_order)` to add our instance to the `Session`.

After we call `session.add(new_order)`, the object is still in pending state and has not been inserted yet.
The transaction still remanins open until we call [`Session.commit()`](https://docs.sqlalchemy.org/en/20/orm/session_api.html#sqlalchemy.orm.Session.commit), or when the context manager automatically calls [`Session.rollback()`](https://docs.sqlalchemy.org/en/20/orm/session_api.html#sqlalchemy.orm.Session.rollback) and/or [`Session.close()`](https://docs.sqlalchemy.org/en/20/orm/session_api.html#sqlalchemy.orm.Session.close).

We can **commit** these changes to the database by calling `Session.commit()`.
If the transaction is successful, a **COMMIT** statement will be logged and the data will get persisted. Otherwise, a **ROLLBACK** will be logged.

Do you recall when we talked about bound parameters? The SQL expression language uses bound parameters by default.

You can call this function to insert a new order by hitting the `add_new_order` API using:

```sh
./run.sh neworder
```

##### SQLAlchemy logs

```sql
BEGIN (implicit)
SELECT customer.id, customer.name, customer.address_id, address_1.id AS id_1, address_1.flat_number, address_1.post_code 
FROM customer LEFT OUTER JOIN address AS address_1 ON address_1.id = customer.address_id 
WHERE customer.id = ?
[generated in 0.00016s] (1,)
INSERT INTO orders (customer_id, order_time) VALUES (?, ?)
[generated in 0.00013s] (1, '2024-05-15 10:44:30.605436')
INSERT INTO order_items (order_id, item_id, quantity) VALUES (?, ?, ?)
[generated in 0.00011s] [(4, 2, 4), (4, 3, 6)]
COMMIT
```

The first **SELECT** statement is to fetch the existing customer with the given `customer_id`. The first **INSERT** statement inserts data into the `orders` table and returns the generated `id` of the new inserted order. The second **INSERT** statement inserts multiple records to the `order_items` table using the same `id`. Notice how SQLAlchemy is using bound parameters for all the statements. Finally, we have a **COMMIT** to depict the data has been successfully comitted to the database.

> ##### Did you Know?
>
> It's possible to emit DDL statements and create tables in the database from the ORM mapped classes, using [`Base.metadata.create_all(engine)`](https://docs.sqlalchemy.org/en/20/core/metadata.html#sqlalchemy.schema.MetaData.create_all).
> See [this section](https://docs.sqlalchemy.org/en/20/orm/quickstart.html#emit-create-table-ddl) of SQLAlchemy docs for more details.
{: .block-tip }

&nbsp;

We have not only introduced ORMs that represent tables with classes, but also removed raw SQL text and made querying and inserting data much easier.
In the next section, we'll take a look at some optimisations and make our codebase even better!

> ##### Kudos!
>
> 🙌 You have now reached the `step-3-orms` part of the tutorial. If not, checkout that branch and continue from there:
>```sh
>git checkout step-3-orms
>```
{: .block-tip }

&nbsp;
