分享会准备材料：
Promise 对象：
---------------------------promise概述-----------------------
Promise 是异步编程的一种解决方案，所谓Promise，简单说就是一个容器，里面保存着某个未来才会结束的事件（通常是一个异步操作）的结果。从语法上说，Promise 是一个对象，从它可以获取异步操作的消息。Promise 提供统一的 API，各种异步操作都可以用同样的方法进行处理。
---------------------------promise概述-----------------------

---------------------------promise特点-----------------------
Promise对象有以下两个特点。
（1）对象的状态不受外界影响。Promise对象代表一个异步操作，有三种状态：pending（进行中）、resolved（已成功）和rejected（已失败）。
（2）只有异步操作的结果，可以决定当前是哪一种状态，任何其他操作都无法改变这个状态。一旦状态改变，就不会再变，任何时候都可以得到这个结果。Promise对象的状态改变，只有两种可能：从pending变为resolved和从pending变为rejected。只要这两种情况发生，状态就凝固了，不会再变了，会一直保持这个结果，这时就称为已定型。如果改变已经发生了，你再对Promise对象添加回调函数，也会立即得到这个结果。这与事件（Event）完全不同，事件的特点是，如果你错过了它，再去监听，是得不到结果的。
有了Promise对象，就可以将异步操作以同步操作的流程表达出来，避免了层层嵌套的回调函数。此外，Promise对象提供统一的接口，使得控制异步操作更加容易。

Promise也有一些缺点。首先，无法取消Promise，一旦新建它就会立即执行，无法中途取消。其次，如果不设置回调函数，Promise内部抛出的错误，不会反应到外部。第三，当处于pending状态时，无法得知目前进展到哪一个阶段（刚刚开始还是即将完成）。

ES6 规定，Promise对象是一个构造函数，用来生成Promise实例。
下面代码创造了一个Promise实例：
const promise = new Promise(function(resolve, reject) {
  // ... some code
  if (/* 异步操作成功 */){
    resolve(value);
  } else {
    reject(error);
  }
});
Promise构造函数接受一个函数作为参数，该参数函数又有两个参数分别是resolve和reject。它们又是两个函数，由 JavaScript 引擎提供，不用自己部署。
resolve函数的作用是，将Promise对象的状态从“未完成”变为“成功”（即从 pending 变为 resolved），在异步操作成功时调用，并将异步操作的结果value，作为参数传递出去；reject函数的作用是，将Promise对象的状态从“未完成”变为“失败”（即从 pending 变为 rejected），在异步操作失败时调用，并将异步操作报出的错误error，作为参数传递出去。

Promise实例生成以后，可以用then方法分别指定resolved状态和rejected状态的回调函数。
then方法可以接受两个回调函数作为参数。第一个回调函数是Promise对象的状态变为resolved时调用，第二个回调函数是Promise对象的状态变为rejected时调用。我猜应该就是分别是上面的resolve(value)函数跟reject(error)函数。
promise.then(function(value) {
  // success
}, function(error) {
  // failure
});
promise.catch 用于处理之前没有被处理的 rejected promise,相当于promise.then(null, errCb)
promise.finally 将最后被调用, 一般用于资源释放的清理操作

关于执行顺序，如：
let promise = new Promise(function(resolve, reject) {
  console.log('1');
  resolve();
});
promise.then(function() {
  console.log('2');
});
console.log('3');
输出顺序为：
// 1、 3、 2
上面代码中，Promise 新建后立即执行，所以首先输出的是1。然后，then方法指定的回调函数，将在当前脚本所有同步任务执行完才会执行，所以2最后输出。


某些特殊情况：
resolve函数的参数除了正常的值以外，还可能是另一个 Promise 实例，比如像下面这样：
const p1 = new Promise(function (resolve, reject) {});
const p2 = new Promise(function (resolve, reject) {resolve(p1);})
即一个异步操作（p2）的结果是返回另一个异步操作（p1）。
注意，这时p1的状态就会传递给p2，也就是说，p1的状态决定了p2的状态。如果p1的状态是pending，那么p2的回调函数就会等待p1的状态改变；如果p1的状态已经是resolved或者rejected，那么p2的回调函数将会立刻执行。如下案例：
const p1 = new Promise(function (resolve, reject) {
  setTimeout(() => reject(new Error('fail')), 3000)
})
const p2 = new Promise(function (resolve, reject) {
  setTimeout(() => resolve(p1), 1000)
})

p2
  .then(result => console.log(result))
  .catch(error => console.log(error))
