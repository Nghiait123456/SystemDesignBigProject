- [Preview](#preview)
- [Design](#design)
- [Functional Requirements](#functional_requirements)
- [Non Function](#non_function)
- [Explain Common](#explain-common)
  - [HomePage](#home-page)
    
## Preview? <a name="preview"></a>
I will design aws e-commerce system serving billions of users. This design will go no-function to function, top-down, common to detail. </br>

## Design? <a name="design"></a>
![AWS-system-design.png](img%2FAWS-system-design.png) </br>


## Functional Requirements? <a name="functional_requirements"></a>
+) Provider homepage, user information page.</br>
+) Provide a search functionality, remove products was out of stock.</br>
+) Provide checkout flow smoothly, payment flow smoothly.</br>
+) Provide a catalog of all products.</br>
+) Provide Cart and Wishlist features. </br>
+) Provide a view for all previous orders. </br>
+) Provider Inventory providers handle race conditions with hundreds of thousands of people buying the same production </br>


## Non Function? <a name="non_function"></a>
+) Low latency. </br>
+) High availability. </br>
+) High consistency. </br>

An e-commerce system that meets all three factors above with a high traffic, billions of users is a difficult problem. To clarify, there are some services that must strictly meet 3 factors: inventory, checkout, ... Search service needs high availability and does not need absolute consistency. Service order's history, user profile also requires 3 factors above, but not absolutely. To thoroughly solve this problem, it is necessary to combine a lot of knowledge and experience in handling race conditions. Here we go... </br>

## Explain Common? <a name="explain-common"></a>
## HomePage? <a name="home-page"></a>
![home-page.png](img%2Fhome-page.png) </br>


A Homepage is the soul of an e-commerce site, it is a determining factor in whether customers are excited to buy. I design a flexible homepage suitable for regions and users. </br>

Every time a user visits, the homepage service will rely on the user's ip to determine the appropriate common homepage load. Common homepage will have parameters like: top N best-selling categories and X products of each category, flatsale information, menu... it will be loaded by territory, installed by admin page. </br>

We have a recommendation service, which are customized and customized according to the individual user (if the user is logged in), or by ID define (cookie, app,...). A personnal homapage means, imagine, a user who is always disinterested in one group of items, and is always interested and interested in another. For a good customer experience, the things customers hate or never buy will not appear on the homepage. A merchine learning will work on the data ware house to come up with the best personalized information. </br>

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

The data of the homepage is in a non-relational form and has high flexibility, the frequency of reading and writing is not too large, the amount of data is quite small, mongoDB will be selected. For details on when to choose which data, see the link:...  </br>
The non-function requirement of the homepage is high availability and short page load time. With a huge amount of homepage traffic, there won't be a single request to get information directly from the Database. All information of homepage will be taken from redis cluster. The updated cache and invalidate cache information will be taken from the event source and updated in the background. </br>

If any of the homepage redis records are lost, no queries will be able to access the database directly. If this happens, a storm of hundreds of thousands of users will overwhelm the database, which was originally designed not to suffer from high frequency of reads. The request sent there will get an error, an empty-homepage-... event will be emitted, and the homepage service will receive that event and update the data to cache. At this step, there can still be an event race, the homePage service will have a minimum time for an update of x seconds. </br>

In frontend, all files, css, js, images, videos ... will be loaded from CDN and must be loaded from CDN. The css, js, html files will have to be cached according to the version in the browser to reduce the CDN load. </br>
These are the most common notes for the homepage service to meet the needs of the problem. </br>

![homepage_aws_cache.png](img%2Fhomepage_aws_cache.png) </br>