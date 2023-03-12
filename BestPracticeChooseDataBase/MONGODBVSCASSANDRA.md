Overview: </br>

MongoDB is a popular NoSQL database system, used for highly flexible and scalable applications. MongoDB stores data in a
document style, allowing for the storage of complex or dynamic structured documents, such as JSON documents. This makes
it suitable for web and mobile applications where data often does not follow a particular structure. </br>

The MongoDB data example is a collection of product documents, where each document corresponds to a different product.
Each document can contain different attributes, such as price, description, images, and reviews. These documents can be
freely added or removed, allowing the system to scale horizontally easily. </br>

Cassandra, another NoSQL database system, is designed to handle large amounts of highly distributed columnar data.
Cassandra is often used for applications that require high availability and responsive performance. It is widely used in
online storage applications for large companies such as Facebook and Twitter.

An example of Cassandra data is a table containing information about customer records, where each column corresponds to
a customer attribute, such as name, address, and phone number. These columns are stored row by row, forming a matrix of
column data for the entire table. This makes it possible for Cassandra to store and query columnar data more quickly and
efficiently than traditional relational database systems.

I will conduct a comparison of each of these two popular DB types, which represent the two most common types of
databases in no-sql: Store data in rows and store data in columns.

1) MongoDB is good for even complex read and write queries, Cassandra is usually only good for simple read
   queries. </br>

1.1) In MongoDB, suppose we have two data tables "Users" and "Orders". The "Users" table contains information about the
user, including fields such as "ID", "Name", "Email", "Age". The "Orders" table contains information about the user's
order, including fields like "OrderID", "UserID", "Product", "Price". </br>

To find all products that cost less than $100 and are rated above 4 stars in both tables, we can use a Join query to
join the two tables together. In MongoDB, the Aggregation Framework can be used to perform this query easily. </br>

Example query in MongoDB:

```
db.Users.aggregate([
{
$lookup:
{
from: "Orders",
localField: "ID",
foreignField: "UserID",
as: "UserOrders"
}
},
{ unwind:"unwind:"UserOrders" },
{
$match: {
"UserOrders.Price": { $lt: 100 },
"UserOrders.Rating": { $gte: 4 }
}
},
{ $project: { Name: 1, Email: 1, Age: 1 } }
])
```

However, in Cassandra, there is no Join feature and Aggregation Framework to perform this query. This makes it more
difficult to execute more complex queries in Cassandra than in MongoDB. </br>

1.2) In MongoDB, we can use features like aggregation pipeline to execute complex queries. For example, we have a table
of customer order list data, which includes fields like "OrderID", "CustomerID", "ProductID", "Quantity", "Price". We
want to get a list of the most sold products in the last month. </br>

To perform this query in MongoDB, we can use the aggregation pipeline to group records by ProductID and total Quantity,
then sort the results in descending order of Quantity to get the top selling products. . Example: </br>

```
db.orders.aggregate([
{ $match: { OrderDate: { $gte: new Date("2022-02-01"), $lte: new Date("2022-02-28") } } },
{ group: { _id: "ProductID", TotalQuantity: { sum:"sum:"Quantity" } } },
{ $sort: { TotalQuantity: -1 } },
{ $limit: 10 }
])
```

Meanwhile, Cassandra does not support aggregation pipeline features, which makes it more difficult to execute complex
queries with multiple data processing steps compared to MongoDB. </br>

1.3) With MongoDB, we can create a database with the following structure: </br>

```
     Database name: student_db </br>
     Collection: students </br>
     Field "student_id": integer, unique ID for each student </br>
     Field "full_name": string, student's full name </br>
     "address" field: string, student's address </br>
     "math_score" field: real numbers, student math scores </br>
     School "english_score": real number, student's English score </br>
     School "science_score": real numbers, students' science scores </br>
```

To perform a data query in MongoDB, we can use the find() and aggregate() methods to retrieve the records that match the
query condition. For example, to get information about students whose math_score is greater than or equal to 8: </br>

```
db.students.find({ math_score: { $gte: 8 } })
```

The advantage of MongoDB is its simplicity and flexibility in use. </br>

With Cassandra, we can create a database with the following structure: </br>

```
     Keyspace: school_ks </br>
     Table: students </br>
     Partition Key: student_id </br>
     Clustering Columns: none </br>
     Column "full_name": string, student's full name </br>
     Column "address": string, student's address </br>
     Column "math_score": real number, student's math score </br>
     Column "english_score": real number, student's English score </br>
     Column "science_score": real number, student's science score </br>
```

