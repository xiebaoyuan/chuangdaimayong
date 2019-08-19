var http = require("http");
var urll = require("url");
var fs = require("fs");
var path = require("path");

http.createServer(function(req,res){
	var pathname = urll.parse(req.url).pathname;
	if(pathname == "/"){
		//默认首页
		pathname = "index.html";
	}
	
	var extname = path.extname(pathname);//得到拓展名

	//去读取pathname的那个文件
	fs.readFile("./" + pathname,function(err,data){
		//这就是静态资源管理的核心！直接指到服务器上的（127.0.0.1的根目录）某个文件夹去读取那里的文件（pathname）。
		if(err){
			fs.readFile("./404.html",function(err,data){
				//如果此文件不存在，返回404页面
				res.writeHead(404,{"Content-type":"text/html;charset=UTF8"});
				res.end(data);
			});
			return;
		};

		//MIME类型，就是响应头的Content-type
		 getMime(extname,function(mime){
            res.writeHead(200,{"Content-Type":mime});
            res.end(data);
        });
	});

}).listen(3000,"127.0.0.1");


function getMime(extname,funn){
    fs.readFile("./mime.json",function(err,data){
        var mimeJSON = JSON.parse(data);
        var mime =  mimeJSON[extname]  || "text/plain";
        funn(mime);
    });
}
