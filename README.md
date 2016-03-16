# Note

This project is now updated to use Node 4.x (not backward compatible). In addition, it uses all latest modules of Express, Socket.io etc. 

You can start app by type ```DEBUG=redispubsub node bin/www```

# Scaling real-time apps (using Redis)



One of the most common things people build on Node.js are real-time apps like chat apps, social networking apps etc. There are plenty of examples showing how to build such apps on the web, but it’s hard to find an example that shows how to deal with real-time apps that are scaled and are running with multiple instances. You will need to deal with issues like sticky sessions, scale-up/down, instance crash/restart, and more for apps that will scale. This post will show you how to manage these scaling requirements.
人们最爱用Node.js做的一件事就是制作实时应用，比如聊天应用，社交应用等。网上有很多例子交给你如何写这样的应用，可是却很难找到例子向你展示如何将实时应用拓展运行在多个实例上。你将会遇到很多问题，比如如何处理sticky sessions，
## Chat App
The main objective of this project is to build a simple chat app and focus on tackling such issues. Specifically, we will be building a simple Express, Socket.io and Redis-based Chat app that should meet the following objectives:
这个项目的目标是完成一个简单的聊天应用。明确的说，我们要用Express, Socket.io 和 Redis为基础建立一个应用，并且实现下列目标：

1. Chat server should run with multiple instances.
1. 聊天服务器将会运行多个实例
2. The user login should be saved in a session.
2. 用户的登录状态应该保存在session中
    * If the user refreshes the browser, he should be logged back in.
    * 如果用户刷新页面，也依然会是登陆状态
    * Socket.io should get user info from the session before sending chat messages. 
    * Socket.io应当可以在发送聊天消息前从session中获得用户信息
    * Socket.io should only connect if user is already logged in.
    * Socket.io 应当仅在用户登陆的情况下连接用户
3. Reconnect: While the user is chatting, if the server-instance to which he is connected goes down / is restarted / scaled-down, the user should be reconnected to an available instance and recover the session.
3. 重新连接：在用户聊天时，如果聊天服务器宕机/重启/ scaled-down, 用户应当可以重新连接上一台可用的聊天服务器

***Chat app's Login page:***

<p align='center'>
<img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/chatAppPage1.png" height="" width="450px" />
</p>

***Chat app's Chat page:***
<p align='center'>
<img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/chatAppPage2.png" height="" width="450px" />
</p>

***Along the way, we will cover:***

1. How to use Socket.io and Sticky Sessions
2. How to use Redis as a session store 
3. How to use Redis as a pubsub service
4. How to use sessions.sockets.io to get session info (like user info) from Express sessions
5. How to configure Socket.io client and server to properly reconnect after one or more server instances goes down (i.e. has been restarted / scaled down / has crashed)



## Socket.io and Sticky Sessions ##

<a href='http://socket.io/' target='_blank'>Socket.io</a> is one of the earliest and most popular Node.js modules to help build real-time apps like chat, social networking etc. (note: <a href='https://github.com/sockjs/sockjs-client' target='_blank'>SockJS</a> is another popular library similar to Socket.io).

When you run such a server in a cloud that has a load-balancer/reverse proxy, routers etc, you need to configure it to work properly, especially when you scale the server to use multiple instances.

One of the constraints Socket.io, SockJS and similar libraries have is that they need to continuously talk to the ***same instance*** of the server. They work perfectly well when there is only 1 instance of the server.

Socket.io, SockJS 和其它类似的库共有的一个缺陷，就是他们只能和一个server实例交谈。在只有一个server实例的情况下，它们工作的很好。

<p align='center'>
<img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/socketio1Instance.png" height="300px" width="450px" />
</p>

When you scale your app in a cloud environment, the load balancer (Nginx in the case of Cloud Foundry) will take over, and the requests will be sent to different instances causing Socket.io to break.
当你想在云端增加server实例的时候，负载均衡器（Cloud Foundry的情况下是Nginx）会接管请求，请求会被分发到不同的server实例上，从而导致Socket.io崩溃。
<p align='center'>
<img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/socketioBreaks.png" height="300px" width="450px" />
</p>

To help in such situations, load balancers have a feature called 'sticky sessions' aka 'session affinity'. The main idea is that if this property is set, then after the first load-balanced request, all the following requests will go to the same server instance.
想要避免这种情况，负载均衡器有一个特性叫做'sticky sessions'。如果你设置了这个功能，那么在负载均衡器处理了第一条请求后，所有后续的请求都会去向同一个server实例。

In Cloud Foundry, cookie-based sticky sessions are enabled for apps that set the cookie **jsessionid**. 
如果你使用Cloud Foundry, 想要给应用设置cookie-based sticky sessions的话，需要设置 jsessionid cookie

