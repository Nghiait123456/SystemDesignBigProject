- [Preview](#preview)
- [Design](#design)
- [Functional Requirements](#functional_requirements)
- [Non Function](#non_function)
- [Explain Common](#explain-common)
    - [User App Flow](#user-app-flow)
    - [User Upload Multimedia Flow](#user-upload-multimedia-flow)
- [Infra Explain](#infra-explain)
    - [Load Balance](#load-balance)
    - [Api Gateway](#api-gateway)
    - [Redis cluster](#redis-cluster)
    - [Cassandra](#cassandra)
    - [Mysql Cluster](#mysql-cluster)
    - [Fire Ware](#file-ware)
    - [Rate Limit](#rate-limit)
    - [Http Server](#http-server)

## Preview <a name="preview"></a>

I will design Messenger system serving 4 billions users. This design will go no-function to function, top-down,
common to detail. </br>

## Design <a name="design"></a>

![messenger-system-design.png](img%2Fmessenger-system-design.png) </br>

## Functional Requirements <a name="functional_requirements"></a>

+) Live chat support </br>

+) Support group chat </br>

+) Send, share images, videos, files </br>

+) Indicate read/receipt of messages </br>

+) Indicate last seen time of Users </br>

## Non Function <a name="non_function"></a>

+) Very Low Latency </br>
+) Always Available </br>
+) Highly Scalable And Scale Out Horizon </br>
+) Shouldn't Be Any Lags </br>

App chat at 4.5 billion users like messenger presents many challenges. Ensuring all 4 non-functions above requires a
tight combination of soft-ware, infra, devops, network and many other parts to solve the problem. Here we go.

## Explain Common <a name="explain-common"></a>

## User App Flow:  <a name="user-app-flow"></a>

![messenger-system-design.png](img%2Fmessenger-system-design.png)</br>
First, to be able to chat, users need a messenger account. This account usually syncs with the facebook account, but to
this extent, I'm ignoring that. User Service will manage all user information: add new, fetch, update,... Group Service
will manage information of Group Chats: members, roles, ... These information are related and structured clear
structure, centered on User information. It is stored in 2 mysql clusters, User DB and User Group DB. These data have a
very large read and write frequency, with a large scale like Messenger, all information will be cached in Redis Cluster.
In terms of data characteristics, when the cache is invalidate, need to add a new cache, User Data has a very low level
of race conditions, User Group Service has an average level of race conditions, both need 100% available but integrity
may not be necessary. is always 100%. A User changes information, after 1ms he sees this change, but his friend may take
100ms or so to see the change in front-end. It's a bit slower, but everyone involved will see this update. This is
acceptable in large distributed systems like facebook. Because of this data feature, updating the cache will become much
simpler. When the User is created, the event will be sent to Kafka, the user's personal information will also be cached
in Redis. Every time it changes, the cached User and User Ground information will also be updated thanks to
event-sourcing. For whatever reason the cache no longer exists, the corresponding request will be read in the DB and
updated to the cache. As analyzed, data has low race conditions, so a storm of requests coming to the DB will not
happen. </br>

We come to the three problems that are considered the most difficult in this design: </br>

1) Scale out Socket for 4 billion users. </br>
2) Contact with hundreds of thousands of Instance Socket scattered in many places. </br>
3) In GroupChat, there can be thousands of people chatting, let's call it thousands of devices, the order of messages
   must always be guaranteed. It is permissible for a message to arrive at person A before person B by a period of time
   less than the system tolerance, but it is not allowed in the wrong order of messages. </br>

Each problem will be dissected by me in the process of understanding the chats flow of the system:  </br>

First, a little about sockets. Socket is a network protocol that allows real-time and stateful communication (2 points
of the socket connection will be kept during the lifetime of the connection). It is widely used in all real-time
communication applications in various platforms, webServer, browser, mobile, desktop,... On the browser, it is an ideal
choice for real-time chat. Here, I choose socket as the communication protocol directly from the end user's device to
the system. </br>

A socket protocol is defined by 4 parameters and these 4 parameters must sum up a single information. They are: Source
IP, Source Port, Destination Ip, Destination port. Think of it as a road to an address, the road only allows one person
to go, so the path must be unique. Each machine has about 65000 ports and can use about 60K ports for socket, and 5K for
internal communication with services of that machine. Thus, one machine can only serve 60K users. There are a number of
ways to increase this number, but it's out of scope for design and it doesn't really work. With 4 billion users, the
whole system must have about 2 billion connect sockets in peak state (this is my assumption). Scale Out Socket is a must
for application like messenger. </br>

One difficulty and difference with http is that sockets have state (2 points of socket connection will be kept during
the lifetime of the connection). The client creates a socket connection with the server and sticks to the connect and
the server until the connection is cancelled. Scaling-ot it has different characteristics and is more difficult than
http Server. </br>

First of all, you need a lot of socket server instances to service billions of concurrent socket connections. Because
there is state, to talk to a device, we need to know where the connection socket to that device is kept. Thus, there are
two pieces of information that need to be saved: </br>

+) Which user is connected to which socket handler. </br>
+) All users are connected to the socket handler. </br>

These two information are sent by Socket Handler to Socket Manager, Socket Manager saves to Redis every time there is a
new connection, connection is canceled, problem,... All these events will be Socket Manager to Kafka for stakeholders.
The lifetime of this information on redis will be very short, because one connect socket session can be dropped or the
user goes offline, I want this information to be invalidate for a very short period of time, avoiding the existence of a
lot of unavailable information. . I put the estimate at 15 minutes. However, a device can stay connected for hours at a
time, and if every 15 minutes it is deleted, this is not reasonable. Each socket handler will have to have a background
job, which is assumed to continuously send information about all the socket connections that are held for that handle to
kafka. This information is collected by Socket Manager and renewed invalidate time on redis. All of this ensures that
the Socket Manager information stored in Redis is meaningful within the allowable real-time threshold. </br>

A data stored in the socket manager's redis will have the format: cache_key: User_id_connect, cache_value: ID of Socket
Handler (I will explain it below). </br>

So here, U1, U2 are online together, its information is saved in Web Socket Manager. U1 sends a message to U2, U1 will
send the message socket directly to the Instance holding the socket's connect, it's Web Socket Handler1. Web Socket
handler1 sends this message to WebSocket Manager. WebSocket Manager will send to the Socket Handler 1 event that your
message has been sent(1). At the same time, event Z to kafka indicates that a new message has been sent. This Event Z
reaches the Message Service and the message is saved to the Cassandra DB. Web Socket Manager will now also find the
information of the Web Socket Handler holding the connection to U2 as Web Socket Handler 2 and send information to it (
2). Web Socket Handler 2 quickly sends message to U2 about new message. U2 receives the response, and sends back the
successfully received socket message to the Web Socket Handler2. The Web Socket Handler2 reports this to the Web Socket
Manager. Web Socket Manager will send message to U1 via Web Socket Handler 1 and send event to Kafka... Stakeholders
will be aware of this event. When the message arrives to U1, U1 will show the U2 viewed status to the user. </br>

The question is, how can Web Socket Manager communicate with Web Socket Handler 1 and Web Socket Handler 2. I will
explain this part and also explain sections (1) and (2). Every time the Web Socket Handler is created, it will generate
a
random and send that ID to the Web Socket Manager. WebSocker Manager will create topic with Kafka based on this ID and
respond back to Web Socket Handler. The Web Socket Handler will also listen to this topic. This is how it listens for
information from the Web Socket Manager. Since the Web Socket Manager will open the port for the http server,
communicating with it will be easy. Any information it sends will be sent to Web Socket Manager via Http Message. The ID
of the Web Socket Handler will be saved and redis, and has a short expiration time. The Web Socket Handler will have to
schedule information to be sent to the Web Socket Manager to renew the timeout. </br>

So, in a nutshell: What Web Socket Handler wants to send will be sent through the Web Socket Manager's http Server. The
Web Socket Manager wants to send messages to the Web Socket Handler through kafka. </br>

A user has many devices online and wants all online devices to receive the message. More broadly, Web Socket Manager
will have to store all User's devices in Redis, it will be formatted as : </br>

{ key : User_id, content { list_socket_Handler_id => { ID1, ID2, ....},.... } } </br>

ID1 is the kafka topic that Web Socket Handler 1 is listening on. Web Socket Manager will find all related devices and
send messages according to the above process. </br>

Handling when the recipient is offline: </br>
U1 sends a message to U2 but U2 is offline. There is no information about the connection socket that U2 is holding. The
message will be sent to the Message Service, which is saved to Cassandra. As soon as U2 is online, it will talk to the
Message Service and pull back all the Messages it missed. </br>

The question is that each device will miss a different number of messages if the online time is different. Thus, to
handle this problem, it is necessary to type the order of messages in a centralized manner. First, each message will
have a random ID associated with that message. When the Message is sent by the device to the Web Socket Handle, the Web
Socket Handle is sent to the Web Socket Manager. At this step, Web Socket Manager will type the message in ascending
order. There will be a race condition for ordering, a Red Lock Redis will be used to handle the race. This sequence
number is returned to Web Socket Manager 1, and Web Socket Manager 1 is returned to U1 attached to the message. This
sequence number will accompany that Message forever and does not change, just like the message ID, all are stored to the
DB. When each device calls the Message Service and says it wants to pull messages it missed while going offline, it must
include a Sequence Number for the last Message received. The Message Service will rely on this sequence number to return
the missing message. <br>

There is a race conditions about the missing message update. When U1 sends U2 message T1, U2 has just been online but
cannot update his information online into Socket Manager. U2 reads all missing messages, and assumes no T1. When U2
finishes pulling the message, T1 will be saved in Message Service. After a while, U2 comes online and chats with U1, not
knowing about Message U1 stored in Message Service. Since this is a distributed system, this is entirely possible. To
handle this, after pulling the first time, U2 pulls a second time after N minutes. As an additional alternative, since
all messages are in consecutive order, if U2 receives a message that is not consecutive with the last message, it knows
that messages are missing and pulls these missing messages from the Message Service. These two solutions will almost
completely overcome this race condition problem. </br>

When U1 sends a message to U2 but U1 is offline, the message will be kept in the local memory of the device and sent as
soon as U1 is online. </br>

A difficult problem to deal with is chats group. Here, a message is sent by U1 to group G1, Group G1 has tens of
thousands of Users, that is, this message must be sent to those tens of thousands of devices. First, messages from U1
will also go to the Web Socket Manager, numbered and processed in the same way as with one-on-one chats. An event is
sent to KafKa and the message is stored by the Message Service in Cassandra. Instead of letting the Direct Message
Handler Flow handle, the Socket Manager will delegate the work to the Group Message Handler Flow. Group Message Handler
Flow will get all the User list of that Group, and also get the list of all Socket Handler attached to User via device.
What follows will be identical to Direct Message Handler Flow, Group Message Handler Flow will send messages to all
devices in the list. </br>

Scaling out the socket system for real-time chat with billions of users is a complicated task. For smaller systems, I
suggest using the famous open source pub sub for realtime chat. One recommended candidate to simplify this task
is https://github.com/centrifugal/centrifugo. </br>

Analysis: </br>
Analytics will do the data analysis task. It receives events from kafka, connects to Spack to parse the content, and
feeds it to Hadoop. I'm not an expert in this area, I won't go into it in depth but here it is the most common step to
take. </br>

Service last time will be based on Kafka event about connect Socket and get data about the last time User was online.
Information stored in Redis or Cassandra about the user id and last online. </br>

Example Design basic database chat with
cassandra: https://towardsdatascience.com/ace-the-system-interview-design-a-chat-application-3f34fd5b85d0 </br>

## User Upload Multimedia Flow:  <a name="user-upload-multimedia-flow"></a>

![upload-flow.png](img%2Fupload-flow.png) </br>
When a multimedia message is sent, it is also serialized by the Socket Manager. Next, video messages and photos will be
sent through the Upload Service and updated to S3, CDN. When the upload is successful, a link to the resource and the
message ID will be sent to the Websocket Manager and Message Service via kafka. The message will be sent to all devices
just like the existing process. One thing to note is that multimedia messages may appear later, but still in the same
order that Socket Manager arranged. If the upload fails, the event is sent to kafka, and the message sequence number
instead of saving the file link will save the Error_upload status and not be displayed to the device. </br>

## Infra Explain  <a name="infra-explain"></a>

## Load Balance  <a name="load-balance"></a>

A large product like Messenger, users are scattered across the globe and at peak times, can have several million rps. (
This
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



