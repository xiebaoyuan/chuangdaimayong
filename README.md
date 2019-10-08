10.1  中间件
get、post回调函数中，没有next参数，那么就匹配上第一个路由，就不会往下匹配了。
如果想往下匹配的话，那么需要写next()
例子：
下面两个路由有冲突（admin可以当做用户名 login可以当做id），这样写的话，可能只会执行第一个路由而不会执行第二个路由。
解决方法： 
总之，路由get、post这些东西，就是中间件，中间件讲究顺序，匹配上第一个之后，就不会往后匹配了。next函数才能够继续往后匹配。
app.use()也是一个中间件。与get、post不同的是，他的参数中的url不是精确匹配的。而是能够无限文件夹拓展下去的。
比如网址：  http://127.0.0.1:3000/admin/aa/bb/cc/dd，其实只认/admin，后面都是其文件夹拓展


10.2 express 中 app.all 和 app.use 的区别是什么？
all 执行完整匹配，use 只匹配前缀

app.use('/a', (req, res, next) =>
  console.log ('qqqqqq');
  next();
)
  
app.all ('/a', (req, res, next) =>
  console.log ('aaaaa');
  next();
)
访问 /a 时，use 和 all 都会被调用；而访问 /a/b 只有 use 被调用。
   

10.3 express的app的其他一些用法介绍：
app.set(name, value)：设置name的值为value。我猜想：相应的可以使用app.get(name)获取值。
 app.set('views', __dirname + '/views')：设置 views 文件夹为视图文件的目录，存放模板文件，__dirname 为全局变量，存储着当前正在执行脚本所在的目录名。
 app.set('view engine', 'ejs')：设置视图模版引擎为 ejs
 app.use([path], function)：使用中间件 function，可选参数path默认为"/" 
app.use(express.favicon())：connect 内建的中间件，使用默认的 favicon 图标，如果想使用自己的图标，需改为app.use(express.favicon(__dirname + '/public/images/favicon.ico')); 这里我们把自定义的 favicon.ico 放到了 public/images 文件夹下。
 app.use(express.logger('dev'))：connect 内建的中间件，在开发环境下使用。




10.4.POST和GET
   GET请求的参数在URL中，在原生Node中，需要使用url模块来识别参数字符串。在Express中，不需要使用url模块了。可以直接使用req.query对象。
POST请求在express中不能直接获得，必须使用第三方body-parser模块。使用后，将可以用req.body得到参数。但是如果表单中含有文件上传，那么还是需要使用formidable模块。



10.5 Express 的Router模块
我们在既有的路由之后，使用express的 Router 功能可以加上额外的一些路由设定：
// ---- 基本设定 ----
var express = require('express');
var app     = express();
var port    = process.env.PORT || 8080;

// 传统路由方法
app.get('/sample', function(req, res) {
  res.send('this is a sample!');
});

// Express Router建立 Router 的方式
var router = express.Router();
router.get('/', function(req, res) {
  res.send('home page!');
});

router.get('/about', function(req, res) {
  res.send('about page!');
});

// 将路由套用至应用
app.use('/', router);
app.listen(port);




10.6 Route Middleware
Express 的 middleware 功能可以让请求（request）在被处理之前，先执行一些前置作业，例如检查使用者是否有登入或是记录一些使用者的浏览资料等等，凡是需要在实际处理请求之前要做的动作，都可以使用 middleware 来处理。
var router = express.Router();
// 在每一个请求被处理之前都会执行的 middleware
router.use(function(req, res, next) {
  console.log(req.method, req.url);
  //继续路由处理
  next();
});

router.get('/', function(req, res) {
  res.send('home page!');
});

router.get('/about', function(req, res) {
  res.send('about page!');
});

app.use('/', router);

注意：在使用 middleware 时要注意他的放置位置必须要在 routes 之前，程式在执行的时候会依据 middleware 与 routes 的先后顺序来执行，如果不小心將 middleware 放在 routes 之后，那么在 routes处理完请求之后就会结束处理的流程，这样 middleware 就根本不会执行。






11 mongodb
首先把应用程序exe路径（D:\Program Files (x86)\mongodb\bin）加入到系统的path环境变量中
mongo   使用数据库
mongod  开机
mongoimport  导入数据
  
