- [Preview](#preview)
- [Design](#design)
- [Functional Requirements](#functional_requirements)
- [Non Function](#non_function)
- [Explain Common](#explain-common)
    - [HomePage Flow](#home-page-flow)
    - [User Search Flow](#user-search-flow)

## Preview <a name="preview"></a>

I will design aws e-commerce system serving billions of users. This design will go no-function to function, top-down,
common to detail. </br>

## Design <a name="design"></a>

![AWS-system-design.png](img%2FAWS-system-design.png) </br>

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
load. Common homepage will have parameters like: top N best-selling categories and X products of each category, flatsale
information, menu... it will be loaded by territory, installed by admin page. </br>

We have a recommendation service, which are customized and customized according to the individual user (if the user is
logged in), or by ID define (cookie, app,...). A personnal homapage means, imagine, a user who is always disinterested
in one group of items, and is always interested and interested in another. For a good customer experience, the things
customers hate or never buy will not appear on the homepage. A merchine learning will work on the data ware house to
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

![homepage_aws_cache.png](img%2Fhomepage_aws_cache.png) </br>

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