To perform a data query in Cassandra, we can use the SELECT statement to retrieve the records that match the query
condition. For example, to get information about students whose math_score is greater than or equal to 8: </br>

```
SELECT * FROM students WHERE math_score >= 8;
```

The advantage of Cassandra is that it is capable of handling large data and multimedia, supporting distributed features
and having high availability. </br>

2) Cassandra read speed is greater than MongoDB: </br>

2.1) Suppose we have a large data table with millions of records in both Cassandra and MongoDB systems. We want to query
the list of all products ordered in the last month. </br>

In MongoDB, indexes can be used to speed up queries. However, because MongoDB stores data in a row fashion, when
executing this query, MongoDB will have to scan through the entire row looking for records related to the previous
month. This makes querying data slower than Cassandra. </br>

In Cassandra, due to column-oriented nature, when executing this query, Cassandra only needs to access the column
containing the month information to search for records related to the previous month. This increases query speed and
reduces resource consumption. </br>

2.2) Suppose we have a large data table with millions of records in both Cassandra and MongoDB systems. We want to query
the list of all products ordered in the last month. </br>

In MongoDB, to speed up reading data, we can create an index on the OrderDate field so that MongoDB can search for
records related to the previous month quickly. Example: </br>

```
db.orders.createIndex( { OrderDate : 1 } )
```

When executing this query, MongoDB will use the index to quickly search for records related to the previous month.
However, because MongoDB stores data in a row fashion, when executing this query, MongoDB will have to scan through the
entire row looking for records related to the previous month. This makes querying data slower than Cassandra. </br>

In Cassandra, due to column-oriented nature, when executing this query, Cassandra only needs to access the column
containing the month information to search for records related to the previous month. This increases query speed and
reduces resource consumption. Example: </br>

```
SELECT * FROM orders WHERE OrderMonth = '2022-02';
```

3) Cassandra stores data in rows and MongoDB stores data in columns: </br>

3.1) Suppose we have a data table containing information about students, including fields like "ID", "Name", "
Address", "MathScore", "EnglishScore", "ScienceScore". </br>

In Cassandra, we can store MathScore, EnglishScore and ScienceScore columns by each row of the table to form a column
data matrix. When performing column data queries, Cassandra only needs to access the relevant columns without scanning
through the entire row. Example query column data of a student with ID 123: </br>

```
SELECT MathScore, EnglishScore, ScienceScore FROM Students WHERE ID = '123';
```

When executing this query, Cassandra will only access the MathScore, EnglishScore, and ScienceScore columns of the row
with ID 123 to return the results. This increases query speed and reduces resource consumption. </br>

Whereas, in the traditional relational database system, columns are usually stored in rows, when performing column data
queries, the system must scan through the entire row to find the columns to be accessed. query, which makes querying
column data slower than Cassandra. </br>

With MongoDB, we can use the "sparse index" feature to index only non-null fields, saving storage space and speeding up
data queries. However, MongoDB is not capable of storing columnar data like Cassandra. </br>

4) The key concepts are in Cassandra or in a columnar database: </br>
   Keyspace, Partition Key and Clustering Columns are important concepts in Cassandra to design database
   structure. </br>
   Keyspace: Similar to database in SQL database management system, keyspace is a collection of interrelated data
   tables. </br>
   Partition Key: this is the data field that Cassandra uses to divide data into partitions. Each partition is stored on
   a node of the cluster. Partition key makes data retrieval faster by simply querying one partition at a time. </br>
   Clustering Columns: these are columns of data that will be sorted in a partition. Clustering columns are arranged in
   order from left to right. When querying data, Cassandra usespartition key to identify the partition to query, then
   use clustering columns to get specific data in that partition. </br>
   Composite Partition Key is the combination of multiple columns to create a single key in Cassandra. When using
   Composite Partition Key, columns are placed in the partition key in descending order of priority, with the first
   column having the highest weight. </br>

let's analyze messenger simple design, here we do not focus on design content but focus on understanding the main
concepts of clustered DB design: </br>

Keyspace: messenger_ks </br>

Message board: </br>

```
     Partition key: conversation_id (to optimize retrieval of messages in a specific conversation) </br>
     Clustering columns: message_id, sent_time </br>
     Column "sender": message sender name </br>
     Column "recipient": name of the recipient of the message </br>
     Column "content": message content </br>
```

Example of a query to get a list of messages in a specific chat: </br>