首先开机命令mongod --dbpath c:\mongo(win7需要安装一个叫做KB2731284-v3-x64.msu的修复补丁)
--dbpath就是选择数据库文档所在的文件夹（c:\mongo 需要自己创建此文件夹）。也就是说，mongoDB中，真的有物理文件，对应一个个数据库。U盘可以拷走。
一定要保持开机命令这个CMD不能动了，不能ctrl+c。 一旦关了了，数据库就自动关闭了。
所以，应该再开一个cmd。输入mongo连接数据库了。同时也进入mongo环境。
列出所有数据库：show dbs
查看当前所在数据库：db
使用某个数据库：use 数据库名字（如果想新建数据库，也是use。use一个不存在的，就是新建。）
如果真想创建成功一个数据库，那么必须在“use 数据库 ” 环境下的集合里插入一条数据，这个数据库才能创建成功。
数据库中不能直接插入数据，只能往集合(collections)中插入数据。集合也不需要创建会自动创建。


11.1 插入数据：
db.jihe01.insert({"name":"xiaoming","age":12,"sex":"man" });
jihe01就是所谓的集合。集合中存储着很多json。假如此时还没有jihe01，那么集合jihe01将自动创建。

案例：创建一个数据库sjk01，同时在这个数据库里创建一个集合jihe01。
   > db
test
> use sjk01
switched to db sjk01
> db.jihe01.insert({"name":"xie","age":"27"});
WriteResult({ "nInserted" : 1 })
> db
sjk01



11.2 查找数据
查找数据，find中如果没有参数，那么将列出这个集合的所有文档：
> db.jihe01.find()
精确匹配：
   > db.jihe01.find({"score.shuxue":70});
   找到结果如下：
   {"name":"小红","age":11,"hobby":["学习","看书"],"score":{"yuwen":100,"shuxue":70}}
{"name":"小强","age":13,"hobby":["打架"],"score":{"yuwen":2,"shuxue":70}}

多个条件：
   > db.jihe01.find({"score.shuxue":70 , "age":11});
大于条件：
   > db.jihe01.find({"score.yuwen":{$gt:50}});
或者。寻找所有年龄是9岁，或者11岁的学生 
   > db.jihe01.find({$or:[{"age":9},{"age":11}]});
查找完毕之后，调用sort可以升降或降序。
   > db.jihe01.find().sort( { "age": -1, "score.yuwen": 1 } );




查找某字段不重复的值，如下是 col 集合的数据
{ "_id": 1, "dept": "A", "item": { "sku": "111", "color": "red" }, "sizes": [ "S", "M" ] }
{ "_id": 2, "dept": "A", "item": { "sku": "111", "color": "blue" }, "sizes": [ "M", "L" ] }
{ "_id": 3, "dept": "B", "item": { "sku": "222", "color": "blue" }, "sizes": "S" }
{ "_id": 4, "dept": "A", "item": { "sku": "333", "color": "black" }, "sizes": [ "S" ] }
------------------------------
>db.col.distinct(“sizes”)
结果：[“M”,”S”,”L”]






  

11.3  修改数据
查找名字叫做小明的，把年龄更改为16岁：
   > db.jihe01.update({"name":"小明"},{$set:{"age":16}});
查找数学成绩是70，把年龄更改为133岁：
  > db.jihe01.update({"score.shuxue":70},{$set:{"age":133}});
更改所有匹配的文档(需要加个条件multi:true)： 
  > db.jihe01.update({"sex":"男"},{$set:{"age":33}},{multi: true});
彻底更换文档，不需要$set关键字了：
  > db.jihe01.update({"name":"小明"},{"name":"大明","age":16});

$inc修改器可以对文档的某个值为数字型的键进行增减的操作。或者在键不存在时创建一个键。
$inc只能用于修改整数、长整数和双精度浮点数。要是其他类型应该使用$set修改器。


MongoDB的update方法的upsert参数:这个参数的意思是，如果不存在query记录则执行insert否则执行update。
MongoDB里面
db.collection.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>
   }
)

