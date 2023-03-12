Database is the simplest place to store and process data is a collection of discrete values that convey information,
describing quantity, quality, fact, statistics, other basic units of meaning, or simply sequences of symbols that may be
further interpreted. This explanation is quite mechanical, but in terms of information technology, I think anyone will
understand what data is. It's like the concept of your body, you don't need a medical definition of your body to know
your body exists. </br>

To choose one or a group of databases suitable for the job, there are many ways. There are the following main
influencing factors: </br>

1) Data structure </br>

2) As query bridge </br>

3) The quantity and size needed are reasonable </br>

An extremely important mindset to remember: </br>

1) In a particular data and requirement, there is always a group of tools that can do it well. You're completely
   interchangeable, and there's always a way to do it: vD; mysql, mariaDB, sqlServer,.... are a group of DB tools that
   function quite similarly, although sometimes with slight differences. </br>

2) Database will not affect your business requirements. No matter what type of database you choose, there will always be
   a way to ensure your requirements, it may take a lot of work, go around or degrade performance. This mindset helps
   you think flexibly about the database and make flexible choices for real-life situations. </br>

Details of the database type and how to choose on request:

+) Caching Solution: </br>
When you need a very high read and write speed and a large number of read and write requests, and the request is allowed
to be lost if the DB crashes (I said the request is lost in the cache, but it is not allowed to be lost in my system,
here are 2 things completely different), then cache is a great choice. The most popular is redis which memcache. Highest
read and write speed and huge scaleout make cache the first choice for this need. </br>

+) Local in memory: </br>
It is kind of cache data. In essence here, the delay due to the network has been completely omitted, it will have a very
high speed, because the query is executed on the ram of the server that executes the query. However, it is not scalable.
It is suitable for super fast data with little change like data config, data that it does not need to scale out to many
instances, LMAX architect is a good example </br>

+) File archive: </br>

When you need to store files: images, videos, audio, ... it is also data but will not be selected to save in the
database. The reason, it's a file, it doesn't need to be queried in the file itself, a database of any kind to handle
the needs of querying, editing, etc. It's just like a databse in that it needs to be stored and read. That's why, in
this case, I need a storage, not a database. The popular storage is: aws S3, google drive, ... Storage usually has a CDN
layer to handle the problem of reading files with large and globally distributed users. You will see similarities with
redis combined with database in this solution. </br>

+) Text Search Capability: </br>
Ecommerce, Social Networking and many places where there is a need for fuzzy search. You want to search for a product, a
place, you only remember the approximate name, the search enginer will have to find and map the reasonable names with
high accuracy. Elastic search is a prime example. It provides fuzzy search, text search, high performance search and
great extensibility. However, it does not guarantee the same data protection as a traditional database. If there is a
search need as above, search enginer is a top choice. Mysql also has full text search but it is not powerful and not
designed for well distributed data. Use the right tools and the right purpose. </br>

+) SQL or NoSQL: </br>

These are the two most common and common types of databases. It is widely used in every part of life, almost always in
every project. <br>

Sql is a relational database, as the name implies, the data stored in sql has a clear relationship and has a specific
structure. The most important features of sql database: ACID </br>

+) Atomic can be understood as your transaction successful or unsuccessful. Only those 2 cases, no other cases. You can
think of it as the smallest, indivisible unit, so it has only 1 state. </br>

+) Consistency ensures your data is always consistent before and after the transaction, it shows the accuracy of the
data. </br>

+) Isolation means that in isolated transactions, a transaction is not affected by another transaction taking place at
the same time, affected in a negative sense. There is no way that one transaction affects the accuracy of another. </br>

+) Durability : when the transaction is completed, the changes will be correctly written to disk and not lost in the
event of a system crash. </br>

Now, we will follow this figure for reference for choosing sql or nosql:

1) When you need the ACID property, you need to use Relational DBMS. Popular Relational DBMS are: mysql, Sql Server,
   Oracle, Postgres,... If you don't need ACID, you can still use Relational DBMS or Non-relational database. </br>


2) If your data is unstructured, specifically like category for e-commerce product, each product type can have many
   different attributes, TV has { color, size, operating system, port charger, other tech info... }, but shorts { color,
   size, fabric...}. In this case, using a database like sql would be very complicated and laborious (there are ways to
   do it, eg magento, but it's not recommended), no-sql is a suitable choice.


3) In the nosql selection step, you will fall and fork between the two most popular types of sql, columnar database (eg
   Cassandra) and row database (MongoDB). As a simple rule, no-sql database has large read and write, and has many
   complex queries, flexible query, complete as of a product, mongoDB is often chosen. And if your data has a very large
   read frequency with many simple queries, then Cassandra is a solution. To elaborate on this, I have a more detailed
   link: </br>

An extremely important issue of choosing a database is combining databases. A product will have many types of DBs
suitable for many business characteristics, a flexible combination is required to solve the problem. For example:

1) User has millions of rides every day, I save them in cassandra. It will be easy to access with driver_id as the
   partition key. Easy query of a ride with a specific driver, Cassandra easily traced by partition key. But need to
   query all the rides the customer has taken, Cassandra will distribute this query to all partitions because the
   customer is not a partition key. You see, the queries are not designed properly, making a huge cost compared to data
   columns like Cassandra. </br>

2) Example one can be improved by having another table that stores customers and trips with the customer ID as the
   partition key. One principle of Cassandra made clear is: Defining Application Queries. </br>

3) A site as big as Amazon, has tens of millions, or more, orders per day. The data has a structure and needs absolute
   ACID, use mysql: inventory, balance, order, card,... Unstructured data, complex read-write queries, unpredictable,
   use MongoDB as your category product. Orders in the final state will be moved to Cassandra for long-term storage, for
   simple, high-frequency reading such as order queries. An ingenious combination makes the design fit for
   purpose. </br>

With this top-down approach, I hope to have helped everyone understand the principles and steps of choosing the right
database for themselves. </br>