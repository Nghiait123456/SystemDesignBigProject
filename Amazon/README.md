- [Preview](#preview)
- [Design](#design)
- [Functional Requirements](#functional_requirements)
- [Non Function](#non_function)
- [Explain Common](#explain-common)
    - [HomePage Flow](#home-page-flow)
    - [User Search Flow](#user-search-flow)
    - [User Purchase Flow](#user-purchase-flow)
    - [User Profile Flow](#user-profile-flow)
- [Infra Explain](#infra-explain)
    - [Load Balance](#load-balance)
    - [Api Gateway](#api-gateway)
    - [Redis cluster](#redis-cluster)
    - [MongoDB](#mongodb)
    - [Cassandra](#cassandra)
    - [Mysql Cluster](#mysql-cluster)
    - [Fire Ware](#file-ware)
    - [Rate Limit](#rate-limit)
    - [Http Server](#http-server)

## Preview <a name="preview"></a>

I will design aws e-commerce system serving billions of users. This design will go no-function to function, top-down,
common to detail. </br>

## Design <a name="design"></a>

![AMZ-system-design.png](img%2FAMZ-system-design.png) </br>

## Functional Requirements <a name="functional_requirements"></a>

+) Provider homepage, user information page.</br>
+) Provide a search functionality, remove products was out of stock, search with delivery's info. </br>
+) Provide checkout flow smoothly, payment flow smoothly.</br>
+) Provide a catalog of all products.</br>
+) Provide Cart and Wishlist features. </br>
+) Provide a view for all previous orders. </br>
+) Provider Inventory providers handle race conditions with hundreds of thousands of people buying the same
production </br>

## Non Function <a name="non_function"></a>

+) Low latency. </br>
+) High availability. </br>
+) High consistency. </br>

An e-commerce system that meets all three factors above with a high traffic, billions of users is a difficult problem.
To clarify, there are some services that must strictly meet 3 factors: inventory, checkout, ... Search service needs
high availability and does not need absolute consistency. Service order's history, user profile also requires 3 factors
above, but not absolutely. To thoroughly solve this problem, it is necessary to combine a lot of knowledge and
experience in handling race conditions. Here we go... </br>

## Explain Common <a name="explain-common"></a>

## HomePage Flow  <a name="home-page-flow"></a>

![home-page.png](img%2Fhome-page.png) </br>

A Homepage is the soul of an e-commerce site, it is a determining factor in whether customers are excited to buy. I
design a flexible homepage suitable for regions and users. </br>

Every time a user visits, the homepage service will rely on the user's ip to determine the appropriate common homepage
load. Common homepage will have parameters like: top N best-selling categories and X products of each category, flat
sale
information, menu... it will be loaded by territory, installed by admin page. </br>

We have a recommendation service, which are customized and customized according to the individual user (if the user is
logged in), or by ID define (cookie, app,...). A personal homepage means, imagine, a user who is always disinterested
in one group of items, and is always interested and interested in another. For a good customer experience, the things
customers hate or never buy will not appear on the homepage. A machine learning will work on the data ware house to
come up with the best personalized information. </br>

Combining the two services, we have the homepage information. </br>

//menu ex:

```
{
version: xxxx,
android: {
                   menu : {...},
                },
web :{
                menu : {...},
           },
........

}
``` </br>

//trend category ex:
```

```
{
version: xxxx,
android: {
category-product-trend : {...},
},
web :{
category-product-trend : {...},
},
........

}
``` 

The data of the homepage is in a non-relational form and has high flexibility, the frequency of reading and writing is
not too large, the amount of data is quite small, mongoDB will be selected. For details on when to choose which data,
see the
link: https://github.com/Nghiait123456/SystemDesignBigProject/blob/master/BestPracticeChooseDataBase/README.md   </br>
The non-function requirement of the homepage is high availability and short page load time. With a huge amount of
homepage traffic, there won't be a single request to get information directly from the Database. All information of
homepage will be taken from redis cluster. The updated cache and invalidate cache information will be taken from the
event source and updated in the background. </br>

If any of the homepage redis records are lost, no queries will be able to access the database directly. If this happens,
a storm of hundreds of thousands of users will overwhelm the database, which was originally designed not to suffer from
high frequency of reads. The request sent there will get an error, an empty-homepage-... event will be emitted, and the
homepage service will receive that event and update the data to cache. At this step, there can still be an event race,
the homePage service will have a minimum time for an update of x seconds. </br>

In frontend, all files, css, js, images, videos ... will be loaded from CDN and must be loaded from CDN. The css, js,
html files will have to be cached according to the version in the browser to reduce the CDN load. </br>
These are the most common notes for the homepage service to meet the needs of the problem. </br>
![homepage_amz_cache.png](img%2Fhomepage_amz_cache.png)</br>

## User Search Flow  <a name="user-search-flow"></a>

![user-search-flow.png](img%2Fuser-search-flow.png) </br>
A large e-commerce platform like AWS will have a lot of suppliers. I have Inbound Services to manage all suppliers.
Inbound Service will interact with supplier systems and fetch the relevant data. When a supplier is added, or suppliers
add a new product, the entire system involved must know so that the information is easily available to the user. These
data will be transmitted by the Inbound service as events to kafka, through consumer listening. Kafka forwards this
event to the homepage service, search service... or any service that listens for the event.

Specifically about storing Inbound data: service Item service will have consumers listening to kafka onboard new items,
new suppliers,... It will perform add, update, fetch items operations. The data will be stored in MongoDB, because the
data is all un-struct. For example, a blanket will have color, size, weight, fabric type, a phone will have: size,
color, operating system, tech info..., items. MongoDB is suitable for this data type, details of how to choose DB are in
the link: https://github.com/Nghiait123456/SystemDesignBigProject/blob/master/BestPracticeChooseDataBase/README.md </br>

As soon as a new item is on-boarded, a search consumer will make sure that this item can be queried by the users. It
will read and process all the on-board products, and store it in the database so that the Search Service can understand
this data. After formatting and standard data, it pushes the data to the ElasticSearch database. I use an ElasticSearch
here as it is very efficient for text-based search and also supports fuzzy search which we need for seamless user
experience. Details of how to select the DB at the
link: https://github.com/Nghiait123456/SystemDesignBigProject/blob/master/BestPracticeChooseDataBase/README.md. </br>

Search services will now interact with Elastic Search which will expose APIs to filter, sort, search,.. the products.
Function require: "search with delivery". That is, the Search Service will have to discard the results that cannot be
shipped. The Search service will have to talk to the Logistic Service to know if there is a route between the user's
default address and the warehouses, which route is closest, the approximate delivery date... and pass it on to the
Search Service. Search Service will display information to the user. To do this, the Logistic Service has to do smart
traffic, it has a different system design, or it can use the api of maps like google. </br>

Search Service also has a function that does not find and display out of stock products. Search Service will have to
list product search to Out Of Stock Service and Out Of Stock Service will return products that are out of stock. Why
need Out Of Stock service without having to get information directly from Elastic Search. Elastic Search takes time to
update inventory, reIndex, ... larger than realtime search demand. Elastic Search is not generated for real-time search,
detail in https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html.
Out Of Stock Service will do a simple job, return all Out Of Stock products from the inbound item list. It must ensure
relatively high realtime and extremely short response time, extremely large number of queries. First, every time a
product is out of stock or from out of stock to in stock, Inventory Service will send events to kafka, Out Of Stock will
listen to these events and update its data. All data will be saved in Redis Cluster, ensuring the fastest response time.
All Out Of Stock work will be done in the background, there will always be direct response results as soon as there is a
fetch request. </br>

Search Service is a difficult service, in addition to accurate and effective search, it also has to ensure the realtime
of many factors. Information when the process is completed will be sent to the user. A detailed search
service: https://github.com/Nghiait123456/eomerce_search_service </br>

From the search screen, the user should be able to wishlist a product or add it to the cart. This happens via Wishlist
Service and Cart Service. Wishlist is a repository for all wishlists and Carts for all products. The two translations
are built the same, they provide API fetch, update, delete items. They are stored on Mysql. Especially Cart and
Wishlist have clear struct, have clear relationship of components, have UUID item and some related information, require
integrity, Mysql will be the choice. The Wishlist information will not be updated continuously, depending on the user
action, Cart and Wishlist are all information with very high read frequency, but the race conditions is not large
because it is specific information of a user, so the cache is created. First time when reading to DB and updating by
event sourcing. When the cache is invalidate, the first get info cache request will read from the DB and update the
cache for the next read. </br>

Every time a lookup event search, an event is sent to kafka. This is the data source for me to build a personalized
system according to user preferences and recommend the most popular and relevant products. </br>
Our Kafka service will be connected to a spark streaming consumer to generate real-time reports such as: most searched
products of the hour, day of the day, favorite products of the day, week, categories with the highest revenue . Our
Kafka service will be connected to a spark streaming consumer, data coming from Kafka will also be dumped in a Hadoop
cluster, ALS algorithms,... machine learning will be used to generate personalized recommendations, search products,
wish list,... This is the most general keyword, I'm not an expert in this field so I won't mention it in detail.
Finally, the personalized user information and Wishlist will be pushed by Spack to kafka and kafka to the
Recommendation Service. This information will be stored on Cassandra DB. Information about Recommendation is un-struct
in nature and has a very large read frequency, so CassandraDB was chosen. </br>

User Service is a service that provides all information and manipulation with user info: fee, add, update, delete, ...
user. The user's information is related, structured and must be complete (rewards, addresses, coupons,...), the
relationships will revolve around the UserID. Therefore, Mysql Cluster will be selected to store information. Most of
the readings will be stored in the Redis Cluster, updated via event sourcing. A little more in-depth analysis of how
invalidate caches are handled. Here, the information representing that individual, when the information is not found on
the cache, can freely go to the DB to read and update it in the cache. Each information represents only one user so a
storm of race conditions update to DB will not take place. Handling cache in this service is simpler than many services
with race conditions when cache invalidate is extremely high. </br>

How does Logistic Service know if delivery is possible, the most boring route. Logistic Service will get all the codes
in the warehouses, Logistic Service will talk to the courier service to find out the closest distance between the
warehouse and the shipping addresses. The shipping addresses will be defined as clusters, in a city there are many
clusters, and the user's specific address will be in one of those clusters. After getting the shortest distance to the
address cluster, it needs to be fine-tuned to get the shortest distance to that address. This work is similar to the
work of google map. I am no expert in this area, I can elaborate on it on another occasion. The key is that all
information must be calculated background and available, to ensure realtime for search and payment operations. </br>

## User Purchase Flow  <a name="user-purchase-flow"></a>

![user_purchase_flow.png](img%2Fuser_purchase_flow.png) </br>
Ordering: When a user places an order, the request is routed to the order talking service, an order management system
that communicates with the order. The order data can be split into tables: customer, item table, order, transaction,
transaction_history, .... These tables are closely related to each other and have strict ACID requirements. Mysql
Cluster will be selected for this problem. </br>

Order Talking Service is the centralized point for coordinating and processing follow orders, payments,... The first
thing that Order Talking Service needs to make sure is that at a time, there is only one event, one request client,...
interacts with the Order Talking Service, this is guaranteed with the Red-lock Redis cluster. A request to Order Talking
Service, an identifier for that request is generated with the OrderId. If it's the first time to create an order (
orders not yet in redis will be considered first and vice versa), I will save in redis: key is the order ID, content:

```
{
order_id
created_at
expiry_time
status: "created"
}
```

Next, the required function is not to over-purchase, Order Talking Service will call Inventory Service to determine if
there is an exact stock for this product. The most difficult problem of e-commerce will arise here. With normal logic, I
need to put a transaction at the mysql layer or a lock at the cache layer. Transaction and lock DB will lock that
warehouse, and only the first order request can join the order, after that, I release the lock, commit or rollback
transaction. A similar job happens with lock redis. But when there are hundreds of thousands, millions of requests to
buy one or a group of products, mysql will not be able to handle it. In normal mode, mysql can only handle a few hundred
update commands per second. The payment request needs extremely high realtime, so putting it on the queue and processing
the background will not be possible. Here, there are 2 most popular solutions: sharding mysql or ignoring the database
in the calculation. The easiest solution to implement and the most likely to increase the load is Sharding Mysql, the
tool chosen here is Mysql Cluster. Mysql Cluster will provide the easiest way to increase mysql performance without much
processing. Another solution is to skip IO, the most famous architecture known is
LMAX(https://martinfowler.com/articles/lmax.html). Digging too deep into LMAX would be within the scope of this
document, I have a projetc that implements LMAX Service quite similar to ecommerce but for
payment-gatewat: https://github.com/Nghiait123456/HighPerformancePaymentGateway-BalanceService. </br>

When Inventory Service has successfully confirmed the order, Order Talking Service will update the redis status
to: </br>

```

{
order_id
created_at
expiry_time
status: "checked_inventory"
}

```

Next, Order Talking Service will talk to Risk Service to manage all risks related to orders, payments, cards, suppliers,
weather,... After successful, Order Talking Service will change status to "pending_checkout", talk to Checkout Service
to proceed payment of the order. </br>

At checkout service, the first thing to do is check with the Pricing Service to make sure there is no abnormality in the
current price compared to the information from the Order Talking Service sent over, if there are any abnormalities, the
item will be cancelled. Checkout Service checks again with the total amount of the order so that nothing is wrong. If
there is any error, the order will be canceled, the Order Talking Service will receive the event and update the order
status in redis. Order Talking Service via socket or event sourcing to send this failure result to the user via the
device to directly pay and display the screen to the User. Order Talking Service will send a failed payment event,
services like Order Service will receive the event and update accordingly. </br>

```

{
order_id
created_at
expiry_time
status: "fail_checkout"
}

```

If all double check information is correct, Checkout Service will send event: Ready-For-Checkout of that order Id, Order
Talking Service receive event and send socket to front-end, user screen redirects to payment screen . From here, the
transaction flow coordination for the user will be performed by the checkout Service. There are three possible outcomes
from this payment flow: Success, failure, or no response.

When all payments are successful, the User will be redirected to the success screen managed by the Order Talking
Service. If the order is successful at 8:05 and the expiry time is 8:10, everything is valid, the Order Talking Service
will update the status: "checkout_success" and send an event to Kafka for the other services to update the DB
accordingly. The expiry time of the order we will place is very large, much larger than my service timeout and related
services, to avoid race conditions over timeout orders with successful orders. However, if the successful order event
comes after the timeout order event, the Order Talking Service will save the status as status: "exception_checkout" with
the corresponding error code. To avoid a race conditions, Order Talking Service will not send success payment event, it
just sends event exception checkout and Service Exception Handle will handle this later. This is necessary and very
rare, so it is allowed to exist in this design. Enabling this will reduce extremely complex race conditions between our
distributed systems. At the beginning of the article, we always only allow 1 event, a request to interact with an order
in Order Talking Service, so there will never be a case, 2 events succeed and expire at the same time.

When the payment fails, the action is the same as the successful payment but with the status "error_checkout". The
inventory service will have to listen for this event and rollback inventory. </br>
When the order timeout, the record will be deleted from redis, and the Order Talking Service will also have the same
action procedure as when it fails, only different status : "timeout". </br>

At this point, the Purchase Follow flow has been completed, I would like to clarify some bottleneck issues. A system
like Amazon can have many millions of orders a day, and an order must be kept for at least several years for audit. 10
years, Amazon has 10 * 365 * 5 * 10 ^6 orders, this terrible number is the bottleneck of mysql. DB is too large will
affect mysql's ability to work normally, migration is required. There are many solutions for this, the most common being
cold and hot databases. Orders that are in a final, immutable state can be backed up to warm or cold mysql clusters.
Here, these orders no longer have the ability to change status, it is just a request to read information, and the
reading information will be as simple as querying by OrderId, querying by status, created_at, category,... me will
choose
Cassandra for this service. The Order processing service is an order management service, event notification about any
change of order, order search. The Archiver System will search the row of orders in the last state, send them to the
History Order System. Here, the History Order System will add the order to Cassandra and send the event to Kafka. Order
Processing System when receiving the event, delete the order from mysql. A request to review order information will be
made to the History Order System via the Order Processing System. </br>

The shipping information will be updated in the respective Order History depending on the business of each
carrier. </br>

Finally, Reconciliation Service will automatically match all import and export orders in Inventory Service with Order
Service, all abnormalities will be alerted. Errors are almost not allowed to occur, but the financial system,
warehousing always must have a Reconciliation Service running in parallel. There will be anomalies that will be very
complicated to deal with by software, but by humans it will be very simple. </br>

## User Profile Flow  <a name="user-profile-flow"></a>

![user_profile_service.png](img%2Fuser_profile_service.png)
User Profile Service will be a read-only service, it aggregates the necessary information of the User and saves it in
Redis. This is done by getting service information, updating via event sourcing. </br>

## Infra Explain  <a name="infra-explain"></a>

## Load Balance  <a name="load-balance"></a>

A large product like Amazon, users are scattered across the globe and at peak times, can have several million rps. (This
is an assumed number of this design, the actual number may be many times larger). With those peculiarities, I needed a
distributed LB that could intelligently navigate the load and was highly fault tolerant.</br>

I have a more detailed analysis of
this: https://github.com/Nghiait123456/InfraSREDevopsBackendForBigProject/tree/master/InfraSREDevops/1_LoadBalancing#load_balancer_region_base_dns_and_l4lb, https://github.com/Nghiait123456/InfraSREDevopsBackend/InfraSREDevops/
/1_LoadBalancing#load_balancer_with_bgp_software </br>

## Api Gateway  <a name="api-gateway"></a>

I have a more detailed analysis of
this: https://github.com/Nghiait123456/InfraSREDevopsBackendForBigProject/tree/master/InfraSREDevops/26_ApiGateway#type_pai_gateway_in_microservice,
https://github.com/Nghiait123456/InfraSREDevopsBackendForBigProject/tree/master/InfraSREDevops/26_ApiGateway#how_do_choose_api_gateway </br>

## Redis cluster <a name="redis-cluster"></a>

I have a more detailed anlysis of
this : https://github.com/Nghiait123456/InfraSREDevopsBackendForBigProject/blob/master/InfraSREDevops/3_Cache/README.md#cache_cluster_is_require_for_high_load,
https://github.com/Nghiait123456/InfraSREDevopsBackendForBigProject/blob/master/InfraSREDevops/3_Cache/README.md#benchmark_rediss </br>

## MongoDB <a name="mongodb"></a>

When to choose mongoDB and benchmark, I have a detailed description at the
link: https://github.com/Nghiait123456/SystemDesignBigProject/blob/master/BestPracticeChooseDataBase/README.md, https://github.com/Nghiait123456/InfraSREDevopsBackendForBigProject/blob/master/InfraSREDevops/9_NoSql/MongoDb/REAME.md </br>

## Cassandra <a name="cassandra"></a>

When to choose Cassandra and benchmark, I have a detailed description at the
link: https://github.com/Nghiait123456/SystemDesignBigProject/blob/master/BestPracticeChooseDataBase/README.md, https://github.com/Nghiait123456/InfraSREDevopsBackendForBigProject/blob/master/InfraSREDevops/9_NoSql/Cassandra/README.md#benchmark_cassandra </br>

## Mysql Cluster <a name="mysql-cluster"></a>

When to choose Mysql Cluster and benchmark, I have a detailed description at the
link: https://github.com/Nghiait123456/SystemDesignBigProject/blob/master/BestPracticeChooseDataBase/README.md, https://github.com/Nghiait123456/InfraSREDevopsBackendForBigProject/blob/master/InfraSREDevops/8_Sql/Mysql/Sharding/README.md#BenchmarkMysqlCluster </br>

## Fire Ware <a name="file-ware"></a>

Firewall, security and ddos prevention system is an important part. Preventing ddos is a difficult problem, because the
reason the threshold of ddos load is very large, much larger than your infrastructure. Please use third party for this
work. I have a link that describes the problem in
detail: https://github.com/Nghiait123456/DissectLaravel#why_do_not_use_rate_limit_laravel_for_attack_ddos, https://github.com/Nghiait123456/DissectLaravel#best_practice_prevent_attack_ddos</br>

## Rate Limit <a name="rate-limit"></a>

I have detail rate limit in
link: https://github.com/Nghiait123456/InfraSREDevopsBackendForBigProject/tree/master/InfraSREDevops/2_RateLimit </br>

## Http Server <a name="http-server"></a>

I have detail scale out http server in
link: https://github.com/Nghiait123456/InfraSREDevopsBackendForBigProject/tree/master/InfraSREDevops/16_HttpServer </br>