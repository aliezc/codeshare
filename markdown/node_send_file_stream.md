# 【Javascript】Node.js使用stream发送文件

```javascript
var fs = require('fs');
var assert = require('assert');
var http = require('http');
var mime = require('mimez');

/*
 * 发送文件   ps:压缩和断点续传交给Nginx做反向代理
 * @param {http.ServerResponse} 响应对象
 * @param {string} 文件地址
 */
function sendFile(res, file){
	assert(res instanceof http.ServerResponse);
	assert.equal('string', typeof file);
	
	try{
		var frs = fs.createReadStream(file);
		
		frs.on('error', function(err){
			// 读取文件出错
			res.statusCode = 404;
			res.end();
		});
		
		res.statusCode = 200;
		res.setHeader("content-type", mime.path(file));
		
		frs.pipe(res);
	}catch(e){
		// 读取文件出错
		res.statusCode = 404;
		res.end();
	}
}
```