最终输出// Error: fail
上面代码中，p1这个promise实例在3 秒之后变为rejected状态。由于p2的resolve依赖p1，导致p2自己的状态无效了，由p1的状态决定p2的状态。


promise chains 的链式调用
任何在 successCb, errCb 中返回的非 $q.reject()对象, 都将成为一个 resolve 的 promise. 
所以可以出现如下语法 promise.then().then().then()

$q.when("1")
    .then(function (data) {
        console.log(data);
        return $q.reject(2);
    })
    .catch(function (err) {
        console.log(err);
        return 3;
    })
    .then(function (data) {
        console.log(data);
    })
    
// output: 1 2 3


var promise = new Promise(function(resolve){
		    resolve('1');
		});
		promise.then(function(value1){
		    console.log(value1);
		    return '2';
		})
		.then(function(value2){
		    console.log(value2);
		});
以上依次输出1、2。说明了两个问题：
尽管第一个then只是return一个’2’，但这个then仍然返回的是一个promise对象，否则其后怎么可以又接一个then？
第一个then里的resolved函数里return的值，会作为参数传入到其后的then里的resolved函数里。
以下代码得到的结果跟上面的代码效果一样：
var promise = new Promise(function(resolve,reject){
		    resolve('1');
		});
		promise.then(function(value1){
		    console.log(value1);
		    return new Promise(function(resolve,reject){
			    resolve('2');
			});
		})
		.then(function(value2){
		    console.log(value2);
		});


采用链式的then，可以指定一组按照次序调用的回调函数。这时，前一个回调函数，有可能返回的还是一个Promise对象（即有异步操作），这时后一个回调函数，就会等待该Promise对象的状态发生变化，才会被调用。
getJSON("/post/1.json").then(function(post) {
  return getJSON(post.commentURL);
}).then(function funcA(comments) {
  console.log("resolved: ", comments);
}, function funcB(err){
  console.log("rejected: ", err);
});
上面代码中，第一个then方法指定的回调函数，返回的是另一个Promise对象。这时，第二个then方法指定的回调函数，就会等待这个新的Promise对象状态发生变化。如果变为resolved，就调用funcA，如果状态变为rejected，就调用funcB。
---------------------------promise特点-----------------------



---------------------------$q、$when-----------------------
$q创建 promise
$q 支持两种写法,第一种是ES6标准构造函数写法：
var iWantResolve = true;
function es6promise() {
    return $q(function (resolve, reject) {
        $timeout(function () {
            if (iWantResolve) {
                resolve("es6promise resolved");
            } else {
                reject("es6promise reject");
            }
        }, 1000)
    })
}


第二种是类似于 commonJS 的写法 $q.deferred()：
function commonJsPromise() {
    var deferred = $q.defer();
    $timeout(function () {
        deferred.notify("commonJS notify");
        deferred.resolve("commonJS resolved");
    }, 500);

    return deferred.promise;
}

commonJsPromise()
    .then(function (data) {
        console.log(data);
    }, function (err) {
        console.log(err);
    }, function (update) {
        console.log(update);
    });
 





$q.all方法说明
$q.all([promise1, promise1]) 接受一个包含若干个 promise 的数组,
等所有的 promise resolve 后, 其本身 resolve 包含上述结果的数组 [data1, data2]
如果上述 promise 有一个 reject, 那么$q.all() 会把这个 rejected promise 作为其 rejected promise (只有一个哦)
$q.all([es6promise(), commonJsPromise()])
    .then(function (dataArr) {
        console.log("$q.all: ", dataArr);
    }, function (err) {
        console.log("$q.all: ", err)
    }, function  (update) {//unnecessary
        console.log("$q.all", update);
    });




$q.reject, $q.when, $q.resolve
$q.reject() 立即返回一个rejected 的 promise, 在链式调用的时候很有用
如：$q.reject("instant reject")
    .catch(function (err) {
        console.log(err);
    });
// output: instant reject

$q.when(value, successCb, errorCb, progressCb)
value 可能是任意数据, 但是 $q.when 最终都会返回一个 promise
$q.when 既可以写成上述的构造函数形式, 也可以写成 $q.when(value).then(fn, fn, fn) 的形式
如：
$q.when(commonJsPromise(),
    function /** success callback **/(data) {
        console.log("$q.when success callback function: " + data);
        return "$q.when success callback return another value";
    })
    .then(function (data) {
        console.log("$q.when then function:" + data);
    });