Note: **jsessionid** is the cookie name commonly used to track sessions in Java/Spring applications. Cloud Foundry is simply adopting that as the sticky session cookie for all frameworks.

So, all the apps need to do is to set a cookie with the name **jsessionid** to make socket.io work.

```javascript
    /*
     Use cookieParser and session middlewares together.
     By default Express/Connect app creates a cookie by name 'connect.sid'.But to scale Socket.io app,
     make sure to use cookie name 'jsessionid' (instead of connect.sid) use Cloud Foundry's 'Sticky Session' feature.
     W/o this, Socket.io won't work if you have more than 1 instance.
     If you are NOT running on Cloud Foundry, having cookie name 'jsessionid' doesn't hurt - it's just a cookie name.
     */
    app.use(cookieParser);
    app.use(express.session({store:sessionStore, key:'jsessionid', secret:'your secret here'}));
```

<p align='center'>
<img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/socketioWorks.png" height="300px" width="450px" />
</p>

In the above diagram, when you open the app, 
过程如上图，当你打开应用的时候，

1. Express sets a session cookie with name **jsessionid**. 
1. Express 设置了名为jsessionid的session cookie
2. When socket.io connects, it uses that same cookie and hits the load balancer
2. 当socket.io连接的时候，它会使用相同的cookie访问负载均衡器
3. The load balancer always routes it to the same server that the cookie was set in.
3. 负载均衡器

## Sending session info to Socket.io 

Let's imagine that the user is logging in via Twitter or Facebook, or we implement a regular login screen. We are storing this information in a session after the user has logged in.

我们想象 用户通过Twitter, Facebook或者我们自己实现的逻辑登陆了，我们在用户登陆后将这些信息保存在session中

```javascript

app.post('/login', function (req, res) {
    //store user info in session after login.
    req.session.user = req.body.user;
    ...
    ...
});
```

Once the user has logged in, we connect via socket.io to allow chatting. However, socket.io doesn't know who the user is and whether he is actually logged in before sending chat messages to others.
一旦用户登陆了，我们要连接socket.io使得用户可以聊天。但是，socket.io并不知道用户是谁，也不知道他是否登陆了

That's where the `socket.io-express-session` library comes in. It's a very simple library that's a wrapper around socket.io. All it does is grab session information during the handshake and then pass it to socket.io's `connection` function. You can access session via `socket.handshake.session` w/in connection listener.
这里我们要用到 `socket.io-express-session`这个库，它在socket.io之上添加了一些功能。它在握手阶段将session信息传入到socket.io的`connection`方法中。于是我们便可以通过 `socket.handshake.session`方法来获得session信息了。

```javascript
//instead of
io.sockets.on('connection', function (socket) {
    //do pubsub here
    ...
})

var socketIOExpressSession = require('socket.io-express-session'); 
io.use(socketIOExpressSession(app.session)); // session support

//But with sessions.sockets.io, you'll get session info

/*
 Use SessionSockets so that we can exchange (set/get) user data b/w sockets and http sessions
 Pass 'jsessionid' (custom) cookie name that we are using to make use of Sticky sessions.
 */
var SessionSockets = require('session.socket.io');
var sessionSockets = new SessionSockets(io, sessionStore, cookieParser, 'jsessionid');

io.on('connection', function (socket) {

    //get info from session
    var user = socket.handshake.session.user;

    //Close socket if user is not logged in
    if (!user)
        socket.close();

    //do pubsub
    socket.emit('chat', {user: user, msg: 'logged in'});
    ...
});
```

<p align='center'>
<img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/sendingSession2SocketIO.png" height="300px" width="450px" />
</p>


## Redis as a session store

So far so good... but Express stores these sessions in MemoryStore (by default). MemoryStore is simply a Javascript object - it will be in memory as long as the server is up. If the server goes down, all the session information of all users will be lost! 
到目前为止还不错。。。但是Express将session存到MemoryStore（默认）中。MemoryStore只是一个简单的Javascript对象 - 它只是存储在内存中。如果服务器关闭，所有的session信息将会丢失。

We need a place to store this outside of our server, but it should also be very fast to retrieve. That's where Redis as a session store come in.
我们需要在服务器外的一个地方存储session数据，并且能够很快的读取。这时候我们就需要Redis作为session store来发挥作用了。
Let's configure our app to use Redis as a session store as below.
下列代码将Redis设置为我们app的session store
```javascript
/*
 Use Redis for Session Store. Redis will keep all Express sessions in it.
 */
var redis = require('redis');
var RedisStore = require('connect-redis')(express);
var rClient = redis.createClient();
var sessionStore = new RedisStore({client:rClient});
    
    
  //And pass sessionStore to Express's 'session' middleware's 'store' value.
     ...
     ...  
app.use(express.session({store: sessionStore, key: 'jsessionid', secret: 'your secret here'}));
     ...

```

