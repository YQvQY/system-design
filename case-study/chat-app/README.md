# Design an Chat-App
Examples: Whatsapp, WeChat, FB messager

## Table of Contents
1. [Architecture Diagram](#architecture-diagram)
2. [Requirements](#requirements)
3. [Design Basics](#design-basics)
4. [Components Deep Dive](#components-deep-dive)
-----------------------

## Architecture Diagram
![architecture](/case-study/chat-app/chat-app.jpg)
-----------------------

## Requirements
### Functional
1. User Register/Login
2. 1-1 chat
3. group chat
4. text, image, audio and video
5. read receipt
6. last seen

### Non-Functional
1. Low latency, no lag
2. High availability
3. Scale - 4B users, 100B messages/day
-----------------------

## Design Basics
* A chat App allows users to send and receive messages. The sending part is straightforward, we can think it as a web request in any kind of web protocols. The receiving part is tricky, in a client-server http connection, normally the server is not allowed to contact the client. There are techniques designed for such bi-directional communication use case:
    * in a master-slave model or a distributed cluster node manager (like Zookeeper and Eureka) how do we know the client or a node is "alive"? There is technique called **heartbeat** where a simple request is sent to all nodes periodically to check the status. Similarly, we can ask the client to send a request to the server periodically to fetch available messages to mimic the server sending messages to the client. This is called **Polling**. 
    
        You may already see the problem here, it is very hard to pick the right polling frequency. If it's too low, there will be lag and this contradicts our non-functional requirement. If it's too high, we are wasting a lot of resource if there is not message to fetch. 
    
    * Can we make a "keep-alive" request and ask the server to keep the connection open until there is a message to fetch? The answer is **Long-Polling**. Long polling is implemented on the back of XMLHttpRequest, which is near-universally supported by devices so there’s usually little need to support further fallback layers.     
    
        However, it still creates a new connection each time, so it will send the HTTP headers, cookie header that may be large. Connections implies the work of many items like TLS handshake, firewalls, load balancers, web servers etc. Timeout is an issue as well since the server cannot hold the connection indefinitely. Another more serious problem is two users in the same chat are not guaranteed to be connected to the same server and it is unrealistic and unsealable to maintain a connection on each server, plus unreliable message ordering is possible for multiple HTTP requests from the same client to be in flight simultaneously. 

    * **WebSockets** is a more promising solution which keeps a unique connection open while eliminating the latency problems that arise with long polling. Full-duplex asynchronous messaging is supported so that both the client and the server can stream messages to each other independently. The initial connection is still established via normal HTTP handshake though. 

    * [HTTP/2 Pus](https://www.smashingmagazine.com/2017/04/guide-http2-server-push/) is another promising new technique but it is not widely used for now. 

* There are more than 65k ports in a modern server. We are not able to use all of them since some are registered, but we are still able to server a lot of connections on a single server. In order to know the server/port that the connection is on, we need a in-memory cache like Redis for fast look-up. 

* One very strict uniqueness we want to maintain is the IDs. For a small single server App, the auto-increment feature for most databases should be good enough. For distributed databases, we cannot guaranteed the uniqueness unless we have a centralized place to manage all the IDs but this defeats the purpose of distributed system. One solution is we can assign a set of IDs to a server to make sure there is no overlap. 
 
-----------------------


## Components Deep Dive

### User App Flow
User service will be a normal RESTful API provides CRUD operation for Users. Read more in the streaming platform [example](https://github.com/douyouzhe/system-design/tree/main/case-study/streaming-platform#user-login-flow).

When you go to the App, the first thing you should see is all the chats you have, including both *1-1 chats* and *group chats*. You can also add or remove chat. All these is managed by a Chat/Group Service which handles the CRUD operation for chats. The Group Chat Service stores the *[group_id, List<user_id>]* pair.

When the users open the App, we want to log the timestamp as the *last-seen* time which will be stored and displayed in the chat with other users. This timestamp does not need to be very accurate so we can adopt a sampling techniques, like pull the user status every minute and record it if the user has the App open. The data structure can be simple, a *[user_id, timestamp]* pair will be enough so any k-v store should work for us. The online time is a very useful metrics so we will feed this to kafka for offline analytic use.

-----------------------

### User Chat Flow (WIP)

Every time the user opens a chat, we establish a websocket connection for this user and the receiver and store the *[user_id, websocket_server/port]* pair using the WebSocket Manager. Using a k-v store will allow us to scale our App easily due to easy horizontal scaling. 

<p align="center">
<img src="/case-study/chat-app/websocket.jpg" width="400">
</p>

When the users send a message, the message is stored via Message Service. A message object should contain message_id, sender_id, receiver_id, encrypted content, status and some timestamps. The object is fed into Kafka for further processing. We first get the receiver's websocket information from the the WebSocket Manager then send the message to the receiver's client. The **status** field can be used for several use cases:
* When it's delivered, we can change the status to *delivered* so the sender will be updated. 
* When it's read,  we can change the status to *read*.
* We can add additional features like archive or update the message so the **status** can be used accordingly. 

<p align="center">
<img src="/case-study/chat-app/messageobj.jpg" width="400">
</p>

The Group Message Object looks similar except for the receiver_id is replaced by a channel_id since we are sending this message to all the members in the group. We can get the list of receivers from Group Chat Service we talked about in the user App flow. 

One very strict uniqueness we want to maintain is the ids




-----------------------
## Reference

https://dev.to/kevburnsjr/websockets-vs-long-polling-3a0o

https://codeburst.io/polling-vs-sse-vs-websocket-how-to-choose-the-right-one-1859e4e13bd9

https://www.smashingmagazine.com/2017/04/guide-http2-server-push/







