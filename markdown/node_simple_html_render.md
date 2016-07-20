# 【Javascript】Node.js简单HTML页面渲染

```javascript
var fs = require('fs');
var assert = require('assert');
var http = require('http');

/*
 * 渲染页面
 * @param {http.ServerResponse} 响应对象
 * @param {string} 模板文件
 * @param {object} 参数
 */
function render(res, view, args){
	assert(res instanceof http.ServerResponse);
	assert.equal('string', typeof view);
	assert.equal('object', typeof args);
	
	fs.readFile(view, function(err, buf){
		assert.equal(null, err);
		
		// 转换成字符串并消除换行
		var template = buf.toString().replace(/>\s*</mg, '><');
		
		for(var i in args){
			var reg = new RegExp('\\{\\$' + i + '\\}', 'g');
			template = template.replace(reg, args[i]);
		}
		
		res.statusCode = 200;
		res.setHeader("content-type", "text/html");
		res.end(template);
	});
}
```