# PostgreSQL optimization through Indexing

In the age of Big Data, Machine Learning, User Data Privacy and other big tech words it is often easy to forget that all of these often live in Databases as single datapoints. When we sift through a shopping website or when we invariably end up in StackOverflow while looking for a solution to a particularly sneaky bug a server somewhere is sending requests to a database that stores millions of usernames, comments, reviews... And those requests need to be handled fast. We are not used to waiting for information anymore. As far as we as users are concerned, the data is immediately available.

Good thing there are many solutions to that problem, all of which are dependant on the specific needs of the project and the technologies available for the development team. For example, Apps make use of Caches and other fast-retrieval strategies for often-accessed data, and multiple databases can work in tandem to exercise their strengths. We wouldn't get anywhere if we tried to think of all possible solutions to the optimization problem in this article. Also, I have no intention of boring you to that extent anyway, so we'll jump straight into a couple of assumptions:

- Out of all storage and database solutions, we are going to talk about **SQL** databases.
- More specifically, we are working with a **PostgreSQL** database.

Now that we have a manageable scope, let's dive in.

## How is a query executed by Postgres?

The first step in trying to get our queries to run faster is understanding how a query is executed by PostgreSQL. When we execute a SQL query in Postgres, it first parses the query, then the _planner_ decides on the best strategy to execute the given query. Once it determines a good **plan**, it executes it and returns any rows/errors to the client.

The _planner_ is an integral part of how a SQL database works. It's main purpose is to use all given and available statistics about the tables and their data to decide on a performant set of steps to retrieve the requested data. It has the freedom to do so because SQL is an inherently _declarative_ language that primarily doesn't concern itself with telling the database **HOW** to do things. It simply determines **WHAT** needs to be done.

When we write

```sql
SELECT name, address FROM users WHERE age > 50 LIMIT 100;
```

We are simply describing what we want the result to be. In this case, it's equivialnt to saying "Give me the name and address of up to 100 users that are above the age of 50". Notice we didn't describe what kind of algorithm it should use to traverse the tables, or how to check for the age, or in which order to make these checks. This allows the _planner_ to use it's knowledge of the current database makeup to design an optimal retrieval strategy.

The key part to pay attention to is the fact that the planner needs to have information about the tables and their data in order to show it's capablities and help us out. If all it has to work with are tables filled with columns that it knows nothing about in an unspecified order that is populated by an unknown number of rows, well... there's not much it can do and it'll probably resort to simply reading the entire table row by row in order to find the rows you're looking for.

It's our job as developers to provide the planner with "better" information about our data so the planner can most effectively help us by designing optimal execution plans. And one of the most basic ways we can do that is by using **indexes**.

## What are indexes

Even if you've never explicitly created an index in your tables, you've probably created at least one index for each table you've set up. That's because when creating a **primary key**, Postgres will automatically create a unique index to enforce that constraint.

Let's start with a trivial example. Suppose we are working with a table of players such as:

```sql
CREATE TABLE players (
  id INT,
  username VARCHAR
);
```

If a common query in our application is to retrieve a player given it's id, we could use a query similar to:

```sql
SELECT username FROM players WHERE id = [requested id];
```

Every time the database looks for a player, it will scan the entire table until it finds a player whose id equals `[requested id]`. Now if the player ids were guaranteed to be unique and the database had access to an ordered list of ids, it could devise a much better plan to find the specified id. Depending on how that list is structured, different search algorithms could be used that would guarantee a faster lookup speed than linear time. In essence, this is what indexing is all about.

One way to create an index for the id column would be the following query:

```sql
CREATE INDEX players_id_index ON players (id);
```

By default, Postgres creates an index using a B-tree, which is a type of balanced tree data structure. This allows lookups on the index to be made in logarithmic time. When dealing with small tables, the gains won't be very noticeable. In fact, Postgres won't even use an index during lookup unless the table has enough records to make the overhead of reading the index and accessing the referenced rows in the actual table worth the extra effort.