<p align='center'>
<img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/redisAsSessionStore.png" height="300px" width="450px" />
</p>

With the above configuration, sessions will now be stored in Redis. Also, if one of the server instances goes down, the session will still be available for other instances to pick up.

## Socket.io as pub-sub server

So far with the above setup our sessions are taken care of - but if we are using socket.io's default pub-sub mechanism, it will work only for 1 sever instance.
通过上边的配置，session就处理好了。 可是还有其他的问题，我们现在如果使用socket.io默认的pub-sub（发布-接收）机制,他只能运行在一个server实例上。
i.e. if user1 and user2 are on server instance #1, they can both chat with each other. If they are on different server instances they cannot do so.
举个例子：如果user1和user2都在同一个server#1上，他们可以互相交谈。如果他们在不同的服务器上，他们是不能交谈的。

```javascript
sessionSockets.on('connection', function (err, socket, session) {
    socket.on('chat', function (data) {
        socket.emit('chat', data); //send back to browser
        socket.broadcast.emit('chat', data); // send to others
    });

    socket.on('join', function (data) {
        socket.emit('chat', {msg: 'user joined'});
        socket.broadcast.emit('chat', {msg: 'user joined'});
    });
}
```

## Redis as a PubSub service

In order to send chat messages to users across servers we will update our server to use Redis as a PubSub service (along with session store). Redis *natively supports pub-sub operations*. All we need to do is to create a publisher, a subscriber and a channel and we will be good. 

```javascript
//We will use Redis to do pub-sub

/*
 Create two redis connections. A 'pub' for publishing and a 'sub' for subscribing.
 Subscribe 'sub' connection to 'chat' channel.
 */
var sub = redis.createClient();
var pub = redis.createClient();
sub.subscribe('chat');


sessionSockets.on('connection', function (err, socket, session) {
    socket.on('chat', function (data) {
        pub.publish('chat', data);
   });

    socket.on('join', function (data) {
        pub.publish('chat', {msg: 'user joined'});
    });

    /*
     Use Redis' 'sub' (subscriber) client to listen to any message from Redis to server.
     When a message arrives, send it back to browser using socket.io
     */
    sub.on('message', function (channel, message) {
        socket.emit(channel, message);
    });
}

```

The app's architecture will now look like this:

<p align='center'>
<img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/redisAsSSAndPS.png" height="300px" width="500px" />
</p>


## Handling server scale-down / crashes / restarts

Our app will work fine as long as all the server instances are running.  What happens if the server is restarted or scaled down or one of the instances crash? How do we handle that?
如果所有的server实例都运行的很好，那么我们的聊天应用就不会有问题。但是一旦有server重启了或者崩溃了，我们该如何处理呢？

Let's first understand what happens in that situation.
我们首先来看一下在这种情况下会发生什么。

The code below simply connects a browser to server and listens to various socket.io events.
下边的代码
```javascript
 /*
  Connect to socket.io on the server (***BEFORE FIX***).
  */
 var host = window.location.host.split(':')[0];
 var socket = io.connect('http://' + host);

 socket.on('connect', function () {
     console.log('connected');
 });
 socket.on('connecting', function () {
     console.log('connecting');
 });
 socket.on('disconnect', function () {
     console.log('disconnect');
 });
 socket.on('connect_failed', function () {
     console.log('connect_failed');
 });
 socket.on('error', function (err) {
     console.log('error: ' + err);
 });
 socket.on('reconnect_failed', function () {
     console.log('reconnect_failed');
 });
 socket.on('reconnect', function () {
     console.log('reconnected ');
 });
 socket.on('reconnecting', function () {
     console.log('reconnecting');
 });
```

While the user is chatting, if we restart the app **on localhost or on a single host**, socket.io attempts to reconnect multiple times (based on configuration) to see if it can connect. If the server comes up with in that time, it will reconnect. So we see the below logs:
当应用跑在locaohost或者单一的服务器上时，我们重新启动应用，socket.io会尝试重新连接（次数由配置决定）。如果服务器恢复正常，他会重新连接。我们会看到如下的log信息
<p align='center'>
<img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/reconnectOn1server.png" height="300px" width="600px" />
</p>

If the user is chatting on the same app that's running ***on Cloud Foundry AND with multiple instances***, and if we restart the server (say using `vmc restart redispubsub`) then we'll see the following log messages:
当应用跑在云平台如Cloud Foundry并且有好几个server实例的时候，如果我们重新启动服务器，我们会看到如下的log信息
<p align='center'>
<img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/reconnectOnMultiServer.png" height="400px" width="600px" />
</p>