mongoose里面
poDao.updateRecord({ppid : 12345},{$set :{ppid : 12345,model : 'qquu'},$inc : { count : 5}},{upsert:true},function(err){});




11.4  删除数据
> db.jihe01.remove( { "name": "大明" } ) )
  如果只删除匹配的第一个文档，而不是匹配的所有文档，则：
  > db.jihe01.remove( { "name": "小红" }, { justOne: true } )

  
11.5  数据分页
利用limit() 配合skip()
假如请求第1页（page=1）的数据，每页显示10条：
>db.jihe01.find({}).limit(10).skip(（page-1）*10)
第二页（page=2）：
>db.jihe01.find({}).limit(10).skip(（page-1）*10)




12. cookie 和 session
   12.1  cookie
   Cookie：当访问一个页面的时候，服务器在下行HTTP报文中，命令浏览器存储一个字符串；浏览器再访问同一个域的时候，将把这个字符串携带到上行HTTP请求中。每一次浏览器往这个服务器发出的请求，都会携带这个cookie。
express中res负责设置cookie， req负责识别cookie。
  案例：
  //使用cookie必须要使用第三方模块cookie-parser中间件
app.use(cookieParser());
app.get(“/”,function(req,res){
     res.cookie("key",value,{maxAge: 900000, httpOnly: true});
});
  
  12.2 session
   session依赖cookie，当浏览器清除了cookie，登陆也消失。session比cookie不一样在哪里呢？  session下发的是乱码，并且服务器自己也会缓存一些东西，下次浏览器的请求带着乱码上来，此时与缓存进行比较，看看是谁。
  案例：
  var express = require("express");
var app = express();
var session = require("express-session");
app.use(session({
      secret: 'keyboard cat',
      resave: false,
      saveUninitialized: true
}));
app.get("/",function(req,res){
  if(req.session.login == "1"){
    res.send("欢迎" + req.session.username);
  }else{
    res.send("没有成功登陆");
  }
});
app.get("/login",function(req,res){
  req.session.login = "1";//设置session
  req.session.username = "考拉";
  res.send("你已经成功登陆");
});
app.listen(3000);





13.Websocket
WebSocket的原理：利用HTTP请求产生握手，HTTP头部中含有WebSocket协议的请求，所以握手之后，二者转用TCP协议进行交流。
写原生的JS，搭建一个服务器，server创建好之后，创建一个io对象

在客户端HTML页面中，必须引用服务端的js文件(socket.io/socket.io.js)。调用io函数，取得socket对象。
在服务端，每一个连接上来的用户，都为之建立一个socket对象。

客户端：
var socket = io();
socket.emit("tiwen","吃了么？");
socket.on("huida",function(msg){
  console.log("服务器说:" , msg);
});

服务器端：
服务器的socket通信原理：
基于TCP/IP通信协议的套接字Socket、客户端和服务端通信问题： 在客户机/服务器工作模式中。用端口号标识正在Server端后台服务的线程。端口号与IP地址的组合称为网络套接字（socket）。 Server机通过端口（总线I/O地址）提供面向Client机的服务；Server机在它的几个不同端口分别同时提供几种不同的服务。 当Client程序和Server程序需要通信时，可以用Socket类建立套接字连接。套接字连接可想象为一个电话呼叫：最初是Client程序建立呼叫，Server程序监听；呼叫完成后，任何一方都可以随时讲话。 双方实现通信有流式socket和数据报式socket两种可选方式： 流式socket是有连接的通信，即TCP(Transmission Control Protocol)：每次通信前建立连接，通信结束后断开连接。 数据报式socket是无连接的通信，即UDP(User Datagram Protocol)：将欲传输的数据分成 小包，直接上网发送。无需建立连接和拆除连接。 流式socket在Client程序和Server程序间建立通信的通道。每个socket可以进行读和写两种操作。对于任一端，与对方的通信会话过程是： 建立socket连接，获得输入/输出流，读数据/写数据，通信完成后关闭socket（拆除连接）。 利用socket的构造方法，可以在客户端建立到服务器的套接字对象：    服务器端程序在指定的端口监听，当收到Client程序发出的服务请求时，创建一个套接字对象与该端口对应的Client程序通信。