```
SELECT * FROM messages
WHERE conversation_id='conversation-id-123'
AND sent_time > '2023-02-01 00:00:00'
AND sent_time < '2023-03-01 00:00:00'
ORDER BY sent_time DESC;
``
In which: </br>

     The partition key is conversation_id, which helps Cassandra to store messages in the same conversation on a separate partition. </br>
     Clustering columns include message_id and sent_time, arranged in order from left to right to optimize data query. When executing queries, Cassandra uses the partition key and clustering columns to search for messages related to specific chats and certain times. </br>
     The query returns all fields of messages between February 1, 2023 and March 1, 2023 of a particular chat, sorted chronologically. decrease. </br>

User information table (users): </br>

     Partition key: user_id (to optimize information retrieval of a specific user) </br>
     Column "user_name": username </br>
     Column "avatar_url": path to user's avatar </br>
     Column "created_at": account creation time </br>

Example of a query to get a specific user's information: </br>

```

SELECT * FROM users
WHERE user_id='user-123';

```


In which: </br>

     Partition key is user_id, which helps Cassandra store each user's information on a separate partition. </br>
     The query statement returns all the data fields of a particular user. </br>

Conversations dashboard: </br>

     Partition key: conversation_id (to optimize information retrieval of a specific conversation) </br>
     Column "conversation_name": the name of the conversation </br>
     Column "created_by": who created the conversation </br>
     Column "created_at": the time the conversation was created </br>
     Column "members": list of members in the chat </br>


```

```
SELECT * FROM conversations
WHERE conversation_id='conversation-id-123';


```

In which: </br>

     Partition key is conversation_id, which helps Cassandra to store information of each conversation on a separate partition. </br></br></br></br></br></br></br></br></br></br></br></br></br>
     The query returns all the data fields of a particular chat. </br>

     Conversation member dashboard (conversation_members): </br.
         Partition key: conversation_id (for optimal retrieval of information about members of a particular conversation) </br>
         Clustering column: user_id </br>
         Column "member_name": the name of the member in the chat </br>

     Example of a query statement to get a list of members in a particular chat: </br>

```
SELECT * FROM conversation_members
WHERE conversation_id='conversation-id-123';

```

In which: </br>

The partition key is conversation_id, which helps Cassandra to store information about members of the same conversation
on a separate partition. </br>
The clustering column is the user_id, sorted to optimally retrieve the information of a particular member of that
chat. </br>
The query statement returns all the data fields of the members in the particular chat. </br>

5) Clarifying row save and column save:

```

account_id balance transaction_history
1 1000 ["2023-03-01: -500", "2023-03-05: +200", "2023-03-10: -100"]
2 500 ["2023-03-02: +200", "2023-03-07: -100"]
3 3000 ["2023-03-06: +500"]

```

```

account_id balance_2023-03-01 balance_2023-03-02 balance_2023-03-03 balance_2023-03-04 balance_2023-03-05
balance_2023-03-06 balance_2023-03-07 balance_2023-03-08 balance_2023-03-09 balance_2023-03- 10
transaction_history_2023-03-01 transaction_history_2023-03-02 transaction_history_2023-03-03
transaction_history_2023-03-04 transaction_history_2023-03-05 transaction_history_2023-03-06
transaction_history_2023-03-07 transaction_history_2023-03-08 transaction_history_2023-03-09
transaction_history_2023-03-10
1 1000 1000 1000 1000 500 500 500 500 500 400 ["-500"] [] [] [] ["+200"] ["-100"] [] [] [] ["-100"]
2 500 700 700 700 700 700 600 600 600 600 [] ["+200"] [] [] [] [] ["-100"] [] [] []
3 3000 3000 3000 3000 3000 3500 3500 3500 3500 3500 [] [] [] [] [] ["+500"] [] [] [] []

```

in row-by-row storage, in the first example, data is stored in rows, growing data means an increase in the number of
rows, but the number of columns remains the same. When querying, most of the cases will have to go through all the data
of that row </br>
In column-by-column, data is arranged in columns, as the data grows, the number of rows increases, but the number of
columns also increases. The column will contain the separate data in the shards. row will increase but will only
increase in that part of the region. That's why Cassandra offers an almost endless theoretical scalability. </br>

balance_2023-03-01 : balance is the column name, 2023-03-01 is Clustering Columns, here choose by time. </br>

Let's look at how to save, when creating or editing a new document, MongoDB will save the document directly in a row,
and Cassandra will save it distributed, it has to find the location of the columns of any partition and save it.
Therefore, the speed of creating new and updating of cassandra is slower than MongoDB. </br>