Taking into consideration a theoretical B-tree (Postgres's implementation might differ), these are some of the number of rows a database would need to read in order to execute the previous SELECT query with and without an index:

| Total Rows | Table Scan Column | Indexed Column |
| ---------- | ----------------- | -------------- |
| 128        | 128               | 7              |
| 1024       | 1024              | 10             |
| 32768      | 32768             | 15             |
| 1073741824 | 1073741824        | 30             |

As you see, the performance gain become more and more apparent the larger the dataset is.

If Indexes are so great, why isn't every column indexed by default?

Well, an index is only useful if it's kept up to date with the data in the actual table that holds the information. And updating an index is expensive because for every write operation (INSERT, UPDATE, DELETE) on the table, all of it's indexes will also need to be updated. The cost of updating an index is not as steep as the cost of looking up data sequentially but it becomes significant when many columns are indexed or the application makes very frequent write operations on a table.

| Total Rows | Write Non-Indexed Column | Write Indexed Column | Write Row with 8 Indexed Columns |
| ---------- | ------------------------ | -------------------- | -------------------------------- |
| 128        | 1                        | 7                    | 56                               |
| 1024       | 1                        | 10                   | 80                               |
| 32768      | 1                        | 15                   | 120                              |
| 1073741824 | 1                        | 30                   | 240                              |

Indexing is a very powerful tool but it comes with it's trade-offs. It's up to the developer to determine where indexes will be useful to the application and where they will become bottlenecks.

## How to identify indexing opportunities

Identifying when and how to create indexes in a database can be really tricky because they don't depend on technical requirements alone. The roadmap for an application, the projected load if the application grows and the seasonality of it all can affect the decision. But there are ways to get started.

The first objective is to identify SELECT queries that are run often in our application. These are the first targets for optimization. Ideally we are working with a table that is read from often but seldom written to. A users table in a webapp that holds static information about each user is a good example.

For this example we'll use a sample database with USDA food information. The .sql file to create and populate the database can be downloaded from https://github.com/morenoh149/postgresDBSamples. We'll also be using SeeQR, an open source Postgres analytics tool that I've recently been contributing to. If you'd like to follow along the examples, you will need:

- PostgreSQL installed with psql available in your PATH
- the USDA sample database imported into your local postgres server
- SeeQR

Let's imagine we are building an application that retrieves data from the USDA database and allows a user to search the records by food description. We can see that the USDA database has a table called **food_des** which contains a **long_desc** column. Using SeeQR, we can also see that 4 columns in that table have **NOT NULL** constraints: ndb_no, fdgrp_cd, long_desc and shrt_desc.

![Table Details](images/table_details.png)

If we wanted to get search for all food items for which the **long_desc** started with _'Cheese'_, we could query the database with the following query:

```sql
SELECT * FROM food_des WHERE long_desc LIKE 'Cheese%';
```

If we run that query in SeeQR and take a look at it's execution plan, we'll notice that it has a single node with the type **Seq Scan**. This means the databse is scanning through the entire table in order to find all records that satisfy our _WHERE_ condition. This table is not very large so the actual execution time is still minimal (this will depend on your system hardware and load when running this test). But if querying this information is the core functionality of our app, we most certainly would want to improve on this.

![Select Non-Index](images/select_nonindex.png)

First off, we can check which indexes are already set for this table via psql. If you run the following command, you'll notice we don't currently have an index that includes the **long_desc** column:

```shell
\d food_des
```

![psql Indexes](images/psql_indexes.png)

Let's go ahead and check the performance of this query with an index:

- Let's first create a copy of our **usda** database and connect to it.
- let's create an index for this column using the following query:

```sql
CREATE INDEX players_id_index ON players (id);
```

Now that we have indexed that column, we can run the same query on our copy of the USDA database and look at the execution plan.

![Select Indexed](images/select_index.png)

You'll notice there are two nodes in the Execution Plan tree now: a Bitmap Index Scan and a Bitmap Heap Scan. The first will search our created index for strings that start with 'Cheese' and the second will retrieve those rows from the table. 

If you jump to the comparison view in Seeqr, we can easily compare the performances of each query side by side.

![Compare Selects](images/compare_select.png)

In my particular run, the indexed version ran around 2.7 times faster than the original non-indexed one. That might not sound like a whole lot, but as our app grows and this table populates with more records, this difference will become more apparent and significant.

Does that mean we should add an index to the production database? Well, that depends on how often we need to write to it. If we follow the same steps to test an insert query on each table in our databases, we'll notice that the insert time rises drastically: 

![Insert Non-Index](images/insert_nonindex.png)

![Insert Indexed](images/insert_index.png)

![Compare all](images/compare.png)

That's where we need to think about the particular application we are working on and decide which queries are run most often, which are the most time-sensitive and where our current bottlenecks are.

It's always a good idea to A/B test the performance of an application's queries when deciding to add/remove indexes. To help make that easier, I reccomend using a database management tool. My personal favorite being SeeQr. 

Keep in mind that PostgreSQL has many other mechanisms for optimization that I am ignoring here. You might notice for example if you repeatedly run these queries while testing their execution time, the results may vary. I am not taking caching into consideration since that would also warrant an entirely different article. The tests here are aimed to give a rough estimate of the potential gains of indexing a column.

If you'd like to know more about SeeQR and contribute to it's development, visit it's repository at 
https://github.com/open-source-labs/SeeQR. If you'd like more information about Indexes and the inner workings of PostgreSQL the https://www.postgresql.org/docs/current/ is a great place to start.

## Other tools worth checking out

- [pgcli](https://www.pgcli.com/) - psql alternative with autocomplete
- [pev](https://tatiyants.com/pev/#/plans) - online execution plan visualizer
- [Explain.dalibo.com](https://explain.dalibo.com/) - online execution plan visualizer