$q.when("some value", function (data){
    console.log(data);
})
// output: some value

$q.resolve 相当于 $q.when(value, successCb, errorCb, progressCb)
---------------------------$q、$when-----------------------










接下来看
------------------------------------------await概述-------------------------------------------
既然有了promise 为什么还要有async await ？当然是promise也不是完美的异步解决方案，而async await 的写法看起来更加简单且容易理解。
async/await相比原来的Promise的优势在于处理then链。
回顾Promise：Promise对象用于表示一个异步操作的最终状态（完成或失败），以及其返回的值。

async await字面理解：async用于申明一个function 是异步的，而await用于等待一个异步任务执行完成的的结果。并且await 只能出现在async函数中。
async告诉程序这是一个异步操作。
await是一个操作符，即await后面是一个异步表达式/promise，程序的执行流会暂停，且一直在等待这个异步表达式/promise的结果。
------------------------------------------await概述-------------------------------------------


-----------------------------------------async函数-------------------------------------
async函数的返回值
当调用一个async函数时，会返回一个Promise对象。
当这个async函数返回的是一个值时，Promise的resolve方法会负责传递这个值；
当async函数抛出异常时，Promise的reject方法也会传递这个异常值。
既然async函数返回的是Promise对象，且由于在async的最外层不能用await获取其返回值(await只能在其里层)，那么肯定可以用原来的方式：then()链来处理这个Promise对象，如：
async function testAsync() {
  return "hello";
}
let data = testAsync().then( (data) => {
  console.log(data) //hello
  return data
});
console.log(data);
而如果async函数没有return返回值,那么调用async函数相当于Promise.resolve(undefined)
-----------------------------------------async函数-------------------------------------


-----------------------------------------await表达式的值---------------------------------
假如await后面的表达式返回的是一个Promise对象，那么这个promise的resolve函数参数将作为await表达式的值。如果这个Promise rejected了，await表达式也能够捕获到这个promise的异常并把这个Promise的异常抛出。
假如await后面的表达式返回的是一个常量，那么会把这个常量转为Promise.resolve(常量)，同理如果没有返回值也是underfind。

demo
返回promose 对象，成功状态
function say() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            resolve('hello-success');
        }, 1000);
    });
 }
  
async function demo() {
    const v = await say();
    console.log(v);//输出hello-success
}
demo();


返回promise 对象，失败状态
function say() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            reject('ee-failed');
        }, 1000);
     });
}
async function demo() {
    try {
        const v = await say();
        console.log(v);//输出 'ee-failed'
    } catch (e) {
        console.log(e);//输出 'ee-failed'
    }
}
demo();
-----------------------------------------await表达式的值---------------------------------
------------------穿插工作实践-----------------





既然说到了async，那么接下来再来看
nodejs常用模块async(waterfall,each,eachSeries,whilst)
----------------------------------------引入背景-------------------------------------------
先看一段代码
var objs = [{name:'A'}, {name:'B'}, {name:'C'}];
function doSomething(obj, cb){
    console.log("我在做" + obj.name + "这件事!");
    cb(null, obj);
}

我们要依次把objs中的3件事做完，顺序为A->B->C，我们可以按下面这样
doSomething(objs[0], function(err, data){
    if(err)
    {
        console.log('处理错误!');
    }
    else
    {
        doSomething(objs[1], function(err, data){
            if(err)
            {
                console.log('处理错误!');
            }
            else
            {
                doSomething(objs[2], function(err, data){
                    if(err)
                    {
                        console.log('处理错误!');
                    }
                    else
                    {
                        console.log('处理成功！');
                    }
                });
            }
        });
    }
});

嗯，层次好多，没错，这就是nodejs的异步IO的必然结果。回到我们做这些的事的出发点，应该如下面这样:
做A这件事
如果A成功，再做B这件事
如果B成功，再做C这件事
*/
----------------------------------------引入背景-------------------------------------------



var async = require('async');
var objs = [{name:'A'}, {name:'B'}, {name:'C'}];
function doSomething(obj, cb){
    console.log("我正在做" + obj.name + "这件事!");
    cb(null, obj);
}