You can see that in the above logs, after the server comes back up, socket.io client (running in the browser) isn't able to connect to socket.io server (running on Node.js in the server). 
从上边的log中我们可以看到，经过服务器重启，socket.io client（跑在浏览器中）不能够连接到socket.io server(跑在服务器上)

This is because, once the server is restarted on Cloud Foundry, ***instances are brought up as if they are brand-new server instances with different IP addresses and different ports and so `jsessionid` is no-longer valid***. That in turn causes the load balancer to *load balance* socket.io's reconnection requests (i.e. they are sent to different server instances) causing the socket.io server not to properly handshake and consequently to throw `client not handshaken` errors!
这是因为一旦Cloud Foundry上边的服务器重启，这些服务器实例会像新的一样，拥有与之前不同的IP地址，与之前不同的端口地址，于是jsessionid就失效了。导致了负载均衡器对sock.io的重新连接请求做负载均衡（把它们分配给不同的服务器实例），socket.io不能正确地握手，结果报出`client not handshaken`错误!


### OK, let's fix that reconnection issue

First, we will disable socket.io's default "reconnect" feature, and then implement our own reconnection feature. 
首先，我们要关掉socket.io默认的重新连接功能，之后我们要实现我们自己的重新连接功能。
In our custom reconnection function, when the server goes down, we'll make a dummy HTTP GET call to index.html every 4-5 seconds. If the call succeeds, we know that the (Express) server has already set ***jsessionid*** in the response. So, then we'll call socket.io's reconnect function. This time because jsessionid is set, socket.io's handshake will succeed and the user will get to continue chatting happily.
在我们自己实现的重新连接功能中，当服务器挂掉时，我们会每4-5秒发送HTTP GET请求。如果请求成功，我们知道(Express)服务器已经在response中设置好了***jsessionid***，于是，我们就可以调用socket.io的reconnect法，这次由于jsessionid已经设置了，socket.io的握手将会成功，用户终于可以继续开心点聊天了。
```javascript

/*
 Connect to socket.io on the server (*** FIX ***).
 */
var host = window.location.host.split(':')[0];

//Disable Socket.io's default "reconnect" feature
var socket = io.connect('http://' + host, {reconnect: false, 'try multiple transports': false});
var intervalID;
var reconnectCount = 0;
...
...
socket.on('disconnect', function () {
    console.log('disconnect');

    //Retry reconnecting every 4 seconds
    intervalID = setInterval(tryReconnect, 4000);
});
...
...



/*
 Implement our own reconnection feature.
 When the server goes down we make a dummy HTTP-get call to index.html every 4-5 seconds.
 If the call succeeds, we know that (Express) server sets ***jsessionid*** , so only then we try socket.io reconnect.
 */
var tryReconnect = function () {
    ++reconnectCount;
    if (reconnectCount == 5) {
        clearInterval(intervalID);
    }
    console.log('Making a dummy http call to set jsessionid (before we do socket.io reconnect)');
    $.ajax('/')
        .success(function () {
            console.log("http request succeeded");
            //reconnect the socket AFTER we got jsessionid set
            io.connect('http://' + host, {
                        reconnect: false,
                        'try multiple transports': false
                    });
            clearInterval(intervalID);
        }).error(function (err) {
            console.log("http request failed (probably server not up yet)");
        });
};

```

In addition, since the jsessionid is invalidated by the load balancer, we can't create a session with the same jsessionid or else the sticky session will be ignored by the load balancer. So on the server, when the dummy HTTP request comes in, we will ***regenerate*** the session to remove the old session and sessionid and ensure everything is fresh before we serve the response.
另外，由于jsessionid已经失效，我们不能够由jsessionid生成一个session,否则会被负载均衡器忽视掉。所以在服务器端，当一个http请求访问的时候，我们将会重新生成session，删除掉旧的session和sessionid,确保在处理响应前，我们的一切是全新的。
  
```javascript
//Instead of..
exports.index = function (req, res) {
    res.render('index', { title: 'RedisPubSubApp', user: req.session.user});
};

//Use this..
exports.index = function (req, res) {
    //Save user from previous session (if it exists)
    var user = req.session.user;
    
    //Regenerate new session & store user from previous session (if it exists)
    req.session.regenerate(function (err) {
        req.session.user = user;
        res.render('index', { title: 'RedisPubSubApp', user: req.session.user});
    });
};








#### Credits

Front end UI: <a href="https://github.com/steffenwt/nodejs-pub-sub-chat-demo">https://github.com/steffenwt/nodejs-pub-sub-chat-demo</a></p>