----------------------------------------async瀑布模型-------------------------------------------
/*
//看看async模块的瀑布模型:
async.waterfall([
    //A这件事
    function(cb)
    {
        doSomething(objs[0], function(err, dataA){
            console.log('看看dataA：');console.log(dataA);
            cb(err, dataA);     //如果发生err，则瀑布就完了，后续流程都不会执行，B和C都不会执行
        });
    },
    //B这件事，dataA就是上一步cb(err, dataA)中传进来的dataA
    function(dataA, cb)
    {
        doSomething(objs[1], function(err, dataB){
            console.log('看看dataB：');console.log(dataB);
            cb(err, dataA, dataB);  //如果发生err， 则瀑布就完了，后续流程都不会执行，C不会执行
        });
    },
    //C这件事
    function(dataA, dataB, cb)
    {
        doSomething(objs[2], function(err, dataC){
            console.log('看看dataC：');console.log(dataC);
            cb(err, dataA, dataB, dataC);
        });
    }
], function (err, dataA, dataB, dataC) {    //瀑布的每一布，只要cb(err, data)的err发生，就会到这
    if(err){
        console.log('处理错误!');
    }
    else{
        console.log('全部处理成功！');
        console.log(dataA);
        console.log(dataB);
        console.log(dataC);
    }
});

输出
我正在做A这件事!
看看dataA：
{ name: 'A' }
我正在做B这件事!
看看dataB：
{ name: 'B' }
我正在做C这件事!
看看dataC：
{ name: 'C' }
全部处理成功！
{ name: 'A' }
{ name: 'B' }
{ name: 'C' }
*/
----------------------------------------async瀑布模型-------------------------------------------
------------------穿插工作实践-----------------


/*
//下一个要介绍的模型，whilst，一直执行主函数，直到条件不满足，或者发生异常
var i = 0;
async.whilst(
    //条件
    function() {
        return i < 3;   //true，则第二个函数会继续执行，否则，调出循环
    },
    function(whileCb) { //循环的主体
        console.log('循环主体里的i：'+i);
        i++;
        whileCb();
    },
    function(err) { //如果条件不满足，或者发生异常
        console.log("发生了一点错误:" + err);
        console.log("结束，i是：:" + i);
    }
);

//输出
循环主体里的i：0
循环主体里的i：1
循环主体里的i：2
发生了一点错误:null
结束，i是：:3
*/




/*
//接下来要介绍的each 
async.each(objs, function(obj, callback) {
    doSomething(obj, function(err,aaa){});
    //callback();//没加参数
    callback('aaa');//加了参数后
}, function(rrr){
    console.log("传输的信息是:"+rrr);
});


//callback没加参数时输出：
我正在做A这件事!
我正在做B这件事!
我正在做C这件事!
传输的信息是:null

//callback加了参数后输出：
我正在做A这件事!
传输的信息是:aaa
我正在做B这件事!
我正在做C这件事!

我也没法解释这两种结果的不同，我只知道这里异步操作就是这3个doSomething操作。
callback函数【就是那个function(rrr){console.log("传输的信息是:"+rrr)})函数】必然阻塞（同一时间只有一个线程在执行它）。
*/





/*接下来介绍eachSeries，这个和each差不多，就是遍历每一个对象是分步的，上一个对象的callback执行完之后，才会执行下一个。 
//假如如果没有异步调用，和each无差别，如果有异步调用，则有差别。
async.eachSeries(objs, function(obj, callback) {
    doSomething(obj, function(){});
    //callback();
    callback('aaa');//有传递参数，表示有异步调用，其实有传递参数可能表示的是报错。
}, function(rrr){
    console.log("传递的信心:" + rrr);
});


//当callback没有传递参数时输出：
我正在做A这件事!
我正在做B这件事!
我正在做C这件事!
传递的信心:null

//当callback有传递参数时输出：
我正在做A这件事!
传递的信心:aaa
*/









//接下来介绍parallel(tasks, [callback])
//parallel函数是并行执行多个函数，每个函数都是立即执行，不需要等待其它函数先执行。传给最终callback的数组中的数据按照tasks中声明的顺序，而不是执行完成的顺序，示例如下：
//tasks参数可以是一个数组或是json对象，和series函数一样，tasks参数类型不同，返回的results格式会不一样。
async.parallel([
    function(callback){
        callback(null, '结果1');
    },
    function(callback){
        callback(null, '结果2');
    }
],
function(err, results){
    console.log('看看结果:'+results);//打印出   看看结果：结果1,结果2
});






接下来如果还想看
co模块，它基于ES6的generator和yield ，让我们能用同步的形式编写异步代码。
co(function *() {
    var data = yield $.get('/api/data');
    console.log(data);
    var user = yield $.get('/api/user');
    console.log(user);
    var products = yield $.get('/api/products');
    console.log(products);
});
------------------穿插工作实践-----------------
